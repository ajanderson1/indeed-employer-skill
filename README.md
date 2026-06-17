# Indeed Employer Skill

> Claude Code skill for working with Indeed Employer data through a custom `indeed_*` MCP server.

<p align="center">
  <strong>Jobs · Candidates · Counts · Applicant detail · GraphQL passthrough</strong>
</p>

<p align="center">
  <a href="#what-this-is">What this is</a> ·
  <a href="#requirements">Requirements</a> ·
  <a href="#capabilities">Capabilities</a> ·
  <a href="#safety-model">Safety</a> ·
  <a href="#install">Install</a>
</p>

---

## What this is

`indeed-employer` is a Claude Code skill that helps an agent use an Indeed Employer MCP server without memorising tool names, filters, or GraphQL shapes.

Ask naturally:

- “List my open jobs”
- “Show new candidates for the cleaner role”
- “Get candidate detail for this applicant”
- “How many people are at interview stage?”
- “Run an Indeed GraphQL passthrough query”

The skill routes those requests to appropriate `indeed_*` MCP tools and applies guardrails around live employer data.

## Requirements

| Requirement | Notes |
|---|---|
| Claude Code | Skill is written for Claude Code skill loading. |
| Indeed Employer account | MCP server uses an authenticated employer browser session. |
| Custom Indeed MCP server | Tools are expected to be exposed with `indeed_*` names. |
| Human re-auth path | If the Indeed session expires, a human must re-seed it. |

> This repository does **not** include the MCP server. It documents how Claude should operate once a compatible custom Indeed Employer MCP is configured.

## Capabilities

### Read-first workflows

| User request | Expected MCP tool family |
|---|---|
| Session health | `indeed_session_status` |
| Current account / permissions | `indeed_whoami` |
| Job lists and counts | `indeed_list_jobs`, `indeed_count_jobs` |
| Job detail | `indeed_job_detail` |
| Candidate lists and counts | `indeed_list_candidates`, `indeed_candidate_counts` |
| Candidate detail | `indeed_candidate_detail` |
| Notes / rejection comments | `indeed_candidate_notes` |
| Filter facets | `indeed_candidate_filter_options` |

### Passthrough discovery

When typed tools do not cover a portal capability, the skill explains how to use:

- `indeed_list_operations`
- `indeed_run_operation`
- `indeed_graphql`
- `indeed://operations`

See [`references/passthrough-discovery.md`](references/passthrough-discovery.md) for captured GraphQL operation patterns and variable shapes.

## Safety model

Indeed Employer data contains real applicant personally identifiable information (PII). This skill is intentionally conservative.

| Area | Guardrail |
|---|---|
| Session state | Always check `indeed_session_status` first. |
| Applicant PII | Show only what user asked for; do not persist candidate data. |
| Resume links | Treat download URLs as session-scoped browser links. |
| Writes / mutations | Require explicit confirmation before any live action. |
| Dead sessions | Stop and ask for human re-seed; no retry loops. |

Write procedures live in [`references/write-procedures.md`](references/write-procedures.md). Mutations can change real candidate status and may be difficult to reverse.

## Install

Copy or symlink this repository into your Claude Code skills directory, preserving this structure:

```text
indeed-employer/
├── SKILL.md
└── references/
    ├── passthrough-discovery.md
    └── write-procedures.md
```

Then configure your MCP client with a compatible Indeed Employer server exposing `indeed_*` tools.

## Suggested triggers

The skill is useful for prompts involving:

- Indeed Employer
- hiring portal data
- applicants / candidates
- CVs / resumes
- job lists and counts
- applicant status, notes, messages, or GraphQL passthrough

## Repository layout

```text
.
├── SKILL.md                         # Main Claude Code skill
├── references/
│   ├── passthrough-discovery.md     # GraphQL discovery notes
│   └── write-procedures.md          # Guarded mutation procedures
├── LICENSE
└── README.md
```

## License

MIT © AJ Anderson
