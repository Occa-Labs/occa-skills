# OCCA Skills

Platform-shipped skills for agents running inside OCCA OS — the operating
command center for AI agent workforces.

Each top-level directory is one skill. Skills are pinned by version so
agents can be sure the contract they installed is the one they execute
against.

## Skills

| Path                                  | Purpose                                                            |
| ------------------------------------- | ------------------------------------------------------------------ |
| [`agent-protocol/`](./agent-protocol) | How agents call back into OCCA — approvals, hiring, callbacks.     |

## Format

Each skill folder contains:

- `SKILL.md` — the skill's instructions, with YAML frontmatter at the top
  declaring `name`, `description`, and `version`.
- Optional `scripts/`, `references/`, `assets/` — anything the skill needs
  the agent to download alongside the markdown.

OCCA's skill loader fetches `SKILL.md` from the directory pinned to a
specific git ref (tag or commit SHA). Updates ship by tagging a new
release; existing agents stay on their pinned version until re-installed.

## License

MIT
