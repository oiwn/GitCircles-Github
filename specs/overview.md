# Architecture Overview

## Purpose

GitCircles-Github bridges the GitCircles reward system with GitHub. It collects
merged pull requests, tracks repository metadata, records wallet addresses for
contributors, and exposes a CLI used by operators and automation.

## High-Level Flow
- CLI commands (project, collect, wallet, status, init, test-token) interact with application services in `src/main.rs` and supporting modules.
- Services rely on:
  - `src/github.rs` for GitHub REST calls via `octocrab` (pagination, auth, rate limits, wallet repo fetches).
  - `src/database.rs` for persistence backed by the embedded `fjall` key-value store.
  - `src/types.rs` for shared domain models and error handling.
  - `src/cli.rs` for argument parsing (`clap`) and output formatting (`comfy-table`, `indicatif` spinners).
- Data enters the fjall store, enabling idempotent syncs and project-scoped reporting.

## Module Map
- `src/main.rs` – command routing, service orchestration, program entry.
- `src/lib.rs` – module wiring and exports.
- `src/github.rs` – GitHub API wrapper, merged PR pagination, wallet repo fetch logic.
- `src/database.rs` – fjall partitions, CRUD APIs, transactional wallet updates.
- `src/types.rs` – domain structs, enums, error types, serde definitions.
- `src/cli.rs` – `clap` CLI definitions, table rendering helpers.
- `src/wallet.rs` (if present) – wallet synchronization workflow using GitHub + database layers.

## Persistence Layout
All data lives under the `gitcircles/db` fjall keyspace.

```
+--------------------+
| fjall keyspace     |
|   gitcircles/db    |
+--------------------+
        |
        +-- repositories      repo:{owner}/{repo} → metadata + project binding
        +-- pull_requests     pr:{owner}/{repo}:{number} → merged PR record
        +-- base_branch_history
        |                     base:{owner}/{repo}:{timestamp_secs} → branch change audit
        +-- projects          project:{id} → project profile
        +-- project_owners    owner:{project_id}:{username} → role assignment
        +-- user_wallets      login:{platform}:{login} → current wallet
        +-- user_wallet_history
        |                     history:{platform}:{login}:{timestamp_nanos} → wallet changes
        +-- wallet_index      wallet:{address}:{platform}:{login} → reverse lookup index
```

Relationships:
- `repositories.project_id` links repositories to projects.
- `pull_requests.repository` stores `{owner}/{repo}` to associate with `repositories`.
- Wallet partitions allow bidirectional lookup, ensuring history and address→login mapping.

## Command Surface
- Core: `init`, `collect --repo <owner/repo> [--base-branch] [--days] [--project-id]`, `status [--project-id]`, `test-token [--token]`.
- Project management: `project create`, `project list`, `project show`, `project delete`, `project add-owner`, `project remove-owner`.
- Wallet management: `wallet sync`, `wallet show`, `wallet history`, `wallet lookup`.

## Dependencies
- `octocrab` – GitHub REST client with pagination helpers.
- `fjall` – embedded persistent store (transactional batches used for wallet updates).
- `clap` – CLI definitions with derive macros.
- `serde` + `chrono` – serialization and UTC timestamp handling.
- `comfy-table` – formatted tabular output.
- `indicatif` – spinners and progress indicators during long-running operations.

## Operational Notes
- Every write currently uses `PersistMode::SyncAll` in `src/database.rs`; batching or deferred flushes may be needed for high-volume syncs.
- `project_owners` and `repositories` scans are in-memory; consider secondary indexes if lookups become hotspots.
- High-level wallet requirements: users publish an Ergo P2PK address in `gitcircles-payment-address/P2PK.pub`; the wallet sync command fetches and validates it.
