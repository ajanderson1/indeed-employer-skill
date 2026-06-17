# AGENTS.md — indeed-employer

This is a Claude Code skill. Skill-builder scaffolded it; future edits happen at
its canonical clone `~/.agent-toolkit/skills/indeed-employer/`.

## Distribution

- Upstream: `github.com/ajanderson1/indeed-employer-skill`
- Install: `agent-toolkit-cli skill add ajanderson1/indeed-employer-skill`
- Ship local edits: `agent-toolkit-cli skill push indeed-employer`
- Pull upstream: `agent-toolkit-cli skill update indeed-employer`

## Files

- `SKILL.md` — the skill itself; frontmatter + body
- `references/` — supporting detail loaded on demand (optional)
- `templates/` — files this skill stamps elsewhere (rare)

## Editing

The body of SKILL.md is the contract. Keep it ≤500 lines / ≤3k words.
Heavy detail belongs in `references/`, not inline.
