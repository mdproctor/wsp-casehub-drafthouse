# Handover — 2026-07-05

**Branch:** `main` (clean — committed, not yet pushed)

## Last Session

Tested DraftHouse live with another Claude session acting as MCP reviewer against rule-engine sample docs (`~/drafthouse-demo/sample-a.md`, `sample-b.md`). Found and fixed three categories of issue:

1. **Panel toggle broken** — pages-runtime dock-toggle handler cleared `display:flex` on restore. Fixed in pages (`a35ab3d` on `issue-101-tokens-and-push-ops`), filed casehub-pages#108.
2. **Section highlight feature** — built click-to-select on review tracker points with persistent diff bar, scroll flash, and toggle on/off. Committed as `31d45ec`, filed #90.
3. **Location matching failures** — MCP callers (LLMs) send compound locations like `"Limitations / When It's Not Worth It"` that don't match any single heading. Fixed slash-separated pattern, but other unknown formats likely remain.

## Immediate Next Step

Continue debugging #90 with **Chrome MCP** connected. The user installed Chrome MCP at session end specifically for this. The next session should:

1. Start DraftHouse server: `QUARKUS_LANGCHAIN4J_ANTHROPIC_API_KEY=dummy-for-ui-testing java -jar server/runtime/target/drafthouse-server-runner.jar`
2. Have another Claude session connect as MCP client from `~/drafthouse-demo/` and raise review points against the sample docs
3. Use Chrome MCP to inspect the browser DOM when clicking points that "do nothing" — the `console.log` diagnostics in `_findHeading` and the tracker click handler are committed and will show exactly which locations fail and why
4. Add the failing location formats as test cases in `SectionHighlightE2ETest`, then fix `_findHeading`
5. Remove diagnostic `console.log` lines once all formats are handled

**Key insight from this session:** the root cause is that LLMs send location strings that combine, paraphrase, or describe headings rather than quoting them exactly. Each format needs a test case before fixing. The `SectionHighlightVisualTest` takes Playwright screenshots to `target/visual-debug/` for visual inspection.

**MCP permissions:** `~/drafthouse-demo/.claude/settings.json` has `mcp__drafthouse__*` wildcard. The other Claude session must be restarted after settings changes to pick them up.

## What's Left

- #90 section highlight — location matching still fails for some LLM-generated formats · S · Med
- Diagnostic `console.log` in diff.js and review-tracker.js — remove after debugging · XS · Low
- Pages fix not yet pushed (on `issue-101-tokens-and-push-ops` branch in pages repo) · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #89 | Migrate from LangChain4j to platform AgentProvider | M | Med | platform#55 shipped; removes API key requirement |
| #53 | Brainstorming UI — richer option exploration | L | High | Unblocked by #75 |
| #72 | Pipeline orchestration — sequential multi-perspective | L | High | Server-side |

## Cross-Repo

- casehub-pages#108 — dock-toggle display restore bug. Fix committed locally (`a35ab3d`), not pushed. DraftHouse consumes via `file:` link so the fix is already bundled.
- casehub-pages#97 (epic), #98–#100 — pages improvements, mechanical migration when delivered.
- casehub-ledger SNAPSHOT drift — `ComplianceSupplement` class was moved between packages. Workaround: build local ledger (`mvn -f ~/claude/casehub/ledger/pom.xml install -DskipTests`) or use cached jar from `~/.m2/.../casehub-ledger-0.2-20260630.090012-236.jar`. Use `-nsu` flag on drafthouse builds to avoid re-pulling broken SNAPSHOT.

## References

- Issue: #90 (section highlight — in progress)
- Issue: #89 (AgentProvider migration — filed, not started)
- Pages issue: casehub-pages#108 (dock-toggle fix)
- Test: `SectionHighlightE2ETest` — 28 tests covering location formats
- Test: `SectionHighlightVisualTest` — screenshot tests to `target/visual-debug/`
- Fixtures: `server/runtime/src/test/resources/fixtures/sample-a.md`, `sample-b.md`
