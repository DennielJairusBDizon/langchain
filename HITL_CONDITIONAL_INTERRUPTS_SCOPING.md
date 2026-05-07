# Conditional interrupts for `HumanInTheLoopMiddleware`

## Problem

`HumanInTheLoopMiddleware` currently decides whether to interrupt solely by tool
name. This works for tools that are always sensitive, but it is too coarse for
file editing tools such as `edit_file` and `write_file`, where most writes can
proceed automatically and only protected paths should require human review.

The target user experience is:

```python
import re

protected_paths = re.compile(r"^(?:\.env|pyproject\.toml|libs/core/)")

HumanInTheLoopMiddleware(
    interrupt_on={
        "edit_file": {
            "allowed_decisions": ["approve", "edit", "reject"],
            "interrupt_when": lambda tool_call, _state, _runtime: bool(
                protected_paths.search(str(tool_call["args"].get("path", "")))
            ),
        },
        "write_file": {
            "allowed_decisions": ["approve", "edit", "reject"],
            "interrupt_when": lambda tool_call, _state, _runtime: bool(
                protected_paths.search(str(tool_call["args"].get("path", "")))
            ),
        },
    }
)
```

Calls to these tools whose `path` argument does not match the predicate would be
treated the same as tools not listed in `interrupt_on`: no interrupt is raised
and the tool call remains in the `AIMessage`.

## Current implementation

Relevant code lives in
`libs/langchain_v1/langchain/agents/middleware/human_in_the_loop.py`.

- `InterruptOnConfig` is an exported `TypedDict` with `allowed_decisions`,
  optional `description`, and optional `args_schema`.
- `HumanInTheLoopMiddleware.__init__` normalizes `interrupt_on`; `False`
  entries are dropped, `True` entries become all decisions, and config dicts are
  kept when they include `allowed_decisions`.
- `after_model` iterates over `last_ai_msg.tool_calls` and interrupts every
  call whose name exists in `self.interrupt_on`.
- `HITLRequest` construction does not need to change. Conditional logic only
  affects which tool calls are included in `action_requests` and
  `review_configs`.
- Decision processing is already index based and preserves tool call order when
  interrupting a subset of model-proposed tool calls.

Existing unit tests are in
`libs/langchain_v1/tests/unit_tests/agents/middleware/implementations/test_human_in_the_loop.py`.
They already cover auto-approved tools mixed with interrupted tools, request
shape, decision count validation, and order preservation. This feature can be
covered by extending that same test file.

## Recommended API

Add an optional `interrupt_when` field to `InterruptOnConfig`.

```python
class _InterruptWhen(Protocol):
    def __call__(
        self,
        tool_call: ToolCall,
        state: AgentState[Any],
        runtime: Runtime[ContextT],
    ) -> bool:
        """Return whether this tool call should interrupt."""
        ...


class InterruptOnConfig(TypedDict):
    allowed_decisions: list[DecisionType]
    description: NotRequired[str | _DescriptionFactory]
    args_schema: NotRequired[dict[str, Any]]
    interrupt_when: NotRequired[_InterruptWhen]
```

Semantics:

- If `interrupt_when` is omitted, behavior is unchanged: every configured call
  for that tool interrupts.
- If `interrupt_when` returns `True`, the call interrupts with the configured
  `allowed_decisions`.
- If `interrupt_when` returns `False`, the call is auto-approved.
- Exceptions raised by `interrupt_when` should propagate. Silently approving on
  predicate failure would be unsafe.
- The predicate should be synchronous and deterministic. `aafter_model`
  currently delegates to `after_model`, and LangGraph interrupt replay requires
  the same interrupt calls to occur when resuming.

This is the smallest public API that supports regex matching without baking path
or regex semantics into the middleware. It also supports future non-path cases
such as interrupting database tools only for `DELETE` statements, HTTP tools
only for certain hosts, or email tools only for external recipients.

## Optional convenience API

If the team wants a more declarative path for the common regex case, add a
second field instead of, or in addition to, the predicate:

```python
class InterruptOnConfig(TypedDict):
    allowed_decisions: list[DecisionType]
    arg_patterns: NotRequired[dict[str, str | Pattern[str]]]
```

Potential semantics:

- All configured argument patterns must match their corresponding args.
- Missing args are non-matches.
- Non-string arg values are converted with `str(value)`.

Example:

```python
HumanInTheLoopMiddleware(
    interrupt_on={
        "edit_file": {
            "allowed_decisions": ["approve", "edit", "reject"],
            "arg_patterns": {"path": r"^(?:\.env|pyproject\.toml|libs/core/)"},
        }
    }
)
```

I would not start here. The predicate is more flexible, requires less API design,
and avoids deciding now whether multiple arg patterns are `all` or `any`, how to
handle regex flags, or whether compiled regex objects should be accepted.
`arg_patterns` can be added later as sugar without breaking the predicate API.

## Implementation scope

Expected code changes:

1. Add the `_InterruptWhen` protocol and `interrupt_when` field in
   `human_in_the_loop.py`.
2. Add a private helper, likely `_should_interrupt`, to centralize condition
   evaluation:

   ```python
   def _should_interrupt(
       self,
       tool_call: ToolCall,
       config: InterruptOnConfig,
       state: AgentState[Any],
       runtime: Runtime[ContextT],
   ) -> bool:
       interrupt_when = config.get("interrupt_when")
       if interrupt_when is None:
           return True
       return interrupt_when(tool_call, state, runtime)
   ```

3. In the `after_model` loop, replace the current exact-name-only check with:

   ```python
   config = self.interrupt_on.get(tool_call["name"])
   if config is not None and self._should_interrupt(tool_call, config, state, runtime):
       ...
   ```

4. Prefer tracking interrupted configs by index during request construction:

   ```python
   interrupt_configs: dict[int, InterruptOnConfig] = {}
   ...
   interrupt_configs[idx] = config
   ...
   if idx in interrupt_configs:
       config = interrupt_configs[idx]
   ```

   This avoids recomputing conditions during decision processing and avoids an
   extra lookup against `self.interrupt_on`.

5. Update docstrings for `InterruptOnConfig` and
   `HumanInTheLoopMiddleware.__init__`.
6. Export nothing new if `_InterruptWhen` stays private. `InterruptOnConfig` is
   already exported.

No changes should be needed to `HITLRequest`, `ReviewConfig`, or the shape of
the interrupt payload.

## Tests

Add unit tests in the existing HITL test file:

- `interrupt_when` returning `False` means no call to `interrupt` and
  `after_model` returns `None`.
- `interrupt_when` returning `True` preserves existing interrupt behavior.
- Mixed tool calls for the same tool name: one protected path interrupts, one
  unprotected path is auto-approved, and final tool call order is preserved.
- Mixed configured tools: one tool omitted from `interrupt_on`, one configured
  but predicate returns `False`, and one configured with predicate returning
  `True`.
- Predicate exceptions propagate.
- The predicate receives the original `ToolCall`, `state`, and `runtime`.

These are unit tests only; no network calls or integration tests are needed.

## Documentation

Update the Python HITL docs in the docs repo:

- `src/oss/langchain/human-in-the-loop.mdx`
- Possibly `src/oss/langchain/middleware/built-in.mdx`

The docs should show a protected file path regex example because that is the
clearest motivating case. Reference docs should update automatically from the
source docstrings.

## Compatibility and risk

This can be backward compatible:

- Existing `True`, `False`, and config dict values keep the same behavior.
- Adding a `NotRequired` key to `InterruptOnConfig` does not change existing
  call sites.
- The public constructor signature does not need to change.

Main risks:

- Non-deterministic predicates can break interrupt replay on resume. The docs
  should explicitly warn users to base predicates only on deterministic inputs.
- Async or I/O-heavy predicates do not fit the current middleware because
  `aafter_model` delegates to synchronous `after_model`.
- A predicate may accidentally auto-approve a sensitive call if user logic has a
  bug. Propagating exceptions and keeping examples defensive around missing args
  helps.
- This feature is Python-only unless mirrored in LangChain JS. The existing
  public docs present Python and JS together, so docs should avoid implying JS
  support until that implementation exists.

## Recommendation

Implement `interrupt_when` as the first version. It is a small, local change
with clear semantics, preserves existing behavior, supports regex-based path
checks, and leaves room for a declarative `arg_patterns` helper later if users
ask for it.
