---
name: agent-updater-user-uninstall
description: Remove a package from the user-level agent-updater setup — reverse the files it installed, delete its checkout, and remove it from ~/agent-updater/manifest.yml. Invoke when the user asks to uninstall/remove an agent-updater package at the user level.
---

# Agent Updater - Uninstall Package (User Level)

## Steps

- Ask the user which package to remove (list the entries in `~/agent-updater/manifest.yml` if not already clear from context).
- For each `{source, target}` pair in that package's `installed_files`:
  - If `target` was created solely by this package (nothing else depends on it), delete it.
  - If `target` is shared/merged content (e.g. a section this package appended into a file another package also touches), remove only the content attributable to this package rather than deleting the whole file. If it's ambiguous which content belongs to this package, stop and ask the user before deleting anything.
- Delete the local checkout at `~/agent-updater/packages/<package-name>`.
- Remove the package's entry from `~/agent-updater/manifest.yml` entirely.
- Confirm to the user what was removed, and flag anything left in place because it was ambiguous.

## Exception Handling

### Unknown package

If the requested package isn't in the manifest, tell the user and stop — don't guess at a similar name.

### Missing installed_files

If the manifest entry has no `installed_files` recorded (e.g. it predates this tracking), tell the user automatic cleanup isn't possible and ask how they'd like to proceed — manual review, or just remove the checkout and manifest entry and leave any applied files in place.
