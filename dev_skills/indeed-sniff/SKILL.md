---
name: indeed-sniff
description: Use when adding a new native read tool to the Indeed Employer MCP from dashboard functionality. Runs a HITL agent-browser sniff of the user's real logged-in Employer Portal profile, confirms the exact captured GraphQL operation with the user, then writes operations.py, MCP tool wrappers, and tests. Triggers on /indeed-sniff, /indeed-sniff <feature>, "sniff an Indeed dashboard operation", "build a native Indeed tool for this dashboard feature", or "map this Employer Portal GraphQL operation".
---

# Indeed Sniff

Build native MCP read tools from real Indeed Employer dashboard GraphQL traffic.

## Guardrails

- Read-only V1. If selected operation is a `mutation`, STOP and read `../../references/write-procedures.md` before any write discussion.
- Never automate login. The user must already have a legitimate authenticated `profile/`.
- Never persist candidate PII outside the capture artifact needed for tool development. Redact in summaries.
- Endpoint truth: Employer Portal dashboard traffic targets `https://apis.indeed.com/graphql`, but it uses internal dashboard operations, not the published ATS partner schema.

## Workflow

1. Resolve feature slug from `/indeed-sniff [feature]`; if missing, ask for one short slug.
2. Run preflight: `test -d profile` and `command -v agent-browser`.
3. Run capture:
   ```bash
   uv run python scripts/indeed_sniff_capture.py <feature>
   ```
4. Ask the user to drive the browser to perform the dashboard operation.
5. When capture finishes, read `captured_ops/pending/<feature>.json`.
6. Show captured operation names and variable shapes. Ask user to pick the exact operation and confirm desired tool behaviour.
7. After confirmation, follow `references/native-tool-checklist.md` to edit:
   - `src/indeed_employer_mcp/operations.py`
   - `src/indeed_employer_mcp/server.py`
   - `tests/unit/test_operations.py`
   - `tests/integration/test_server_tools.py`
8. Run `./verify.sh r0`, then `./verify.sh r1`; run `./verify.sh r2` (R2 base, not `r2-live`) before claiming verified.

## Output Contract

End with:
- new native `indeed_*` MCP tool name,
- captured operation name,
- files changed,
- verification commands and artifact paths,
- any live-unconfirmed risks.
