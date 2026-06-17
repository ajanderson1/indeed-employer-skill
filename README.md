<!--
GitHub-only. This README is rendered on the repo page on GitHub but is NEVER
loaded into the skill's runtime context. The skill itself is SKILL.md plus
references/ and templates/. Add marketing copy, screenshots, install hype,
"why this exists" prose here freely — it costs nothing at agent-load time.

Do NOT @-link this file from SKILL.md or any references/*.md. Doing so
would force-load it into Claude's context when the skill activates.
-->

# indeed-employer

> working with the Indeed Employer portal via the indeed_* MCP tools — listing jobs and candidates, reading applicant detail and counts, or discovering new GraphQL operations through passthrough

## What it does

A go-between layer for the [Indeed Employer MCP](https://indeed-mcp.ajanderson.net/mcp) (`indeed_*` tools). It turns "show me new candidates for job X" or "list my open jobs" into the right read-tool call with the right arguments, so you never have to remember tool names or filter shapes. It is **read-first** by design: the recruiting reads (jobs, candidates, counts, detail) are the core, and it also documents how to discover *new* capabilities the typed tools don't yet expose — through the GraphQL passthrough (`indeed_run_operation` / `indeed_graphql`). Mutation procedures (status change, reject, reply) are documented with loud warnings: they take **real, hard-to-reverse actions on a live Indeed account** and are audited server-side.

## Install

```bash
agent-toolkit-cli skill add ajanderson1/indeed-employer-skill
```

After install, the skill lives at `~/.agent-toolkit/skills/indeed-employer/` and Claude Code activates it on the trigger phrases declared in `SKILL.md`.

## Triggers

Activates on phrases like show me new candidates, list my Indeed jobs, candidate detail, draft a reply, indeed passthrough, run an Indeed GraphQL operation.

## Sister projects

This skill works alongside two companion repos. It functions standalone but is more useful with them.

### [`agent-toolkit-cli`](https://github.com/ajanderson1/agent-toolkit-cli) — skill manager

The lock-file-driven CLI used to install, update, and push this skill. The install command above (`agent-toolkit-cli skill add ...`) comes from there. Install it with `uv tool install --from git+https://github.com/ajanderson1/agent-toolkit-cli agent-toolkit`.

### [`conventions`](https://github.com/ajanderson1/conventions) — cross-project source of truth

Personal dev conventions: copyright (AJ Anderson), default license (MIT), GitHub-only hosting, release-please as the versioning default. This skill was scaffolded by [`skill-builder`](https://github.com/ajanderson1/skills-authoring/tree/main/skill-builder), which stamps in conventions-derived defaults at build time. The release-please workflow and config in this repo come from [`conventions/templates/release-please/`](https://github.com/ajanderson1/conventions/tree/main/templates/release-please).

## Built with

Scaffolded by [`skill-builder`](https://github.com/ajanderson1/skills-authoring/tree/main/skill-builder). To improve this skill, say "improve the indeed-employer skill" in a Claude Code session — skill-builder's improve-mode mines the current session for friction signals.

## License

MIT © AJ Anderson
