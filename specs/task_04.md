# Task 04 – Notification Layer

Reference `specs/overview.md` for the CLI, persistence, and GitHub I/O layers, and `specs/tokens_automation.md` for the state machine that drives these notifications.

## State Transition Notifications

Each message below is posted as a GitHub **pull request comment** via the notification worker in `src/github.rs`. Timestamps should be recorded in UTC and aligned with the fjall persistence clock.

### Registered → ScheduledForSending
- **Trigger**: GitCircles discovers a merged PR and immediately posts the notification.
- **Message (Markdown)**:
  ```
  GitCircles detected this merge request for **{PROJECT_ID}**.
  Appreciation will run in **14 days** unless `{STOP_COMMAND}` is posted.
  Ergo wallet required; follow the [setup guide](https://github.com/GitCircles/GitCircles-Roadmap?tab=readme-ov-file#-setting-up-payment-your-step-by-step-guide).
  ```
- **Origin & Timing**: Posted as soon as the PR enters the Registered state; this is effectively real-time registration.
- **Terms & Conditions**: Continuing after the notice implies consent to the published policy at `https://gitcircles.io/toc.html`, which defines eligibility, wallet requirements, and the STOP override.
- **STOP Command Constant**: Define `pub const STOP_COMMAND: &str = "GitCircles STOP APPRECIATION";` (suggest `src/types.rs`) and reuse that constant anywhere the phrase is rendered or parsed so UI and automation stay in sync.
- **Audit Note**: Persist PR id, comment id, project id, and the scheduled send timestamp.

### ScheduledForSending → ProcessedWithoutStop
- **Trigger**: Fourteen days pass with no `{STOP_COMMAND}` comment and the author has a wallet.
- **Message (Markdown)**:
  ```
  ✅ GitCircles has sent **{LINE_ADDS + LINE_DELS} tokens** to [{Author}](AuthorProfileLink).
  Transaction details: [View on Blockchain Explorer](TransactionLink).
  Thank you for contributing to PROJECT_ID!
  ```
- **Origin & Timing**: PR comment published immediately after blockchain transaction finalizes.
- **Token Calculation Data**: Pull `additions` + `deletions` from the GitHub PR payload during `collect --repo` and store them as `lines_added` / `lines_removed` columns in `pull_requests`. The notification worker must rely on the persisted values instead of re-calling GitHub so historical payouts remain reproducible.
- **Audit Note**: Store tx hash, explorer link, comment id, and amount.

### ScheduledForSending → AppreciationStopped
- **Trigger**: `{STOP_COMMAND}` detected before the payout window closes.
- **Message (Markdown)**:
  ```
  ⛔ Appreciation has been cancelled per request from [{UserWhoInterrupted}](ProfileLink).
  No tokens will be sent for this merge request.
  ```
- **Origin & Timing**: PR comment posted within the same polling cycle that observed the STOP comment (≤1 hour).
- **Audit Note**: Record cancelling user, upstream comment id, and timestamp.

### ScheduledForSending → WaitingForWallet
- **Trigger**: Fourteen days pass without a `{STOP_COMMAND}` comment, but no wallet is registered.
- **Message (Markdown)**:
  ```
  GitCircles cannot send **{CalculatedTokens} tokens** because [{Author}](AuthorProfileLink) has no registered Ergo wallet.
  Appreciation will be retried automatically once a wallet is detected, or can be cancelled by commenting `{STOP_COMMAND}`.
  ```
- **Origin & Timing**: PR comment emitted at the same time the system would have paid out.
- **Audit Note**: Persist retry schedule, author login, and comment id.

### WaitingForWallet → ProcessedWithoutStop
- **Trigger**: Author registers an Ergo wallet after the waiting notice.
- **Message (Markdown)**:
  ```
  Wallet detected for [{Author}](AuthorProfileLink). GitCircles has now sent **{CalculatedTokens} tokens**.
  Transaction details: [View on Blockchain Explorer](TransactionLink).
  ```
- **Origin & Timing**: PR comment posted immediately after the wallet sync run succeeds and the payout transaction completes.
- **Audit Note**: Tie to prior WaitingForWallet entry and include tx hash + wallet proof.

### WaitingForWallet → StoppedByAuthor
- **Trigger**: Author without a wallet comments `{STOP_COMMAND}`.
- **Message (Markdown)**:
  ```
  GitCircles acknowledges the cancellation by [{Author}](AuthorProfileLink).
  No further appreciation attempts will be made for this merge request.
  ```
- **Origin & Timing**: PR comment posted within the monitoring cycle that captured the STOP command.
- **Audit Note**: Record author login, upstream STOP comment id, and terminal state.

## Notification Template Handling
- The snippets above are canonical phrasing but should ultimately live in template files (e.g., Markdown + `{placeholder}` syntax) so copy changes do not require recompilation.
- **Open Subtask**: Confirm with stakeholders whether templates should reside in repo (e.g., `specs/templates/*.md`), use a lightweight engine (Handlebars, Tera), or be sourced externally. Capture the decision and implementation plan before wiring the notification worker to file I/O.
- Until the template format is chosen, keep messages to short paragraphs with required links only, minimizing later migration work.

## Minimal Audit Requirements
- Log every outbound PR comment with `{pr_node_id, repo, comment_id, template_key, rendered_sha256, actor, state_transition, created_at}`.
- Retain a monotonic sequence number in fjall (`notifications/{repo}/{pr}/seq`) to support replay detection.
- Capture any external references (blockchain transaction hash, wallet verification artifacts) in the same record to ensure end-to-end traceability.
