# Cross-Issue Pattern Analysis — Phase 1b

**Issue:** casehubio/drafthouse#97
**Date:** 2026-07-18

## Data Source

brainstorming-ui-decomposition review (24 issues: 7 HIGH, 12 MEDIUM, 5 LOW).
Analyzed implementor rounds 1-2 (20 fixes total).

## Findings

| Pattern | Count | % of fixes | Quality Impact |
|---------|-------|------------|---------------|
| Self-contained fixes | 18 | 90% | None |
| Same-priority references | 1 | 5% | None (preserved in chunks) |
| Cross-priority references | 1 | 5% | None (incidental) |
| Batched fixes | 0 | 0% | N/A |

## Cross-Priority References (detail)

**R1-09 (MEDIUM) → R1-15 (LOW):**
R1-09 (terminal injection race conditions) and R1-15 (querySelector coupling
across shadow boundaries) both converge on the same solution: replace
`document.querySelector("terminal-panel")` with a `brainstorm-inject` custom
event on `document`.

The implementor mentioned R1-15 when fixing R1-09: "See R1-15." However, R1-09's
fix — switching to custom events for input injection — would be identical whether
or not R1-15 was visible. The custom event pattern is the right solution for
R1-09 on its own merits (loose coupling, consistent with existing cross-panel
patterns). R1-15 just happens to benefit from the same change.

**Verdict:** Incidental reference. A chunked implementor seeing only MEDIUM items
would have produced the same fix for R1-09. When later seeing R1-15 (LOW), it
would mark it as already resolved by the R1-09 fix.

## Same-Priority Reference

**R1-04 (HIGH) → R1-05 (HIGH):**
R1-04 (wrong package name) references R1-05 (repo assignment): "Since Slice 1
now starts in DraftHouse (see R1-05)." Both are HIGH — this reference would
be preserved in chunked mode since both issues are in the same priority tier.

## Decision

**GO.** Cross-issue coupling is rare (5% of fixes) and incidental (fix quality
would not change without the cross-reference). The implementor's fixes are
overwhelmingly self-contained — each issue is addressed on its own merits
without needing visibility into other priority tiers.

The one cross-priority reference (R1-09 → R1-15) demonstrates a common
pattern: when a fix in one tier happens to resolve an issue in another tier,
the second-tier implementor simply notes "already resolved" — no quality loss.

Proceed to Phase 2: build `--chunked` mode.
