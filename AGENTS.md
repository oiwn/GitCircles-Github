# AI Agent Playbook

## Read First
- Skim `specs/overview.md` for architecture, subsystem specs, and data model context before editing code.
- Note any open TODOs or FIXMEs in touched files; surface blockers in your summary.

## Quick Commands
```bash
cargo check        # fast validation while iterating
cargo test         # run full test suite
cargo fmt --all    # format before handing off
cargo clippy -- -D warnings  # lint gate used by maintainers
```

## Workflow Expectations
- Announce intended edits up front; prefer small, reviewable diffs.
- Preserve user changes already in the tree; coordinate before modifying unrelated files.
- When touching code paths with GitHub I/O or persistence, reference the relevant section in `specs/overview.md`.
- Add or update tests whenever behaviour changes or regressions are possible; document gaps if tests are impractical.

## Quality Gates
- Run `cargo fmt`, `cargo clippy -- -D warnings`, and the targeted `cargo test` scope you influenced; state anything you skipped and why.
- Capture notable error handling, concurrency, and performance risks in your final note.
- Link supporting specs or decisions in commit messages or summaries to keep context traceable.
