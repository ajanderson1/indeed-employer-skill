# Native Tool Checklist

## 1. User confirmation gate

Before code edits, confirm:

- selected operation name,
- user-facing tool name (`indeed_<noun>_<verb>`),
- required inputs and defaults,
- expected response shape,
- whether response contains PII,
- whether operation is read-only.

If query opens with `mutation`, stop.

## 2. Add operation

In `src/indeed_employer_mcp/operations.py`:

- add raw query text as `_FEATURE_QUERY = """..."""`,
- add `Operation(...)` with `source="captured"` when verbatim,
- keep default variables from capture unless user-confirmed inputs replace them,
- register operation in `CATALOGUE`.

## 3. Add MCP tool

In `src/indeed_employer_mcp/server.py`:

- add a thin `@mcp.tool()` wrapper,
- use `run_named_operation(...)`,
- merge user params into captured variables deliberately,
- document PII/session-scoped URLs in docstring.

## 4. Add tests

- `tests/unit/test_operations.py`: catalogue contains op, query starts with `query`, no unused variables.
- `tests/integration/test_server_tools.py`: fake `BrowserSession` receives expected operation and variables. Create a synthetic fake response matching the user-confirmed output shape.
- Add page-and-select tests if tool scans lists.

## Integration test template

```python
async def test_indeed_new_tool_sends_expected_variables():
    browser = FakeBrowserSession(response={"data": {"feature": {"ok": True}}})
    state = HealthState(authenticated=True)

    await run_named_operation(
        browser,
        "CapturedOperationName",
        {"input": {"limit": 10}},
        state=state,
        logger_calls=[],
    )

    assert browser.calls[-1]["operation_name"] == "CapturedOperationName"
    assert browser.calls[-1]["variables"]["input"]["limit"] == 10
```

## 5. Verify

Run:

```bash
./verify.sh r0
./verify.sh r1
./verify.sh r2
```

Do not run `r2-live` unless explicitly allowlisted.
