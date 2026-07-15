# Agent Updater

A tool that lets an AI coding agent keep its *own* configuration (skills, rules, prompts — whatever a "package of agent config" looks like) up to date, by having the agent do the checking and applying itself at session start. See [README.md](README.md) for the full pitch and [INSTALL.md](INSTALL.md) for the agent-agnostic install shape.

## What this repo actually is

There is **no runnable code, build, or test suite here.** The "product" is authored prose — Markdown skills and rules that an agent *interprets and executes* — plus documentation. When you work in the `example-*` folders, read their contents as the instructions an agent will follow, not as inert text.

Deliverables, in order of importance:
- `example-user-level/` and `example-repo-level/` — the two concrete Claude Code reference implementations. These *are* the product.
- `README.md` — public explainer (problem, use cases, how it works, cost table).
- `INSTALL.md` — how to port the mechanism to any agent/harness.

## The two building blocks (the whole mechanism)

Everything reduces to these, per INSTALL.md:
1. **An always-loaded due-check** — the `.claude/CLAUDE.md` in each example. It does *only* the cheap check: read `update_history.yml`, compare `last_update` + `update_freq` (Daily/Weekly/Monthly → 1/7/30 days; anything unrecognised → treat as Weekly and say so) against today's local date, and *only if due* invoke the update skill. This is paid on every session — keep it minimal.
2. **On-demand skills** — the heavy logic, loaded only when needed. Four per level:
   - `agent-updater-{user,repo}` — the update worker invoked by the due-check.
   - `agent-updater-{user,repo}-install` — add a new package.
   - `agent-updater-{user,repo}-uninstall` — remove one (reverse `installed_files`, delete checkout).
   - `agent-updater-{user,repo}-select` — reconfigure which pieces of an installed package are on disk, with no upstream pull.

State lives in `agent-updater/manifest.yml` (packages) and `agent-updater/update_history.yml` (cadence). User level roots at `~/agent-updater/`; repo level roots at `agent-updater/` in the repo root.

## Core principles (do not violate)

- **Minimal persistent token cost.** This is *the* defining constraint. The always-loaded due-check and skill `description` frontmatter are the only things paid on every session, so keep the due-check sparse and descriptions concise. All heavy instructions must stay in skill *bodies*, which load on demand. README.md has a "Persistent Context Cost" table with figures **measured using a local tokeniser (tiktoken, `cl100k_base`)** — if you change the due-check or skill descriptions, re-measure with a local tokeniser and update the table.
- **The two levels are fully independent.** Separate manifests, histories, checkouts, and skills. Name-clash detection is scoped per level only. Don't introduce shared state between them.
- **Agent-agnostic at heart.** The Claude-specific filenames (`CLAUDE.md`, `SKILL.md`) are one implementation of the two building blocks. Keep the mechanism portable; INSTALL.md is the contract.

## Mirroring requirement

`example-user-level/` and `example-repo-level/` are **parallel implementations of the same logic.** A change to one almost always must be mirrored in the other. The only intended differences:
- Paths: `~/agent-updater/...` (user) vs `agent-updater/...` at repo root (repo).
- Apply targets: under the user's home dir (e.g. `~/.claude/`) vs inside the repo.
- Repo level has the mandatory `packages/` gitignore step; user level does not.
- Upgrade-override approval: repo level tracks it per-machine in gitignored `approvals.local.yml` because its manifest may be committed and shared; the user level's manifest is inherently single-machine, so its `upgrade.command` doubles as the approval and there is no approvals file. This divergence is deliberate — don't "fix" it by mirroring.

When editing skill logic, diff the two levels afterward to confirm they stayed in sync.

## Invariants a future change must preserve

- **Package checkouts (`packages/`) are agent-managed clones and must NEVER be committed** at the repo level (mandatory ignore — see [example-repo-level/.gitignore](example-repo-level/.gitignore) and INSTALL.md).
- **Checkouts are agent-only; local drift is discarded, never merged.** Updates force the checkout to the remote (`git fetch` → `git reset --hard origin/<branch>` → `git clean -fd`). Never stash/preserve/merge a dirty worktree.
- **Only configuration-shaped changes are applied** (file placement, settings keys) that the diff actually shows; prose in a package's docs is informational. Anything running a shell script or writing outside the package's own directory needs explicit user approval on top.
- **Confirm current platform conventions before applying config.** Because the acting agent's training may predate the host platform's current config format, install always researches the platform's current conventions for the surfaces a package touches (the install skills' *Confirm current platform conventions* step); updates do so only when the diff introduces a *new* config surface. This governs *how* the diff is placed/formatted — never license to restructure the package or go beyond the diff. Falls back to the package's own INSTALL/README layout when research is unavailable.
- **manifest sha semantics:** `sha` = last sha the user was *notified* about; `applied_sha` = what's actually applied locally; `in_sync` = whether the two agree. On Skip, advance `sha` to the remote head and set `in_sync: false`, but leave `applied_sha`. On Update, set both to head and `in_sync: true`.
- **`installed_files` is the reversal ledger** — `{source, target, category, item}` per file. `target` differs from `source` only when a name-clash rename applied. Keep it accurate on every apply/rename/delete, or uninstall/update can't clean up.
- **A package that ships its own upgrader gets an `upgrade` override, never a hand-off.** Telling the user to run the upgrade themselves defeats the tool's purpose — it only happens when they remember. Install records `{command, requires}` on the manifest entry and runs the package's install command; updates run the upgrade command on cadence *instead of* diff-and-apply, and `selection`/`installed_files` don't apply to what it generates. The command is approved once at install and **re-approved if it changes upstream** — detect that change *from the diff*, never by re-parsing the docs and string-comparing every run, or a paraphrase reintroduces a prompt on every check. **Missing external tooling is never installed silently**: help the user resolve it, Skip if unresolved. No external deps is a *portability* property, not a safety one; it never waives approval.

- **Upgrade approval is per-machine at the repo level, because that manifest is shared.** A committed `upgrade.command` records only *what* the command is, never that anyone approved it — otherwise one teammate's install authorizes unattended command execution in everyone's session. Repo level keeps approval in gitignored `agent-updater/approvals.local.yml` (`<package>: <approved command string>`), counting only while it still matches the manifest. User level has no such file — its manifest is already single-machine, so the recorded command *is* the approval.

- **Selection (`selection` block) is a standing subscription, not re-asked on updates.** `{mode: all}` auto-installs future items and reports them; `{mode: subset, include, declined}` installs only `include`, and updates prompt *only* for genuinely new items (remembering a "no" in `declined`). Empty `{}` = predates tracking = treat as all-sections `mode: all`. Only the `-select` skill re-opens existing choices.

## Conventions

- Keep the example `CLAUDE.md`/rules sparse and skill `description` fields concise.
- Skills sort a package's files into four sections in fixed order: **Rules → Skills → Agents → Other**. Preserve that ordering and the categorization rules (in the `-install` skill's "Categorization" section) across both levels.
- multi-bot integration is a supported repo-level pattern (targets under `.agent_config/`, bump `agent_config_version` after apply, due-check runs before multi-bot's sync). See the README's "With multi-bot" section before touching repo-level apply targets.

## Design history & working notes

Context from prior sessions that isn't obvious from the code:
- **The tool is self-hosting.** `agent-updater` is listed as the first package in its own example manifests, so it keeps *itself* up to date. When adding/renaming things, keep that self-reference intact.
- **It began as a platform-agnostic "have N days passed since a stored date?" check** and grew outward. Cross-platform, cross-agent portability is a founding requirement, not an afterthought — resist Claude-only or OS-only assumptions.
- **README use cases are deliberately curated.** The user rejected weak framings ("team shared config can just be stored in the repo — no value add there") and kept only ones with a genuine value-add: offline multi-device sync of a personal "agent brain," corporate governance hierarchy / separation of concerns, ownerless upkeep of a repo framework (e.g. Spec Kit), and multi-bot fleet consistency. Don't pad this section with generic use cases.
- **Working style:** the user frequently hand-edits files themselves, then asks for a review or a "fresh look" — check current file state before assuming your last edit is intact. Commit only when explicitly asked ("make a commit").
