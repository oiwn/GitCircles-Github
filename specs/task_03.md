# Task 03 – Wallet & Payout Gate

## Scope
- Define how the CLI determines whether a pull request author has a valid Ergo wallet before sending appreciation.
- Document the behaviour when wallets are missing, added later, or change between checks.
- Establish how the appreciation amount is decided now that line-diff totals are out of scope.
- Capture scheduling expectations when the wallet gate is evaluated.

## Wallet Verification Flow
- Data source: `user_wallets` partition (`login:github:{author_login}` keys) populated by existing wallet sync commands.
- Validation rule:
  - Wallet entry must exist with `platform == "github"` and non-empty `address`.
  - The stored address must pass `WalletAddress::try_from`, enforcing Ergo P2PK format. If conversion fails, treat it as missing wallet and log an error for manual correction.
- When evaluating a record:
  1. Compose wallet lookup key from `pr_author`.
  2. If record already stores `wallet_address` and `wallet_login`, re-validate they match the current `user_wallets` value; update if changed.
  3. Persist the validated `WalletAddress` back into the appreciation record whenever it’s present.

## Behaviour by State
- `scheduled_for_sending`:
  - If wallet exists during the first evaluation after the wait period, proceed to payout (`processed_without_stop`).
  - If wallet missing, transition to `waiting_for_wallet` and record the attempt.
- `waiting_for_wallet`:
  - On every command run, re-check wallet. Once present, transition to `processed_without_stop`.
  - If wallet remains absent, leave record untouched aside from optional `last_wallet_check_at` update.
- Wallet changes:
  - If the stored wallet address differs from the one recorded in the appreciation record, update the record and proceed (no STOP required).
  - Log a WARN if the change happens after payout to support audits (future enhancement).
- Recurrence:
  - A cron job (outside this task) runs approximately every 15 minutes and invokes the CLI processing command, ensuring new wallets are picked up promptly.

## Payout Amount Determination
- Appreciation amount follows the Issue 11 rule: `total_tokens = lines_added + lines_removed`.
- Implementation notes:
  - Fetch additions and deletions via GitHub’s pull request API (or cache them when PRs are collected) and persist them with the appreciation record.
  - Store the computed total alongside raw additions/deletions so downstream steps can audit or adjust payouts.
  - If diff stats are unavailable, surface an error and keep the record pending; manual intervention or data backfill will be needed.
- Future enhancements (configurable multipliers, caps) can extend this logic later.

## Missing Wallet Handling
- When transitioning to `waiting_for_wallet`, store:
  - `last_wallet_check_at = now`
  - Increment `wallet_checks`.
  - Optionally append a note to `last_error` explaining the missing wallet (cleared once wallet found).
- Optional user notification (e.g., reminder comment) is outside this task; document as future enhancement.

## Interaction with Token Transfer
- Before calling the blockchain stub, ensure the appreciation record includes:
  - `wallet_login`, `wallet_address`
  - `lines_added`, `lines_removed`, and `token_amount = lines_added + lines_removed`
  - PR context (repo, number) for log correlation
- Expect the blockchain stub to return success/failure indicator; on failure, stay in `waiting_for_wallet` with `last_error` updated.
