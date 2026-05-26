# CaseHub DraftHouse — Session Handover
**Date:** 2026-05-26 — infrastructure migration + Electron restoration

---

## What Was Done This Session

Full infrastructure migration: `mdproctor/md-compare` → `casehubio/drafthouse`. 14-task plan executed via subagent-driven development. GitHub repo transferred, fork created at `mdproctor/drafthouse`. Maven artifacts renamed (`io.casehub:casehub-drafthouse`), Java packages moved to `io.casehub.drafthouse`. CaseHub parent POM, CI dispatch chain, build scripts, website, and all platform docs updated. Workspace scaffold created. Post-review fixes: `distributionManagement` for GitHub Packages, deprecated CORS property removed, stale test fixture name updated. Issue #15 closed.

Electron shell restored (had been incorrectly removed during migration). Restored `java-server.js`, `main.js`, `preload.js`, `package.json` with DraftHouse naming. Made `index.html` dual-mode: Electron (IPC, native dialog, drop zones) and browser (relative URLs, prompt, query params). Issue #16 closed.

## Cross-repo changes made

- **casehub/parent**: BOM entry, PLATFORM.md, APPLICATIONS.md, deep-dive doc, CI workflows, build-all.sh, dashboard index — all committed and pushed
- **casehubio.github.io**: SVG architecture diagram + project card — committed and pushed
- **casehubio/qhorus**: #203 filed (dispatch chain gap — Qhorus doesn't dispatch to DraftHouse)

## Immediate Next Step

Start the **Sparge follow-on**: update Sparge references that pointed at md-compare (symlinks, shared node_modules, CLAUDE.md mentions). This was explicitly noted as out-of-scope for the migration plan.

## What's Left

- Qhorus #203: dispatch chain gap (Qhorus needs workflow to dispatch to DraftHouse on publish)
- Sparge follow-on: update symlinks and references that pointed at md-compare
- Playwright E2E tests: need migration to Java Playwright (npm Playwright removed, Electron stays)

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| 1 | Critique feature (LLM-driven document review via LangChain4j) | L | High | Research spec at `docs/superpowers/specs/2026-05-26-document-review-tool-research.md` |
| 2 | Sparge follow-on cleanup | S | Low | Symlinks, node_modules, CLAUDE.md references |
| 3 | Playwright test migration to Java Playwright | M | Med | Replace npm Playwright with Quarkus-native Java Playwright |

## References

| What | Path |
|------|------|
| Migration plan | `docs/superpowers/plans/2026-05-26-drafthouse-infrastructure-move.md` |
| Research spec | `docs/superpowers/specs/2026-05-26-document-review-tool-research.md` |
| Parent deep-dive | `casehub/parent/docs/repos/casehub-drafthouse.md` |
