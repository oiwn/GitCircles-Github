# Overview Issue-11: GitCircles Token Automation Process

GitCircles automates token rewards for merge requests (MRs) when they are
closed. This document describes the state machine and workflow for processing
MRs and distributing tokens to contributors.

# State Definitions and Transitions

```
stateDiagram-v2
    [*] --> Registered
    Registered --> ScheduledForSending: Notification published
    ScheduledForSending --> ProcessedWithoutStop: 14 days passed, no STOP, wallet exists
    ScheduledForSending --> AppreciationStopped: STOP comment found
    ScheduledForSending --> WaitingForWallet: 14 days passed, no STOP, no wallet
    WaitingForWallet --> ProcessedWithoutStop: Wallet detected
    WaitingForWallet --> StoppedByAuthor: STOP comment found
    ProcessedWithoutStop --> [*]
    AppreciationStopped --> [*]
    StoppedByAuthor --> [*]
```

# Detailed State Descriptions

1. Registered State
  - Default state for all discovered merge requests
  - Assigned when GitCircles first detects a merge request, regardless of age
  - Transition: Moves to ScheduledForSending when notification is published
  - Notification template:
  ```
    The Merge Request has been discovered by [GitCircles](https://gitcircles.io/projects/PROJECT_ID).
    The author's contribution will be automatically appreciated in XX days.
    [Repository Owner](https://github.com/owner_profile) or [Merge Request Author](https://github.com/author_profile)
    can cancel the appreciation process. To cancel, comment `GitCircles STOP APPRECIATION` below.
    Leave this message to proceed with appreciation and agree with [GitCircles Terms of Conditions](https://gitcircles.io/toc.html).
    Successful appreciation requires the author to have a registered Ergo wallet.
    Register your wallet by following the [instructions](https://github.com/GitCircles/GitCircles-Roadmap?tab=readme-ov-file#-setting-up-payment-your-step-by-step-guide).
  ```
  - XX: 14 days by default (configurable by repository owner in the future)

2. Scheduled for Token Sending
  - Active state when notification message is successfully published

3. Processed Without STOP

  - Trigger: 14-day period expires with no STOP comment and author has a registered wallet
  - Process:
    - Sync author's Ergo wallet address from GitHub account
    - Calculate appreciation amount (XX): Total tokens = Lines added + Lines removed
    - Issue a blockchain transaction for the calculated amount
    - Publish success notification:
    ```
      GitCircles has successfully sent appreciation of (XX tokens)[BlockchainExplorerLink]
      to [Author](AuthorProfileLink) for this merge request.
      Transaction details: [View on Blockchain Explorer](TransactionLink).
    ```
  - Move to completed state after transaction publication

4. Appreciation Stopped

  - Trigger: STOP comment detected during daily monitoring
  - Action: Publish cancellation message:
  ```
    GitCircles will not send appreciation of (XX tokens)[BlockchainExplorerLink]
    for this merge request due to manual interruption by [UserWhoInterrupted](ProfileLink).
  ```
  - Final state after message publication

5. Waiting for Wallet Registration

  - Trigger: 14 days pass with no STOP comment but author has no registered wallet
  - Action: Publish notification:
  ```
    GitCircles cannot send appreciation of (XX tokens)[BlockchainExplorerLink]
    because [Author](AuthorProfileLink) has no registered Ergo wallet.
    Appreciation will be sent automatically when a wallet is detected.
    Author can cancel future attempts by commenting `GitCircles STOP APPRECIATION`.
  ```
  - Monitoring: GitCircles continuously checks author's profile for wallet registration

6. Stopped by Author Cancellation

  - Trigger: Author without wallet explicitly cancels via STOP comment
  - Action: Publish acknowledgement message:
  ```
    GitCircles has acknowledged the cancellation request from [Author](AuthorProfileLink).
    No further appreciation attempts will be made for this merge request.
  ```
  - Final state with no further action
  
# Testing Requirements

For proper testing implementation:

  - Test Token: Create a dedicated test token on test blockchain
  - Ergo Node Connection: Establish secure connection to Ergo testnet node
  - Encrypted Key Storage: Implement secure storage for:
    - API keys
    - Wallet addresses
    - Transaction credentials
  - Mock Environment: Set up test GitHub repository with webhook simulations

# Key Implementation Considerations

- Security: All blockchain interactions must use encrypted communications
- Error Handling: Robust error handling for wallet detection failures
- Logging: Comprehensive logging for audit trails and debugging
- Rate Limiting: Implement appropriate rate limiting for GitHub API calls

# Conclusion

The automated token system provides a fair, transparent way to reward
contributors while maintaining repository owner control through the STOP
mechanism and ensuring author consent through wallet registration.


