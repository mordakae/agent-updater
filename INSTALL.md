# Install

This tool is not Claude-specific. The `example-user-level/` and `example-repo-level/` folders show one concrete implementation, written for Claude Code's conventions (a `CLAUDE.md` always-loaded file plus an invokable `SKILL.md`). If you are a different agent (or a different harness), read past the specific filenames to the two building blocks they represent, and place them wherever your own conventions call for:

1. **An always-loaded entry point** — whatever your agent reads at the start of every session (a system prompt file, a rules file, `AGENTS.md`, etc). This should contain *only* the cheap due-check: read `update_history.yml`, compare `last_update` + `update_freq` against today (the local date; treat an unrecognised `update_freq` as `Weekly`), and if due, invoke building block 2. Keep this minimal — it's paid for on every session whether or not an update is due.
2. **An on-demand/invokable unit** — whatever mechanism your agent has for loading instructions only when needed (a skill, a tool, a sub-agent, an included file). This should contain the full manifest-checking, update/skip, and exception-handling logic. If your agent has no such on-demand mechanism, fold this into the always-loaded file instead — correctness matters more than the context-cost optimization.

Before you place either building block, check how the agent/harness you're targeting *currently* handles (1) always-loaded session context and (2) on-demand/invokable units — its conventions may have changed since this guide or your training data was written. Use its current documentation to choose the right filenames and locations for your platform.

Everything else below (file formats, the ignore rule, upgrade overrides) is agent-agnostic and applies regardless of which agent is doing the install.

## Upgrade overrides

Most packages are config files that the update flow copies in by diffing the tracked repo. Some instead ship their **own** installer/upgrader — a scaffolding CLI or update script that generates the files itself. Deferring to the package ("run `X` yourself when you want to upgrade") defeats the point, since the upgrade then only happens when the user remembers. Building block 2 must instead record the package's command as an `upgrade` block on its manifest entry (`command`, plus the `requires` list of external tooling it needs) and run it for the user when the cadence says an upgrade is due. The tracked repo still answers *whether* one is due; the command does the applying.

Three rules, wherever this is implemented:

- **The command is approved once, explicitly, at install**, and re-approved if the package changes it upstream, so a package can never swap in a new command that then runs unattended. Detect that change from the diff the update flow already reads, rather than re-parsing the docs and string-comparing every run — a paraphrase isn't a change, and prompting on one puts manual friction back in front of every check. Needing no external tooling is a portability property, not a safety one; it doesn't waive approval.
- **Approval is per-machine wherever the manifest is shared.** A committed manifest records only *what* the command is, never that anyone approved it — one person's approval must not authorize an unattended run in a teammate's session. Keep approval in a never-committed local file (`agent-updater/approvals.local.yml`, excluded as `packages/` is) holding the exact command string approved on this machine; it counts only while that string still matches the manifest's. The user level needs none of this: its manifest is already local to one machine, so the command recorded there *is* the approval.
- **Missing tooling is never installed silently.** Help the user get it, running the tool's own documented install only with their explicit approval. If it stays unresolved, treat the package as skipped so it resurfaces next check.

## User-level install

- Create `~/agent-updater/manifest.yml` and `~/agent-updater/update_history.yml` (see `example-user-level/agent-updater/` for the format).
- Set up building blocks 1 and 2 above per your agent's conventions (see `example-user-level/.claude/` for the Claude Code version).
- Package checkouts live under `~/agent-updater/packages/<package>` (the same layout as the repo level)

## Repo-level install

- Create `agent-updater/manifest.yml` and `agent-updater/update_history.yml` at the repo root (see `example-repo-level/agent-updater/` for the format).
- Set up building blocks 1 and 2 above per your agent's conventions (see `example-repo-level/.claude/` for the Claude Code version).
- `agent-updater/packages/` (the actual package checkouts) must never be committed to the repo, regardless of anything else below. Before finishing install, confirm this exclusion is in place: `git check-ignore agent-updater/packages` should succeed. This check and the ignore mechanism itself are plain git, not agent-specific. See `example-repo-level/.gitignore` for a worked example.
- If any package uses an [upgrade override](#upgrade-overrides), `agent-updater/approvals.local.yml` must be excluded too — it records what *this machine's* user approved to run, and must never be committed or shared.
- The exclusion always goes in the repo's tracked `.gitignore` — add `agent-updater/packages/` there, so every clone gets it automatically. Don't ask the user where to put it, and don't use `.git/info/exclude` or any other local-only mechanism: the checkouts must be excluded for everyone, not just this working copy.
- Separately, ask the user whether they also want the rest of `agent-updater/` (i.e. `manifest.yml` and `update_history.yml`) added to the same `.gitignore`, or left tracked so the team shares the same package list and update cadence. This part is optional and up to them — only the `packages/` exclusion is mandatory.
