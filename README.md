# Agent Updater

Agent Updater keeps an AI coding agent's own configuration — skills, rules, prompts, whatever a given package of "agent config" looks like — up to date, by having the agent do the checking and updating itself.

## The problem

Tools like Homebrew, npm, or apt all solve "keep software current," but they all rely on a human remembering to run `update`/`upgrade` and then deciding what to do about it. That works fine for software a person runs deliberately. It works less well for the growing pile of agent configuration a developer accumulates — shared skills, house rules, prompt libraries, personal automations — which nobody thinks to "check for updates" on because there's no natural moment to do it.

Agent Updater moves that responsibility onto the agent. The agent already starts a session every time you use it; that's the natural checkpoint. So instead of a CLI you have to remember to invoke, this is a small, cheap check the agent performs at the start of its own sessions, and a set of on-demand skills it invokes itself when there's something to do — clone new packages, pull updates, summarise what changed, ask you what you want, apply the result, and keep enough of a record to install packages, uninstall them, or undo a skip if you change your mind. You stay in the loop for anything risky (there's no such thing as a silent auto-apply of a shell script), but you don't have to be the one who remembers to look.

## Use cases

- **Syncing a personal "agent brain" across machines.** Keep an evolving personal knowledge repo — notes, preferences, accumulated context, the kind of running "wiki" some people now keep for their agent to consult — in one git repo, and let every machine's agent pull it into sync on its own schedule. Because the sync is just "does the agent's own session-start check see a new sha," it works even for agents that are otherwise offline or air-gapped between sessions — no always-on service required, just a git remote both ends can reach whenever a session happens to start.
- **Corporate governance hierarchy, without polluting the repo.** A manifest can list multiple packages, so an org can publish separate company-wide, division-wide, and team-wide rules repos, and every employee's agent pulls all three into their user-level config independently versioned. That keeps repo-level config strictly scoped to the project itself, instead of every repo having to vendor a copy of company policy — a real separation of concerns, not just a convenience.
- **Safe adoption of third-party agent config.** Since applying a package's changes is filtered through explicit rules (config-shaped changes only, shell scripts/out-of-directory changes need explicit approval), you can pull in someone else's published package without it being a blank check.
- **Keeping a project's third-party framework current, without a designated owner.** Say a team adopts something like Spec Kit into a repo. Normally, staying current means someone has to remember it's there, periodically check the upstream repo for changes, pull them in, and push a commit for the rest of the team — a chore that quietly stops happening once that person gets busy or moves on. As a repo-level package instead, every team member's agent independently notices drift the moment it's next due, so the check isn't riding on one person's memory. Whoever happens to be in a session when it comes due can review the changes and push the update, and since `manifest.yml`/`update_history.yml` are committed, that one push brings the whole team's checkout current — no single owner required.
- **Consistency across a fleet of different agentic platforms.** Since the mechanism is deliberately agent-agnostic (see [INSTALL.md](INSTALL.md)), it isn't tied to any one vendor's tooling. Paired with [multi-bot](https://github.com/mordakae/multi-bot), a company can run whichever agentic platforms different teams prefer while still rolling out the same governance/rules packages to all of them from a single source — the fleet stays consistent without forcing everyone onto one platform.

## How it works

Everything is built from the same primitives at two independent **levels** — user and repo — described below and in [INSTALL.md](INSTALL.md).

Each level has:
- A **manifest** (`manifest.yml`) listing packages: their git `url`, `branch` to track, the `sha` the user was last notified about, the `applied_sha` actually applied locally, whether the two are `in_sync`, and `installed_files` (a `{source, target}` map used to apply and later reverse a package's changes cleanly).
- An **update history** (`update_history.yml`): `last_update` and `update_freq` (Daily/Weekly/Monthly), which together answer "is a check due right now?"
- A cheap **always-loaded check** (e.g. Claude Code's `CLAUDE.md`) that reads `update_history.yml` and, only if due, invokes...
- An **on-demand skill** that does the real work: diff each package's stored sha against its remote, let the user choose Update All / decide individually with names only / decide individually with a change summary, force each accepted package's checkout to match the remote exactly (the checkout is agent-only — local drift is always discarded, never merged), then apply only the configuration-shaped parts of the diff.
- **Install/uninstall skills**, invoked directly by request rather than by the due-check, for adding a new package to the manifest or removing one (reversing its `installed_files` and deleting its checkout).

See `example-user-level/` and `example-repo-level/` for the concrete Claude Code implementation of both, and [INSTALL.md](INSTALL.md) for how the same shape maps onto a different agent's conventions.

### In isolation

- **User-level only**: `~/agent-updater/` holds the manifest, history, and package checkouts (under `~/agent-updater/packages/`), with config applied under the user's home directory (e.g. `~/.claude/`). This travels with the person, independent of any particular project.
- **Repo-level only**: `agent-updater/` at the repo root holds the same three things, with config applied inside the repo. Package checkouts (`agent-updater/packages/`) must never be committed — see [INSTALL.md](INSTALL.md) for the mandatory ignore step — while the manifest and history can optionally be committed so the whole team shares the same package list and cadence.

### In combination

The two levels are entirely independent — separate manifests, separate histories, separate checkouts, separate on-demand skills. Installing both means a session start pays two cheap due-checks instead of one, and each runs its own update flow only when its own `update_freq` says it's due. There's no shared state between them, which also means name-clash detection is scoped per level: a user-level package and a repo-level package that happen to install a same-named file won't be cross-checked against each other, only against other packages at the same level.

In practice this lets you split concerns cleanly: personal, cross-project tooling lives at the user level; project-specific, team-shared tooling lives at the repo level; and a given project gets the union of both without either one having to know the other exists.

### With multi-bot

[multi-bot](https://github.com/mordakae/multi-bot) is the natural companion at the repo level: Agent Updater keeps packages *current*, multi-bot keeps their content *consistent across agent platforms*. In a repo that uses multi-bot, don't install package config into platform-native locations (`.claude/`, `.cursor/`, etc.) — those are generated, gitignored outputs that multi-bot's sync will clobber or orphan. Compose the two instead:

- Package `installed_files` targets live under `.agent_config/` — rules into `rules/`, skills into `skills/`, agents into `agents/` — so the canonical source stays the single point of truth.
- After applying (or uninstalling) package changes, increment `.agent_config/agent_config_version`; multi-bot's own sync then propagates the changes to every platform on its next open.
- Agent Updater's always-loaded due-check (building block 1 in [INSTALL.md](INSTALL.md)) is itself authored as an unscoped multi-bot rule (e.g. `.agent_config/rules/agent-updater-check.md`), and its skills as multi-bot skills. That single authoring makes Agent Updater available on every platform multi-bot supports, with no per-platform hand-translation.
- Ordering at session start: the Agent Updater due-check runs *first* (it may change `.agent_config/` and bump the version), then multi-bot's version check — which then picks up any bump in the same session.

That split is the actual point, not just a tidiness win. A single global rule set that tries to cover every repo tends to drift toward the lowest common denominator — vague enough to not break anything, specific enough to help nothing. Agent Updater lets the reusable, DRY stuff (org policy, personal conventions, cross-project skills) live once and propagate everywhere via the user level, while each repo's own config stays free to be as sharp, opinionated, and bespoke as that one codebase actually needs, at the repo level. You get the maintenance benefits of shared context and the precision of atomic, per-repo control — without either one having to compromise for the other's sake.
