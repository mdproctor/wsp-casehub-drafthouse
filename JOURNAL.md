# Design Journal — issue-117-composable-capabilities

## 2026-08-20 — Architecture design and foundation implementation

### Design decisions (D1-D7)

Designed composable facet architecture replacing the three separate session types
(ReviewSession, DebateSession, BrainstormSession) with a thin DraftHouseSession
container and independently activatable facets. Key decisions:

- **D1:** Composable facet model over phase state machine or flat session
- **D2:** Artifacts as cross-facet integration API (files, not events or method calls)
- **D3:** Immutable voice-to-document pipeline with explicit re-run
- **D4:** Local Whisper via whisper.cpp + Quarkus FFM/Panama
- **D5:** Dynamic tool registration via Quarkus MCP ToolManager (not platform WorkerFunctionProvider — review caught the mismatch)
- **D6:** "Facet" naming to avoid 6 collisions with platform Capability types
- **D7:** STT scoped to casehub-blocks-stt optional submodule

### Foundation implementation (Plan 1)

Implemented the session container infrastructure:
- `Facet` interface, `ArtifactSpec` record, `DraftHouseSession` container (api)
- `DraftHouseSessionStore` SPI with `SessionSnapshot` (api)
- `DraftHouseSessionRegistry`, `NoOpDraftHouseSessionStore` (runtime)
- `SessionResource` REST endpoints at `/api/sessions`
- `SessionMcpTools` — 10 MCP tools for session lifecycle + document management
- Fixed pre-existing compilation failures from qhorus MessageView content/payload split
- 666 tests green (20 new + all existing)
