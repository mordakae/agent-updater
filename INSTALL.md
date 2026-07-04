# Install

This tool is not Claude-specific. The `example-user-level/` and `example-repo-level/` folders show one concrete implementation, written for Claude Code's conventions (a `CLAUDE.md` always-loaded file plus an invokable `SKILL.md`). If you are a different agent (or a different harness), read past the specific filenames to the two building blocks they represent, and place them wherever your own conventions call for:

1. **An always-loaded entry point** — whatever your agent reads at the start of every session (a system prompt file, a rules file, `AGENTS.md`, etc). This should contain *only* the cheap due-check: read `update_history.yml`, compare `last_update` + `update_freq` against today (the local date; treat an unrecognised `update_freq` as `Weekly`), and if due, invoke building block 2. Keep this minimal — it's paid for on every session whether or not an update is due.
2. **An on-demand/invokable unit** — whatever mechanism your agent has for loading instructions only when needed (a skill, a tool, a sub-agent, an included file). This should contain the full manifest-checking, update/skip, and exception-handling logic. If your agent has no such on-demand mechanism, fold this into the always-loaded file instead — correctness matters more than the context-cost optimization.

Everything else below (file formats, the ignore rule) is agent-agnostic and applies regardless of which agent is doing the install.

## User-level install

- Create `~/agent-updater/manifest.yml` and `~/agent-updater/update_history.yml` (see `example-user-level/agent-updater/` for the format).
- Set up building blocks 1 and 2 above per your agent's conventions (see `example-user-level/.claude/` for the Claude Code version).
- Package checkouts live under `~/agent-updater/packages/<package>` (the same layout as the repo level)

## Repo-level install

- Create `agent-updater/manifest.yml` and `agent-updater/update_history.yml` at the repo root (see `example-repo-level/agent-updater/` for the format).
- Set up building blocks 1 and 2 above per your agent's conventions (see `example-repo-level/.claude/` for the Claude Code version).
- `agent-updater/packages/` (the actual package checkouts) must never be committed to the repo, regardless of anything else below. Before finishing install, confirm this exclusion is in place: `git check-ignore agent-updater/packages` should succeed. This check and the ignore mechanism itself are plain git, not agent-specific. See `example-repo-level/.gitignore` for a worked example.
- Ask the user where the exclusion should live:
  - **Shared with the team** — add `agent-updater/packages/` to the repo's tracked `.gitignore`, so every clone gets it automatically.
  - **Local to this checkout only** — add `agent-updater/packages/` to `.git/info/exclude` instead, so nothing is committed and other clones are unaffected.
- Separately, ask the user whether they also want the rest of `agent-updater/` (i.e. `manifest.yml` and `update_history.yml`) ignored the same way, or left tracked so the team shares the same package list and update cadence. This part is optional and up to them — only the `packages/` exclusion is mandatory.
