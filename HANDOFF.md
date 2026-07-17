# Handover — 2026-07-17

**Branch:** `main` (CI fix — #107 closed)

## What Happened This Session

Fixed CI which had been red since July 10. Root causes spanned two repos:

**casehub-pages** (5 PRs merged: #194, #197, #198, #200, #202): Added Maven
CI/publish workflow for backend modules, published `casehub-pages-push` to
GitHub Packages, bumped npm package versions to 0.2.3/0.2.1.

**drafthouse** (6 commits on main): CDI request context activation on virtual
threads (ReviewerChannelBackend) and Awaitility threads (test lambdas). CSS
layout fix — rows slot rule for host-panel children so diff panels are
height-constrained. Stale tag names in 5 tests. Smart quote stripping in
location parser. Blocks API alignment (ChannelAgentRequest 4-arg,
ThreadEntry 7-arg, ConversationPoint 5-arg, ConversationFold new signatures).
CI workflow updated to checkout and build casehub-pages for webui file:
dependencies.

508 tests pass locally. CI green on main.

## Immediate Next Step

Pick from the backlog — all items are independent.

## What's Next

| # | Title | Scale | Complexity | Notes |
|---|-------|-------|------------|-------|
| #53 | Brainstorming UI slices 3-6 | S-M each | Low-Med | Slice 3: read-only panel, Slice 4: interactive injection, Slice 5: skill integration, Slice 6: convergence view |
| #99 | Live workspace watching | M | Med | Can consume JSONL events directly |
| #97 | Chunked orchestration research | M | High | Unblocked since #96 closed |
| #93 | Document workbench (epic) | XL | High | 5 remaining child issues |
| #100 | Channel-based HIL | L | High | Blocked by #97, #99 |
| #101 | Panel extraction | XL | High | Blocked by all above |

## References

| Context | Where |
|---------|-------|
| Garden entries | GE-20260717-b31a92 (Edit tool smart quotes), GE-20260717-074283 (GitHub Packages 422 ghost) |
| Pages PRs | casehubio/casehub-pages #194, #197, #198, #200, #202 |
