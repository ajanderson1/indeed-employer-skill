---
name: indeed-employer
description: >
  Use when working with Indeed Employer or hiring-portal applicant data through
  the indeed_* MCP tools — jobs, applicants/candidates, CVs/resumes, applicant
  detail, counts, notes, messages, or GraphQL passthrough. Trigger even when the
  user does not say "Indeed" if context is hiring/applicants and they ask for a
  named candidate's CV/resume, e.g. "get me NOWORRY DZENGA cv", "show Sarah's
  resume", "download applicant CV", "new applicants for the cleaner job",
  "list my Indeed jobs", "candidate detail", "applicant counts", "draft a reply
  to a candidate", "run an Indeed GraphQL operation", or "is my Indeed session alive".
  Read-first; before any mutation/status change/reject/shortlist/reply, read
  references/write-procedures.md and confirm the exact target.
---

# Indeed Employer

A go-between for the live Indeed Employer MCP. Translates plain requests ("new
candidates for the cleaner job", "how many at interview stage") into the right
`indeed_*` tool call. **Read-first**: reads and passthrough-discovery are the core;
writes are documented but gated (see § Writes).

The MCP server replays the employer portal's internal GraphQL from an
authenticated browser session. All tools are prefixed `indeed_`. The endpoint is
`https://indeed-mcp.ajanderson.net/mcp`.

## When to use

- "get me <candidate name> CV" / "show <applicant name>'s resume" / "download applicant CV"
  when the surrounding context is hiring/applicants, even if the user does not say "Indeed"
- "show me new candidates" / "new applicants for job X" / "list applicants"
- "list my Indeed jobs" / "how many jobs / candidates do I have"
- "candidate detail for <id>" / "applicant counts" / "filter options"
- "draft a reply to a candidate" (drafting is a read; sending is a guarded write)
- "Indeed passthrough" / "run an Indeed GraphQL operation" / "discover/add an op"
- "is my Indeed session alive" / "do I need to re-seed"

## First move every session: check the session is alive

Always start with `indeed_session_status`. It returns `{mcp_ok, authenticated,
last_ok, needs_reseed, expires_hint}`. If `needs_reseed` is true, STOP and tell
the user — the session needs a manual human re-seed (the VNC re-auth gauntlet); no
tool will work and there is no automated fix. Never retry a dead session in a loop.

## Read recipes (the core)

Wrap these typed read tools — all read-only, all already catalogued server-side.

| User asks | Tool | Notes |
|---|---|---|
| Who am I / my permissions | `indeed_whoami` | logged-in employer user + grants |
| List my jobs | `indeed_list_jobs(limit, statuses, sort_field, sort_direction)` | `statuses` e.g. `["ACTIVE"]`, `["PAUSED"]`, `["CLOSED"]` |
| How many jobs | `indeed_count_jobs` | estimated total |
| One job's fuller record | `indeed_job_detail(job_id)` | paged scan; `job_id` is the **base64 EmployerJob IRI token** (portal URL `selectedJobs` param), NOT `jobData.id` or `legacyId` |
| List applicants | `indeed_list_candidates(limit, dispositions)` | **returns PII**; `dispositions` e.g. `["INTERVIEWED","OFFER_MADE"]` |
| Counts by stage / sentiment | `indeed_candidate_counts` | milestone + shortlist counts |
| Filter facets | `indeed_candidate_filter_options` | locations, milestones, sentiments |
| One applicant, full record | `indeed_candidate_detail(submission_id)` | **PII**; paged scan; resume *availability*, not content |
| Notes / rejection comments | `indeed_candidate_notes(submission_id)` | **PII**; paged scan |

Recipe pattern for "show me new candidates for job X":
1. `indeed_session_status` → confirm alive.
2. `indeed_list_jobs` → find job X's id/title (confirm with the user which job).
3. `indeed_list_candidates` (filter by disposition for stage) → list applicants.
4. For one person: `indeed_candidate_detail(submission_id)` → full record.

**When typed tools fail — use the passthrough fallback.** The catalogued
`indeed_candidate_detail`, `indeed_candidate_resume`, and `indeed_candidate_notes`
use a paged scan (25 pages max by default). If the target is not in the first 5
pages the tool raises "not found". Recovery: run a raw `indeed_graphql` passthrough
with `findRCPMatches`, supplying the exact `submissionUuid` in
`input.identifiers.candidateSubmissionUuids[]` (plural), and inline the fields you
need — including the `resume` fragment for download URLs. See
references/passthrough-discovery.md § Inline resume fetch for the exact query shape.

**Resume download URLs are session-scoped.** The `downloadUrl` returned by
`CandidatePdfResume` requires an active `employers.indeed.com` browser session.
The agent cannot fetch the PDF bytes. Give the user the URL to click in their
logged-in browser, or open the candidate in the portal.

**PII discipline:** candidate tools return real people's names, contact details, and
employment history. Surface only what the user asked for; never persist applicant
data to a file or paste it into any external service.

**Known limit — per-job narrowing.** The typed candidate tools do NOT yet filter by
a single job (the server's catalogued ops send an empty job filter). To narrow by
job today, use the passthrough — see references/passthrough-discovery.md § Narrow
candidates by job. The key is `employerJobIds` (a base64 EmployerJob IRI), not `jobIds`.

## Passthrough discovery (the second core job)

When no typed tool covers what the user wants (messages, analytics, per-job
narrowing, anything new), use the GraphQL passthrough — and document what you find so
it can become a catalogued op or a typed tool later.

- `indeed_list_operations` — list the off-the-shelf catalogued operations.
- `indeed_run_operation(operation_name, variables)` — run a catalogued op by name
  (body comes from the trusted server catalogue — **prefer this** over raw GraphQL).
- `indeed_graphql(operation_name, query, variables)` — raw passthrough (reads AND
  mutations). Use only when the operation is not catalogued.
- Resource `indeed://operations` — the full catalogue.

**To find a new operation:** the surfaces, exact operation names, and variable key
shapes (sanitized, captured live) are in **references/passthrough-discovery.md** —
candidates, candidate detail, messages, analytics, account. Read it before guessing;
variable KEYS drift (e.g. `employerJobIds` not `jobIds`) and a wrong key 400s.

## Writes (documented, gated — NOT the default)

Mutation procedures take **real, hard-to-reverse actions on a live Indeed account**
and are **audited server-side** (the audit log is the only write-safety guardrail).
A "write" is any GraphQL op whose body opens with `mutation`.

Do not fire a mutation casually. Before any write: read
**references/write-procedures.md**, confirm the exact target with the user, and
follow the confirm-first checklist there. The one mutation with a confirmed shape is
`UpdateCandidateStatus` (status / milestone move); reject, shortlist, and reply are
documented by location and must have their op name confirmed via the audit log on
first use.

**Side-effect to know:** opening a *New* applicant's detail in the portal auto-moves
them to *Reviewing*. The MCP read tools are paged GraphQL reads and do NOT trigger
that UI transition — but be aware the states can differ from what a human saw.

## Examples

**"How many candidates are at interview stage?"**
1. `indeed_session_status` → alive.
2. `indeed_candidate_counts` → read the INTERVIEWED milestone count.
Done — one read, no PII dump.

## Common mistakes

- Skipping `indeed_session_status` and looping on a dead session instead of telling
  the user to re-seed.
- Reaching for `indeed_graphql` (raw) when `indeed_run_operation` (catalogued) or a
  typed read tool already covers it.
- Guessing passthrough variable keys — they drift; read references/passthrough-discovery.md.
- Treating a mutation as routine. Writes are gated, audited, and hard to reverse.
- Persisting or forwarding candidate PII beyond answering the immediate question.
- **RankerException** (`Exception calling CANDIDATE_NAIVE_SORTER`) — usually means
  the `findRCPMatches` input carries conflicting filters (e.g. `candidateSubmissionUuids`
  combined with other constraints). Recovery: drop non-essential filters and retry
  with a minimal input (just the UUID array + surface context).
- Assuming the agent can fetch a resume PDF from `downloadUrl`. It can't — the URL
  is session-scoped and the MCP has no browser cookies.
