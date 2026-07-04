# Agent Updater - User Level

## These steps must be performed as the first action in a session:

- Store the initial message.
- Check the update history.
  - Read `~/agent-updater/update_history.yml`. It has two fields:
    - `last_update`: an ISO 8601 date (`YYYY-MM-DD`), or blank if never run.
    - `update_freq`: one of `Daily`, `Weekly`, `Monthly` (map to 1, 7, 30 days). If the value is anything else, treat it as `Weekly` and mention that to the user.
  - Get the current local date.
  - If the number of days between `last_update` and the current date is greater than or equal to the mapped `update_freq` threshold (or `last_update` is blank), invoke the `agent-updater-user` skill to check for and apply package updates.
  - Continue with the initial message.