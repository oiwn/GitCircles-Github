# Task 01 – Model & Persistence

**Notification terminology:** the “discovery notification” is a GitCircles-authored comment on the pull request. Later tasks will describe how to create it; here we just store its identifier.

## Scope
- Capture the minimal state machine that drives appreciation for GitHub pull requests (treated as merge requests).
- Define how appreciation records are stored, updated, and related to existing fjall partitions (`pull_requests`, `user_wallets`, etc.).
- Provide enough structure that downstream tasks (scheduler, notifications, payout) can read/write state without ambiguity.

## State Catalogue
State enum `AppreciationState` (persisted as lowercase snake case strings):
- `registered` – PR detected, notification not yet posted.
- `scheduled_for_sending` – initial notification posted, countdown running.
- `waiting_for_wallet` – notification posted, countdown elapsed, wallet still missing.
- `appreciation_stopped` – STOP comment detected after scheduling, payout cancelled.
- `stopped_by_author` – STOP comment detected while waiting for wallet; no further retries.
- `processed_without_stop` – payout completed (or queued via blockchain stub) with no STOP comment.

Terminal states: `processed_without_stop`, `appreciation_stopped`, `stopped_by_author`. Other states allow further transitions.

## Transition Rules
1. `registered → scheduled_for_sending`
   - Trigger: task posts the discovery notification comment.
   - Side effects: store `notification_comment_id`, set `scheduled_at`, compute `send_after = scheduled_at + wait_period` (default 14 days).

2. `scheduled_for_sending → appreciation_stopped`
   - Trigger: STOP comment detected before `send_after`.
   - Side effects: store `stop_comment_id`, `stopped_by = {login, comment_url}`, `stopped_at`.

3. `scheduled_for_sending → waiting_for_wallet`
   - Trigger: `send_after` reached, no STOP comment, wallet missing in `user_wallets`.
   - Side effects: increment `wallet_checks` counter, timestamp `last_wallet_check_at`.

4. `waiting_for_wallet → processed_without_stop`
   - Trigger: wallet present in `user_wallets` and payout succeeds (Task 05 handles transfer).
   - Side effects: record `wallet_login`, `wallet_address`, `processed_at`, optional transaction reference.

5. `waiting_for_wallet → stopped_by_author`
   - Trigger: STOP comment detected while in waiting state.
   - Side effects: store stop metadata as in transition #2.

6. `scheduled_for_sending → processed_without_stop`
   - Trigger: `send_after` reached, wallet already present, no STOP comment.
   - Side effects: same as #4, without extra wallet lookup delay.

Re-entrancy rules:
- STOP comments only transition once; ignore additional STOP attempts after a terminal state is reached.
- Wallet re-checks in `waiting_for_wallet` replace `last_wallet_check_at` but must not reset `send_after`.

## Record Structure
Create a dedicated fjall partition: `appreciations`.
- **Key**: `appreciation:{owner}/{repo}:{pr_number}`
  - Owner/repo uses lowercase canonical form from GitHub.
- **Value** (serde struct `AppreciationRecord`):

```rust
#[derive(Serialize, Deserialize, Debug, Clone)]
pub struct AppreciationRecord {
    pub repository: String,              // "owner/repo"
    pub pr_number: u64,
    pub pr_title: String,
    pub pr_author: String,
    pub state: AppreciationState,
    pub detected_at: DateTime<Utc>,
    pub scheduled_at: Option<DateTime<Utc>>,
    pub send_after: Option<DateTime<Utc>>,
    pub notification_comment_id: Option<u64>,
    pub stop_comment_id: Option<u64>,
    pub stopped_by: Option<String>,      // GitHub login
    pub stopped_at: Option<DateTime<Utc>>,
    pub wallet_login: Option<String>,    // matches user_wallets key
    pub wallet_address: Option<String>,
    pub last_wallet_check_at: Option<DateTime<Utc>>,
    pub wallet_checks: u32,
    pub processed_at: Option<DateTime<Utc>>,
    pub transaction_ref: Option<String>, // filled by Task 05
    pub last_error: Option<String>,      // diagnostic surface for scheduler
}
```

## Supporting Indexes
- Reuse `user_wallets` to resolve wallet status; no duplication of wallet storage here.
- Scheduler reads operate via prefix scans on `appreciation:{owner}/{repo}` or whole-partition scans; no additional fjall partitions or indexes required for this task.

## Invariants
- Only one `AppreciationRecord` per PR. Creation is idempotent: re-discovery overwrites metadata but preserves state unless explicitly reset.
- `send_after` must be >= `scheduled_at` and immutable once set.
- `processed_at` implies `state == processed_without_stop` and `transaction_ref.is_some()` once blockchain integration lands.
- STOP state (`appreciation_stopped` or `stopped_by_author`) requires `stop_comment_id` + `stopped_by`.

## Interactions with Existing Data
- `pull_requests` remains the source for PR metadata; Task 01 may read from it when seeding `AppreciationRecord`.
- Wallet presence is checked via `user_wallets` (`login:github:{author_login}` key layout).
- We intentionally omit line-addition/removal counters. Per the follow-up guidance (“they should not count”), payout amounts will be provided by configuration or a fixed schedule defined in later tasks.

## Seed & Migration Notes
- Backfill routine should iterate existing merged PRs, insert `registered` records, and mark as `scheduled_for_sending` once discovery notifications are posted.
- Provide helper to upsert records: if a record exists in a terminal state, skip mutation unless an admin reset is applied (out of scope here).
