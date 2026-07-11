---
name: agent-updater-user-select
description: Change which pieces (Rules/Skills/Agents/Other) of an installed user-level agent-updater package are on disk, without an upstream update. Invoke when the user asks to add, drop, or reconfigure which parts of a user-level package are installed.
---

# Agent Updater - Re-select Package Pieces (User Level)

Re-open a package's selection on demand, without pulling any upstream update. This applies the delta between the current selection and the new one against what's on disk. It does **not** change `sha`/`applied_sha` — it is not an update.

## Steps

- Ask the user which package to reconfigure (list the entries in `~/agent-updater/manifest.yml` if not already clear from context).
- Make sure the checkout at `~/agent-updater/packages/<package-name>` exists (if missing, clone the configured `branch`, or the default branch if `branch` is blank — same as the update skill's Missing package handling). Do **not** fetch or reset to move it; use whatever HEAD is already checked out, so this stays decoupled from updates.
- Enumerate the package's current items and sort each into a section (Rules / Skills / Agents / Other) using the same Categorization rules as the `agent-updater-user-install` skill.
- For each section **in order Rules → Skills → Agents → Other** (skip empty sections), show the current selection state (`mode: all`, or the `include`/`declined` items) and ask the user what they now want:
  - Offer **"Everything"** (subscribe to all of that section's items, including future ones) versus picking specific items.
  - Use `AskUserQuestion` for the Everything-vs-choose decision; if picking and there are more items than fit as structured options, list them in prose and take the user's answer.
- Apply the delta for each section against `installed_files`:
  - **Newly selected** item (not previously installed) → apply its files, add `{source, target, category, item}` entries to `installed_files`, and remove it from `declined` if present.
  - **Dropped** item (previously installed, now deselected) → delete its `target` files, drop its entries from `installed_files`, and add it to that section's `declined` list (so a later update doesn't treat it as new).
  - **Unchanged** item → leave it alone.
  - Shell/out-of-directory items in **Other** still require explicit user approval before applying, even when newly selected.
- Rewrite the package's `selection` block to match the new choices (`{ mode: all }`, or `{ mode: subset, include: [...], declined: [...] }`).
- Leave `sha`, `applied_sha`, and `in_sync` unchanged — this is a re-selection, not an update.
- Confirm to the user what was added and what was removed.

## Exception Handling

Shares the checkout-and-git behavior of the `agent-updater-user` skill — a dirty/unexpected local checkout, git failures, a missing checkout, etc. are handled the same way here. See that skill's Exception Handling section.

### Unknown package

If the requested package isn't in the manifest, tell the user and stop — don't guess at a similar name.

### Missing selection block

If the entry has no `selection` block (it predates this tracking), treat every section as currently `mode: all` when showing the starting state, then proceed normally.
