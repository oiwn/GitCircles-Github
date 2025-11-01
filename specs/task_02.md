# Task 02 – Scheduler & STOP Detection

## Scope
- Define a CLI command that inspects appreciation records and advances their state—no background worker yet.
- Document how to read GitHub pull-request conversations to detect STOP intent and map it to `AppreciationState`.
- List the essential logs and error surfaces the command must produce so operators can understand failures.

## Processing Command (CLI)
- Command name TBD (e.g., `tokens process`). It runs once per invocation and exits after all work is processed or an error occurs.
- Responsibilities per run:
  - Load appreciation records that are not in a terminal state.
  - For each record, fetch the latest PR metadata needed for decisions (comments, merged timestamps, etc.).
  - Apply transition rules defined in Task 01 when conditions are satisfied (STOP detected, wait period elapsed, wallet present).
- Exit behaviour: non-zero exit code on fatal GitHub or storage failures; zero otherwise.
- **Open question:** scheduling cadence and automation strategy (manual trigger vs. cron vs. other) to be decided with the team later.

## Notification Handling (Open Question)
- The command may also be responsible for posting the initial discovery notification when a record is still `registered`.
- Exact timing (immediate vs. delayed) and throttling rules need team alignment; leave as TODO in implementation.

## STOP Detection Surface
- Fetch conversation from GitHub via:
  - Issue comments `GET /repos/{owner}/{repo}/issues/{issue_number}/comments`.
  - (Optional) Review comments `GET /repos/{owner}/{repo}/pulls/{pull_number}/comments` if STOP might appear there; confirm with team.
- Detection rule:
  - Comment body must equal the exact, case-sensitive token `STOP` after trimming surrounding whitespace.
  - Only accept STOP from the pull request author (`pr_author`); repository-owner overrides are an open question.
- Stored metadata:
  - Persist the comment ID and commenter login so we know which entry triggered the state change.
  - Keep a pointer to the comment URL only if the notification layer needs to reference it (optional).
- Simple path: once a STOP is handled the record moves to the relevant terminal state, and additional STOP comments are ignored without special tracking.

## Command Flow & Transitions
- When a notification is posted (`registered → scheduled_for_sending`), set `scheduled_at` and compute `send_after = scheduled_at + WAIT_PERIOD_DAYS`.
- `scheduled_for_sending` handling:
  - If STOP found → transition to `appreciation_stopped`.
  - Else if `now >= send_after`:
    - Wallet present → transition to `processed_without_stop` and hand off to payout queue (Task 05).
    - Wallet missing → transition to `waiting_for_wallet`, update `last_wallet_check_at`, bump `wallet_checks`.
- `waiting_for_wallet` handling:
  - If STOP found → transition to `stopped_by_author`.
  - Else if wallet now present → transition to `processed_without_stop`.
  - Otherwise leave state unchanged; revisit next command invocation.

## Errors & Logging
- Log at INFO when changing state or posting notifications; include repo, PR number, and target state.
- Log WARN when STOP is detected so operators can audit cancellations.
- Log ERROR on GitHub API failures, storage errors, or payout-enqueue failures; include context necessary for retries.
- Propagate errors up to the CLI so failures surface to the caller; do not silently drop failures.
- Future enhancement: consider retry/backoff strategy once scheduling model is chosen.

## Constants & Configuration
- Wait period: 14 days, defined as a constant in code; no user configuration in this task.
- STOP token: single constant `STOP` referenced across detection logic.
- Rate-limiting and batching strategy are future concerns; for now, document GitHub’s default limits and add TODOs where implementation must be cautious.
