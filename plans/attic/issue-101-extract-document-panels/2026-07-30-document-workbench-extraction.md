# Document Workbench Extraction — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> executing-plans to implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural editing.
> Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #101 — Extract document panels as reusable @casehubio components for cross-app embedding
**Issue group:** #101

**Goal:** Extract all 9 DraftHouse Lit panels into `@casehubio/blocks-ui-document-workbench`, add showcase gallery pages, and migrate DraftHouse to consume the extracted package.

**Architecture:** Single blocks-ui component package containing all 9 panels with shared types. Panels adapt CSS variables to pages tokens with fallbacks, and panels making HTTP calls gain a configurable `apiBaseUrl` property. DraftHouse becomes a consumer via the existing Maven SNAPSHOT WebJar pattern.

**Tech Stack:** Lit 3.x, TypeScript 5.x, vitest 4.x, Vite 6.x, `@casehubio/pages-component` for event subscription

## Global Constraints

- Tag `channel-feed` is renamed to `debate-feed` to avoid collision with `blocks-channel-feed`
- CSS vars use pages tokens where equivalents exist, drafthouse vars kept with fallbacks where not
- `marked` is a peer dependency — not bundled into the package
- No barrel re-export from `blocks-ui-core` (GE-20260729-f3f3a1 double-registration gotcha)
- IntelliJ MCP mandatory for all .ts file operations — open workspace with both repos:
  `ide_open_workspace({"modules": ["/Users/mdproctor/claude/casehub/drafthouse", "/Users/mdproctor/claude/casehub/blocks-ui"]})`
  Use `project_path` for every subsequent call. If IntelliJ becomes unavailable, STOP.

---

### Task 1: Package scaffold + shared types + debate-feed

Establishes the full component package pattern with one panel end-to-end.
Every subsequent task follows the recipe proven here.

**Files:**
- Create: `blocks-ui/components/document-workbench/package.json`
- Create: `blocks-ui/components/document-workbench/tsconfig.json`
- Create: `blocks-ui/components/document-workbench/tsconfig.build.json`
- Create: `blocks-ui/components/document-workbench/vitest.config.ts`
- Create: `blocks-ui/components/document-workbench/src/types.ts`
- Create: `blocks-ui/components/document-workbench/src/debate-feed.ts`
- Create: `blocks-ui/components/document-workbench/src/debate-feed.test.ts`
- Create: `blocks-ui/components/document-workbench/src/index.ts`

**Interfaces:**
- Produces: `DebateStreamEntry`, `Snapshot`, `TrailHighlight`, `BrainstormOptionData`, `OptionsPayload`, `ConvergedPayload` — shared types used by all subsequent panels
- Produces: `<debate-feed>` custom element — tag name and API contract

- [ ] **Step 1: Create package.json**

```json
{
  "name": "@casehubio/blocks-ui-document-workbench",
  "version": "0.1.0",
  "description": "Document workbench panels — diff viewer, debate feed, review tracker, timeline, brainstorm",
  "repository": {
    "type": "git",
    "url": "https://github.com/casehubio/blocks-ui.git"
  },
  "publishConfig": {
    "registry": "https://npm.pkg.github.com"
  },
  "type": "module",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "scripts": {
    "build": "tsc -p tsconfig.build.json",
    "typecheck": "tsc --noEmit",
    "test": "vitest run",
    "clean": "rimraf dist"
  },
  "dependencies": {
    "lit": "^3.3.3"
  },
  "peerDependencies": {
    "@casehubio/pages-component": "*",
    "marked": "^15.0.0"
  },
  "peerDependenciesMeta": {
    "marked": { "optional": true }
  },
  "devDependencies": {
    "@types/node": "^22.0.0",
    "jsdom": "^29.1.1",
    "rimraf": "^6.1.0",
    "typescript": "^5.6.0",
    "vitest": "^4.1.9"
  },
  "license": "Apache-2.0"
}
```

- [ ] **Step 2: Create tsconfig.json**

```json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "outDir": "dist",
    "rootDir": "src",
    "experimentalDecorators": true,
    "useDefineForClassFields": false
  },
  "include": ["src"]
}
```

- [ ] **Step 3: Create tsconfig.build.json**

```json
{
  "extends": "./tsconfig.json",
  "exclude": ["src/**/*.test.ts"]
}
```

- [ ] **Step 4: Create vitest.config.ts**

```typescript
import { defineConfig } from 'vitest/config';
import path from 'path';
import { existsSync } from 'fs';

export default defineConfig({
  resolve: {
    alias: [
      ...(existsSync(path.resolve(__dirname, '../../../pages/packages/pages-component/src')) ? [{ find: /^@casehubio\/pages-component\/dist\/(.*)/, replacement: path.resolve(__dirname, '../../../pages/packages/pages-component/src/$1') }] : []),
      ...(existsSync(path.resolve(__dirname, '../../../pages/packages/pages-component/src')) ? [{ find: '@casehubio/pages-component', replacement: path.resolve(__dirname, '../../../pages/packages/pages-component/src') }] : []),
    ],
  },
  esbuild: {
    target: 'es2022',
    tsconfigRaw: {
      compilerOptions: {
        experimentalDecorators: true,
        useDefineForClassFields: false,
      },
    },
  },
  test: {
    environment: 'jsdom',
    globals: true,
  },
});
```

- [ ] **Step 5: Create types.ts**

Consolidate all shared interfaces from the 9 drafthouse panels. Read each panel to extract its interfaces (using `ide_file_structure` on each panel file), then write the consolidated types file.

```typescript
export interface DebateStreamEntry {
  entryType: string;
  content: string;
  round: number;
  agentRole: string;
  timestamp?: string;
  pointId?: string;
  priority?: string;
  scope?: string;
  location?: string;
  commitHash?: string;
  documentPath?: string;
}

export interface Snapshot {
  label: string;
  round: number;
  commitHash: string;
  documentPath: string;
}

export interface TrailHighlight {
  raiseRound: number | null;
  fixRound: number | null;
  verifyRound: number | null;
}

export interface BrainstormOptionData {
  id: string;
  title: string;
  description: string;
  tradeoffs: string;
  status: string;
}

export interface OptionsPayload {
  sessionId: string;
  options: BrainstormOptionData[];
  state: string;
}

export interface ConvergedPayload extends OptionsPayload {
  recommendation?: string;
  selectedOptionId?: string;
}

export interface WorkspaceProgress {
  agentName?: string;
  status?: string;
  cost?: number;
  elapsed?: number;
  terminal?: boolean;
}

export interface DocumentEntry {
  path: string;
  label: string;
  slot?: 'A' | 'B';
}

export interface ComparisonState {
  pathA: string;
  pathB: string;
}
```

Review the actual panels for any additional interfaces needed. The above are derived from the brainstorming analysis — verify each against the source before writing.

- [ ] **Step 6: Write debate-feed.test.ts**

```typescript
import { describe, it, expect, afterEach } from 'vitest';
import './debate-feed.js';
import type { DebateStreamEntry } from './types.js';

function entry(round: number, overrides: Partial<DebateStreamEntry> = {}): DebateStreamEntry {
  return {
    entryType: 'RAISE', content: `Point from round ${round}`,
    round, agentRole: 'REV', timestamp: '2026-07-30T12:00:00Z',
    ...overrides,
  };
}

afterEach(() => { document.body.innerHTML = ''; });

describe('debate-feed', () => {
  it('renders placeholder when not configured', async () => {
    const el = document.createElement('debate-feed') as any;
    document.body.appendChild(el);
    await el.updateComplete;

    const placeholder = el.shadowRoot!.querySelector('.placeholder');
    expect(placeholder).toBeTruthy();
    expect(placeholder!.textContent).toContain('Waiting');
  });

  it('renders empty state when configured with no entries', async () => {
    const el = document.createElement('debate-feed') as any;
    el.configure({ debateSessionId: 'test-session' });
    document.body.appendChild(el);
    await el.updateComplete;

    const placeholder = el.shadowRoot!.querySelector('.placeholder');
    expect(placeholder).toBeTruthy();
    expect(placeholder!.textContent).toContain('No entries');
  });

  it('renders entries grouped by round', async () => {
    const el = document.createElement('debate-feed') as any;
    el.configure({ debateSessionId: 'test-session' });
    el._entries = [
      entry(1, { entryType: 'RAISE', content: 'First point' }),
      entry(1, { entryType: 'COUNTER', agentRole: 'IMP', content: 'Counter' }),
      entry(2, { entryType: 'RAISE', content: 'Second round' }),
    ];
    document.body.appendChild(el);
    await el.updateComplete;

    const dividers = el.shadowRoot!.querySelectorAll('.round-divider');
    expect(dividers.length).toBe(2);
    expect(dividers[0].textContent).toContain('Round 1');
    expect(dividers[1].textContent).toContain('Round 2');
  });

  it('applies entry type class for styling', async () => {
    const el = document.createElement('debate-feed') as any;
    el.configure({ debateSessionId: 'test-session' });
    el._entries = [entry(1, { entryType: 'DISPUTE' })];
    document.body.appendChild(el);
    await el.updateComplete;

    const entryEl = el.shadowRoot!.querySelector('.entry-dispute');
    expect(entryEl).toBeTruthy();
  });

  it('renders RESTART_CONTEXT as separator', async () => {
    const el = document.createElement('debate-feed') as any;
    el.configure({ debateSessionId: 'test-session' });
    el._entries = [entry(1, { entryType: 'RESTART_CONTEXT', content: '' })];
    document.body.appendChild(el);
    await el.updateComplete;

    const restart = el.shadowRoot!.querySelector('.entry-restart_context');
    expect(restart).toBeTruthy();
    expect(restart!.textContent).toContain('session branched');
  });

  it('shows human badge for HUMAN agent role', async () => {
    const el = document.createElement('debate-feed') as any;
    el.configure({ debateSessionId: 'test-session' });
    el._entries = [entry(1, { agentRole: 'HUMAN', entryType: 'COMMENT' })];
    document.body.appendChild(el);
    await el.updateComplete;

    const agent = el.shadowRoot!.querySelector('.entry-agent.human');
    expect(agent).toBeTruthy();
  });

  it('dispatches point-selected on entry click', async () => {
    const el = document.createElement('debate-feed') as any;
    el.configure({ debateSessionId: 'test-session' });
    el._entries = [entry(1, { pointId: 'pt-1', location: '§3.2' })];
    document.body.appendChild(el);
    await el.updateComplete;

    let selectedDetail: any = null;
    el.addEventListener('point-selected', (e: CustomEvent) => { selectedDetail = e.detail; });

    const entryEl = el.shadowRoot!.querySelector('.entry');
    entryEl!.click();

    expect(selectedDetail).toBeTruthy();
    expect(selectedDetail.pointId).toBe('pt-1');
    expect(selectedDetail.location).toBe('§3.2');
  });
});
```

- [ ] **Step 7: Run test to verify it fails**

Run: `cd /Users/mdproctor/claude/casehub/blocks-ui/components/document-workbench && npx vitest run --reporter=verbose 2>&1 | tail -20`

Expected: FAIL — `debate-feed.js` module not found.

- [ ] **Step 8: Create debate-feed.ts**

Copy drafthouse `channel-feed.ts` to `debate-feed.ts`. Apply these changes:
1. Rename `@customElement('channel-feed')` → `@customElement('debate-feed')`
2. Rename class `ChannelFeed` → `DebateFeed`
3. Replace `interface DebateStreamEntry` inline declaration with `import type { DebateStreamEntry } from './types.js';`
4. Apply CSS variable migration — replace each drafthouse var with its pages-token equivalent (see mapping in spec). For vars with no equivalent, use fallback syntax: `var(--sepia, var(--pages-neutral-11, #6b7280))`

Key CSS replacements (apply throughout the styles block):
- `var(--muted)` → `var(--pages-neutral-8, #9ca3af)`
- `var(--border)` → `var(--pages-neutral-5, #d4d4d4)`
- `var(--border-light)` → `var(--pages-neutral-4, #e5e7eb)`
- `var(--ink)` → `var(--pages-neutral-12, #111)`
- `var(--accent)` → `var(--pages-accent-9, #6366f1)`
- `var(--accent-tint)` → `var(--pages-accent-2, #e0e7ff)`
- `var(--error)` → `var(--pages-error-9, #dc2626)`
- `var(--warn)` → `var(--pages-warning-9, #d97706)`
- `var(--approve)` → `var(--pages-success-9, #16a34a)`
- `var(--sepia)` → `var(--sepia, var(--pages-neutral-11, #6b7280))`
- `var(--chrome)` → `var(--chrome, var(--pages-neutral-2, #f5f5f5))`
- `var(--human-badge, #e67e22)` → `var(--human-badge, #e67e22)` (keep as-is)
- `background: white` → `background: var(--pages-neutral-1, #fafafa)`
- `background: #fff8f0` → `background: var(--pages-warning-2, #fef3c7)`

- [ ] **Step 9: Create initial index.ts barrel**

```typescript
export { DebateFeed } from './debate-feed.js';
export type { DebateStreamEntry, Snapshot, TrailHighlight, BrainstormOptionData, OptionsPayload, ConvergedPayload, WorkspaceProgress, DocumentEntry, ComparisonState } from './types.js';
```

- [ ] **Step 10: Install dependencies and run tests**

Run: `cd /Users/mdproctor/claude/casehub/blocks-ui/components/document-workbench && yarn install && npx vitest run --reporter=verbose 2>&1 | tail -30`

Expected: All 6 tests PASS.

- [ ] **Step 11: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add components/document-workbench/
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "feat(drafthouse#101): scaffold document-workbench package + debate-feed panel"
```

---

### Task 2: document-diff + document-timeline

These two panels are coupled via the `timeline-comparison-changed` DOM event.
document-diff is the largest panel (1167 lines) and needs `apiBaseUrl` + `marked`.

**Files:**
- Create: `blocks-ui/components/document-workbench/src/document-diff.ts`
- Create: `blocks-ui/components/document-workbench/src/document-diff.test.ts`
- Create: `blocks-ui/components/document-workbench/src/document-timeline.ts`
- Create: `blocks-ui/components/document-workbench/src/document-timeline.test.ts`
- Modify: `blocks-ui/components/document-workbench/src/index.ts` (add exports)

**Interfaces:**
- Consumes: `Snapshot`, `TrailHighlight`, `DebateStreamEntry` from `types.ts`
- Produces: `<document-diff>` with `apiBaseUrl` property, `<document-timeline>` custom element
- Produces: `timeline-comparison-changed` DOM event contract (detail: `{ sessionId, indexA, indexB, labelA, labelB }`)

- [ ] **Step 1: Write document-timeline.test.ts**

```typescript
import { describe, it, expect, afterEach } from 'vitest';
import './document-timeline.js';

afterEach(() => { document.body.innerHTML = ''; });

describe('document-timeline', () => {
  it('hides when no snapshots', async () => {
    const el = document.createElement('document-timeline') as any;
    document.body.appendChild(el);
    await el.updateComplete;

    expect(el.classList.contains('hidden')).toBe(true);
  });

  it('renders markers for snapshots', async () => {
    const el = document.createElement('document-timeline') as any;
    el._snapshots = [
      { label: 'Initial', round: 0, commitHash: 'abc', documentPath: 'spec.md' },
      { label: 'Round 1', round: 1, commitHash: 'def', documentPath: 'spec.md' },
    ];
    document.body.appendChild(el);
    await el.updateComplete;

    const markers = el.shadowRoot!.querySelectorAll('.marker');
    expect(markers.length).toBe(2);
  });

  it('emits timeline-comparison-changed on click', async () => {
    const el = document.createElement('document-timeline') as any;
    el._snapshots = [
      { label: 'Initial', round: 0, commitHash: 'abc', documentPath: 'spec.md' },
      { label: 'Round 1', round: 1, commitHash: 'def', documentPath: 'spec.md' },
      { label: 'Round 2', round: 2, commitHash: 'ghi', documentPath: 'spec.md' },
    ];
    document.body.appendChild(el);
    await el.updateComplete;

    let detail: any = null;
    el.addEventListener('timeline-comparison-changed', (e: CustomEvent) => { detail = e.detail; });

    const markers = el.shadowRoot!.querySelectorAll('.marker');
    markers[1].click();

    expect(detail).toBeTruthy();
    expect(detail.indexA).toBeDefined();
    expect(detail.indexB).toBeDefined();
  });

  it('auto-selects last two snapshots when two or more exist', async () => {
    const el = document.createElement('document-timeline') as any;
    el.configure({ sessionId: 's-1' });
    document.body.appendChild(el);
    await el.updateComplete;

    let detail: any = null;
    el.addEventListener('timeline-comparison-changed', (e: CustomEvent) => { detail = e.detail; });

    el._handleEntries([
      { entryType: 'ROUND_SNAPSHOT', content: 'Initial', round: 0, commitHash: 'abc', documentPath: 'spec.md' },
      { entryType: 'ROUND_SNAPSHOT', content: 'Round 1', round: 1, commitHash: 'def', documentPath: 'spec.md' },
    ]);
    await el.updateComplete;

    expect(detail).toBeTruthy();
    expect(detail.indexA).toBe(0);
    expect(detail.indexB).toBe(1);
  });
});
```

- [ ] **Step 2: Write document-diff.test.ts**

```typescript
import { describe, it, expect, afterEach } from 'vitest';
import './document-diff.js';

afterEach(() => { document.body.innerHTML = ''; });

describe('document-diff', () => {
  it('renders without errors', async () => {
    const el = document.createElement('document-diff') as any;
    document.body.appendChild(el);
    await el.updateComplete;
    expect(el).toBeTruthy();
  });

  it('accepts apiBaseUrl property', async () => {
    const el = document.createElement('document-diff') as any;
    el.apiBaseUrl = 'http://localhost:9001';
    document.body.appendChild(el);
    await el.updateComplete;
    expect(el.apiBaseUrl).toBe('http://localhost:9001');
  });

  it('defaults apiBaseUrl to empty string', async () => {
    const el = document.createElement('document-diff') as any;
    document.body.appendChild(el);
    await el.updateComplete;
    expect(el.apiBaseUrl).toBe('');
  });
});
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `cd /Users/mdproctor/claude/casehub/blocks-ui/components/document-workbench && npx vitest run --reporter=verbose 2>&1 | tail -20`

Expected: FAIL — modules not found.

- [ ] **Step 4: Create document-timeline.ts**

Copy drafthouse `document-timeline.ts`. Apply:
1. Import `Snapshot`, `TrailHighlight`, `DebateStreamEntry` from `./types.js` (remove inline declarations)
2. Apply CSS variable migration (same mapping as Task 1 Step 8)
3. Make `_handleEntries` public (test accesses it directly)

- [ ] **Step 5: Create document-diff.ts**

Copy drafthouse `document-diff.ts`. Apply:
1. Add `apiBaseUrl` property: `@property({ type: String }) apiBaseUrl = '';`
2. Prepend `${this.apiBaseUrl}` to all fetch URLs:
   - `fetch(\`/api/file?path=...\`)` → `fetch(\`${this.apiBaseUrl}/api/file?path=...\`)`
   - `fetch(\`/api/debate/${id}/snapshot/${index}\`)` → `fetch(\`${this.apiBaseUrl}/api/debate/${id}/snapshot/${index}\`)`
3. Apply CSS variable migration
4. Import from `marked` stays unchanged (peer dependency)

- [ ] **Step 6: Update index.ts barrel**

Add exports:
```typescript
export { DocumentDiff } from './document-diff.js';
export { DocumentTimeline } from './document-timeline.js';
```

- [ ] **Step 7: Run tests**

Run: `cd /Users/mdproctor/claude/casehub/blocks-ui/components/document-workbench && npx vitest run --reporter=verbose 2>&1 | tail -30`

Expected: All tests PASS (previous + new).

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add components/document-workbench/src/
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "feat(drafthouse#101): add document-diff + document-timeline panels"
```

---

### Task 3: review-tracker

Has HTTP calls (needs `apiBaseUrl`), emits `point-selected`/`point-deselected` events,
derives status per pointId from the debate event stream. Mid-sized panel (536 lines).

**Files:**
- Create: `blocks-ui/components/document-workbench/src/review-tracker.ts`
- Create: `blocks-ui/components/document-workbench/src/review-tracker.test.ts`
- Modify: `blocks-ui/components/document-workbench/src/index.ts` (add export)

**Interfaces:**
- Consumes: `DebateStreamEntry` from `types.ts`
- Produces: `<review-tracker>` with `apiBaseUrl` property
- Produces: `point-selected`, `point-deselected` DOM events

- [ ] **Step 1: Write review-tracker.test.ts**

```typescript
import { describe, it, expect, afterEach } from 'vitest';
import './review-tracker.js';
import type { DebateStreamEntry } from './types.js';

function entry(overrides: Partial<DebateStreamEntry> = {}): DebateStreamEntry {
  return {
    entryType: 'RAISE', content: 'Test point', round: 1,
    agentRole: 'REV', pointId: 'pt-1', priority: 'HIGH',
    ...overrides,
  };
}

afterEach(() => { document.body.innerHTML = ''; });

describe('review-tracker', () => {
  it('renders placeholder when not configured', async () => {
    const el = document.createElement('review-tracker') as any;
    document.body.appendChild(el);
    await el.updateComplete;

    const placeholder = el.shadowRoot!.querySelector('.placeholder');
    expect(placeholder).toBeTruthy();
  });

  it('renders review points from debate entries', async () => {
    const el = document.createElement('review-tracker') as any;
    el.configure({ debateSessionId: 'test-session' });
    el._entries = [
      entry({ pointId: 'pt-1', content: 'Missing error handling' }),
      entry({ pointId: 'pt-2', content: 'Race condition risk' }),
    ];
    document.body.appendChild(el);
    await el.updateComplete;

    const points = el.shadowRoot!.querySelectorAll('.review-point');
    expect(points.length).toBeGreaterThanOrEqual(2);
  });

  it('accepts apiBaseUrl property', async () => {
    const el = document.createElement('review-tracker') as any;
    el.apiBaseUrl = 'http://localhost:9001';
    document.body.appendChild(el);
    await el.updateComplete;
    expect(el.apiBaseUrl).toBe('http://localhost:9001');
  });

  it('defaults apiBaseUrl to empty string', async () => {
    const el = document.createElement('review-tracker') as any;
    document.body.appendChild(el);
    await el.updateComplete;
    expect(el.apiBaseUrl).toBe('');
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd /Users/mdproctor/claude/casehub/blocks-ui/components/document-workbench && npx vitest run src/review-tracker.test.ts --reporter=verbose 2>&1 | tail -20`

Expected: FAIL — module not found.

- [ ] **Step 3: Create review-tracker.ts**

Copy drafthouse `review-tracker.ts`. Apply:
1. Import types from `./types.js`
2. Add `@property({ type: String }) apiBaseUrl = '';`
3. Prepend `${this.apiBaseUrl}` to all fetch URLs:
   - `POST /api/debate/${id}/human/comment`
   - `POST /api/debate/${id}/human/override`
   - `POST /api/debate/${id}/human/prioritise`
   - `POST /api/debate/${id}/human/batch`
4. Apply CSS variable migration

- [ ] **Step 4: Update index.ts**

Add: `export { ReviewTracker } from './review-tracker.js';`

- [ ] **Step 5: Run tests**

Run: `cd /Users/mdproctor/claude/casehub/blocks-ui/components/document-workbench && npx vitest run --reporter=verbose 2>&1 | tail -30`

Expected: All tests PASS.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add components/document-workbench/src/
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "feat(drafthouse#101): add review-tracker panel"
```

---

### Task 4: Remaining 5 panels (context-gauge, doc-picker, brainstorm-options, brainstorm-picker, workspace-status)

Mechanical extraction — same pattern as above. doc-picker and brainstorm-options have HTTP calls.

**Files:**
- Create: `blocks-ui/components/document-workbench/src/context-gauge.ts`
- Create: `blocks-ui/components/document-workbench/src/doc-picker.ts`
- Create: `blocks-ui/components/document-workbench/src/brainstorm-options.ts`
- Create: `blocks-ui/components/document-workbench/src/brainstorm-picker.ts`
- Create: `blocks-ui/components/document-workbench/src/workspace-status.ts`
- Create: `blocks-ui/components/document-workbench/src/remaining-panels.test.ts`
- Modify: `blocks-ui/components/document-workbench/src/index.ts` (add exports)

**Interfaces:**
- Consumes: `BrainstormOptionData`, `OptionsPayload`, `ConvergedPayload`, `WorkspaceProgress`, `DocumentEntry`, `ComparisonState` from `types.ts`
- Produces: 5 custom elements: `<context-gauge>`, `<doc-picker>`, `<brainstorm-options>`, `<brainstorm-picker>`, `<workspace-status>`

- [ ] **Step 1: Write remaining-panels.test.ts**

```typescript
import { describe, it, expect, afterEach } from 'vitest';
import './context-gauge.js';
import './doc-picker.js';
import './brainstorm-options.js';
import './brainstorm-picker.js';
import './workspace-status.js';

afterEach(() => { document.body.innerHTML = ''; });

describe('context-gauge', () => {
  it('renders without errors', async () => {
    const el = document.createElement('context-gauge') as any;
    document.body.appendChild(el);
    await el.updateComplete;
    expect(el.shadowRoot).toBeTruthy();
  });

  it('displays progress bar', async () => {
    const el = document.createElement('context-gauge') as any;
    el._percent = 65;
    document.body.appendChild(el);
    await el.updateComplete;
    const bar = el.shadowRoot!.querySelector('.gauge-fill');
    expect(bar).toBeTruthy();
  });
});

describe('doc-picker', () => {
  it('renders without errors', async () => {
    const el = document.createElement('doc-picker') as any;
    document.body.appendChild(el);
    await el.updateComplete;
    expect(el.shadowRoot).toBeTruthy();
  });

  it('accepts apiBaseUrl property', async () => {
    const el = document.createElement('doc-picker') as any;
    el.apiBaseUrl = 'http://localhost:9001';
    document.body.appendChild(el);
    await el.updateComplete;
    expect(el.apiBaseUrl).toBe('http://localhost:9001');
  });
});

describe('brainstorm-options', () => {
  it('renders without errors', async () => {
    const el = document.createElement('brainstorm-options') as any;
    document.body.appendChild(el);
    await el.updateComplete;
    expect(el.shadowRoot).toBeTruthy();
  });

  it('accepts apiBaseUrl property', async () => {
    const el = document.createElement('brainstorm-options') as any;
    el.apiBaseUrl = 'http://localhost:9001';
    document.body.appendChild(el);
    await el.updateComplete;
    expect(el.apiBaseUrl).toBe('http://localhost:9001');
  });
});

describe('brainstorm-picker', () => {
  it('renders without errors', async () => {
    const el = document.createElement('brainstorm-picker') as any;
    document.body.appendChild(el);
    await el.updateComplete;
    expect(el.shadowRoot).toBeTruthy();
  });
});

describe('workspace-status', () => {
  it('renders without errors', async () => {
    const el = document.createElement('workspace-status') as any;
    document.body.appendChild(el);
    await el.updateComplete;
    expect(el.shadowRoot).toBeTruthy();
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd /Users/mdproctor/claude/casehub/blocks-ui/components/document-workbench && npx vitest run src/remaining-panels.test.ts --reporter=verbose 2>&1 | tail -20`

Expected: FAIL — modules not found.

- [ ] **Step 3: Extract all 5 panels**

For each panel, copy from drafthouse and apply:

**context-gauge.ts** (134 lines, no HTTP):
- Import types if needed
- CSS variable migration

**doc-picker.ts** (271 lines, HTTP):
- Import types from `./types.js`
- Add `@property({ type: String }) apiBaseUrl = '';`
- Prepend `${this.apiBaseUrl}` to `POST /api/debate/${id}/comparison`
- CSS variable migration

**brainstorm-options.ts** (275 lines, HTTP):
- Import `BrainstormOptionData`, `OptionsPayload`, `ConvergedPayload` from `./types.js`
- Add `@property({ type: String }) apiBaseUrl = '';`
- Prepend `${this.apiBaseUrl}` to `PATCH /api/brainstorm/${id}/options/${optionId}`
- CSS variable migration

**brainstorm-picker.ts** (169 lines, no HTTP):
- CSS variable migration only

**workspace-status.ts** (156 lines, no HTTP):
- Import `WorkspaceProgress` from `./types.js` if applicable
- CSS variable migration

- [ ] **Step 4: Update index.ts barrel**

Add all 5 exports:
```typescript
export { ContextGauge } from './context-gauge.js';
export { DocPicker } from './doc-picker.js';
export { BrainstormOptions } from './brainstorm-options.js';
export { BrainstormPicker } from './brainstorm-picker.js';
export { WorkspaceStatus } from './workspace-status.js';
```

- [ ] **Step 5: Run all tests**

Run: `cd /Users/mdproctor/claude/casehub/blocks-ui/components/document-workbench && npx vitest run --reporter=verbose 2>&1 | tail -40`

Expected: All tests PASS.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add components/document-workbench/src/
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "feat(drafthouse#101): add remaining 5 panels — context-gauge, doc-picker, brainstorm-options/picker, workspace-status"
```

---

### Task 5: Showcase gallery

Add "Document Workbench" category to the blocks-ui examples app with mock data and 9 individual pages.

**Files:**
- Create: `blocks-ui/examples/mock-data/document-workbench.ts`
- Create: `blocks-ui/examples/src/pages/document-diff-page.ts`
- Create: `blocks-ui/examples/src/pages/debate-feed-page.ts`
- Create: `blocks-ui/examples/src/pages/review-tracker-page.ts`
- Create: `blocks-ui/examples/src/pages/document-timeline-page.ts`
- Create: `blocks-ui/examples/src/pages/context-gauge-page.ts`
- Create: `blocks-ui/examples/src/pages/doc-picker-page.ts`
- Create: `blocks-ui/examples/src/pages/brainstorm-options-page.ts`
- Create: `blocks-ui/examples/src/pages/brainstorm-picker-page.ts`
- Create: `blocks-ui/examples/src/pages/workspace-status-page.ts`
- Modify: `blocks-ui/examples/src/shell.ts` (add nav category)
- Modify: `blocks-ui/examples/src/main.ts` (add page imports)
- Modify: `blocks-ui/examples/vite.config.ts` (add alias)

**Interfaces:**
- Consumes: All exported types and elements from `@casehubio/blocks-ui-document-workbench`

- [ ] **Step 1: Add Vite alias**

In `examples/vite.config.ts`, add to the `alias` array:
```typescript
{ find: '@casehubio/blocks-ui-document-workbench', replacement: resolve(__dirname, '../components/document-workbench/src') },
```

- [ ] **Step 2: Create mock data**

Create `examples/mock-data/document-workbench.ts` with:

```typescript
import type { DebateStreamEntry, Snapshot, BrainstormOptionData, DocumentEntry } from '@casehubio/blocks-ui-document-workbench';

export const MOCK_ENTRIES: DebateStreamEntry[] = [
  { entryType: 'RAISE', content: 'The retry logic in §3.2 has no backoff — under load this will hammer the downstream service.', round: 1, agentRole: 'REV', timestamp: '2026-07-30T10:00:00Z', pointId: 'pt-1', priority: 'HIGH', scope: 'reliability', location: '§3.2' },
  { entryType: 'COUNTER', content: 'The downstream service has rate limiting. Exponential backoff adds complexity without clear benefit at current scale.', round: 1, agentRole: 'IMP', timestamp: '2026-07-30T10:00:30Z', pointId: 'pt-1' },
  { entryType: 'QUALIFY', content: 'Rate limiting protects the downstream, not us. Without backoff our thread pool saturates on retries. Adding jittered backoff is 5 lines.', round: 1, agentRole: 'REV', timestamp: '2026-07-30T10:01:00Z', pointId: 'pt-1' },
  { entryType: 'AGREE', content: 'Fair point on thread pool saturation. Will add jittered exponential backoff.', round: 1, agentRole: 'IMP', timestamp: '2026-07-30T10:01:30Z', pointId: 'pt-1' },
  { entryType: 'RAISE', content: 'No input validation on the document path parameter — potential path traversal.', round: 1, agentRole: 'REV', timestamp: '2026-07-30T10:02:00Z', pointId: 'pt-2', priority: 'HIGH', scope: 'security', location: '§5.1' },
  { entryType: 'RESTART_CONTEXT', content: '', round: 2, agentRole: 'SUPERVISOR', timestamp: '2026-07-30T10:05:00Z' },
  { entryType: 'RAISE', content: 'The comparison endpoint returns full document content — consider pagination for large files.', round: 2, agentRole: 'REV', timestamp: '2026-07-30T10:05:30Z', pointId: 'pt-3', priority: 'MEDIUM', scope: 'performance', location: '§7' },
  { entryType: 'COMMENT', content: 'Agreed with pt-1 resolution. The backoff implementation looks correct.', round: 2, agentRole: 'HUMAN', timestamp: '2026-07-30T10:06:00Z', pointId: 'pt-1' },
  { entryType: 'HUMAN_OVERRIDE', content: 'Accept pt-2 as-is — path validation already handled by the file API layer.', round: 2, agentRole: 'HUMAN', timestamp: '2026-07-30T10:06:30Z', pointId: 'pt-2' },
];

export const MOCK_SNAPSHOTS: Snapshot[] = [
  { label: 'Initial', round: 0, commitHash: 'abc123', documentPath: 'spec.md' },
  { label: 'Round 1 fixes', round: 1, commitHash: 'def456', documentPath: 'spec.md' },
  { label: 'Round 2 refinement', round: 2, commitHash: 'ghi789', documentPath: 'spec.md' },
];

export const MOCK_DOC_A = `# API Design Specification

## §3.2 Retry Strategy

When a downstream call fails, the client retries up to 3 times
with no delay between attempts.

## §5.1 Document API

The \`/api/file\` endpoint accepts a \`path\` query parameter
and returns the file content as plain text.

## §7 Comparison Endpoint

Returns both documents in full for client-side diffing.
`;

export const MOCK_DOC_B = `# API Design Specification

## §3.2 Retry Strategy

When a downstream call fails, the client retries up to 3 times
with jittered exponential backoff (base 200ms, max 5s).

## §5.1 Document API

The \`/api/file\` endpoint accepts a \`path\` query parameter,
validates it against a configurable root directory, and returns
the file content as plain text.

## §7 Comparison Endpoint

Returns both documents in full for client-side diffing.
Large files (>1MB) are streamed with chunked transfer encoding.
`;

export const MOCK_BRAINSTORM_OPTIONS: BrainstormOptionData[] = [
  { id: 'opt-1', title: 'WebSocket push model', description: 'Server pushes events to all connected clients via WebSocket. Real-time, low latency.', tradeoffs: 'Requires persistent connections. More server memory per client. Connection management complexity.', status: 'RECOMMENDED' },
  { id: 'opt-2', title: 'SSE (Server-Sent Events)', description: 'One-directional server push over HTTP. Simpler than WebSocket, auto-reconnect built in.', tradeoffs: 'One-directional only. Limited browser connections per domain. No binary support.', status: 'ACTIVE' },
  { id: 'opt-3', title: 'Long polling', description: 'Client holds an open HTTP request until server has data. Compatible with all infrastructure.', tradeoffs: 'Higher latency. More HTTP overhead. Scaling requires sticky sessions or shared state.', status: 'ELIMINATED' },
  { id: 'opt-4', title: 'Periodic polling', description: 'Client polls at fixed intervals. Simplest implementation.', tradeoffs: 'High latency (up to interval duration). Wasted requests when no changes. Not suitable for real-time UX.', status: 'ELIMINATED' },
];

export const MOCK_DOCUMENTS: DocumentEntry[] = [
  { path: '/specs/api-design.md', label: 'API Design Spec', slot: 'A' },
  { path: '/specs/api-design-v2.md', label: 'API Design Spec v2', slot: 'B' },
  { path: '/specs/data-model.md', label: 'Data Model' },
];

export const MOCK_CONTEXT_PERCENT = 67;
```

- [ ] **Step 3: Create debate-feed-page.ts**

Follow the channel-activity-page pattern. Create a page component that:
- Imports mock data
- Mounts `<debate-feed>` in a container with fixed height
- Configures it with a session ID
- Feeds entries via direct property assignment (since this is a demo, not event-driven)

```typescript
import { LitElement, html, css } from 'lit';
import { customElement } from 'lit/decorators.js';
import '@casehubio/blocks-ui-document-workbench';
import { MOCK_ENTRIES } from '../../mock-data/document-workbench.js';

@customElement('blocks-example-debate-feed')
export class DebateFeedPage extends LitElement {
  static override styles = css`
    :host { display: block; padding: 24px; }
    h2 { margin: 0 0 8px; font-size: 18px; font-weight: 600; color: var(--pages-neutral-12, #111); }
    p { margin: 0 0 16px; font-size: 14px; color: var(--pages-neutral-10, #666); }
    .demo { height: 500px; border: 1px solid var(--pages-neutral-5, #e0e0e0); border-radius: 8px; overflow: hidden; }
  `;

  override render() {
    return html`
      <h2>Debate Feed</h2>
      <p>Adversarial debate conversation — entries grouped by round, colour-coded by type (raise, counter, agree, dispute). Human entries show a badge. Click an entry to emit point-selected.</p>
      <div class="demo">
        <debate-feed id="feed"></debate-feed>
      </div>
    `;
  }

  override firstUpdated() {
    const feed = this.shadowRoot!.querySelector('#feed') as any;
    feed.configure({ debateSessionId: 'demo-session' });
    feed._entries = [...MOCK_ENTRIES];
  }
}
```

- [ ] **Step 4: Create remaining 8 page files**

Follow the same pattern for each. Key variations:

- **document-diff-page.ts**: Embeds `<document-diff>`, sets `fileA`/`fileB` properties with mock markdown. Note: in showcase mode the fetch to `/api/file` won't work — configure with inline content if the panel supports it, or show a note explaining the panel requires a running backend.
- **review-tracker-page.ts**: Feeds `MOCK_ENTRIES`, shows the review point checklist.
- **document-timeline-page.ts**: Feeds snapshots via `_handleEntries` or direct property.
- **context-gauge-page.ts**: Shows gauge at 67% — tiny inline component, wrap in a mock topbar strip.
- **doc-picker-page.ts**: Shows document dropdown with `MOCK_DOCUMENTS`.
- **brainstorm-options-page.ts**: Shows option cards with `MOCK_BRAINSTORM_OPTIONS`.
- **brainstorm-picker-page.ts**: Shows session switcher dropdown with 2 mock sessions.
- **workspace-status-page.ts**: Shows agent progress indicator.

Each page is a `@customElement('blocks-example-<name>')` extending LitElement with a description header and a demo container.

- [ ] **Step 5: Update shell.ts NAV array**

Add the new category to the NAV array in `examples/src/shell.ts`, after the existing "Composed" category:

```typescript
{
  label: 'Document Workbench',
  items: [
    { id: 'document-diff', label: 'Document Diff', hash: '#document-workbench/document-diff' },
    { id: 'debate-feed', label: 'Debate Feed', hash: '#document-workbench/debate-feed' },
    { id: 'review-tracker', label: 'Review Tracker', hash: '#document-workbench/review-tracker' },
    { id: 'document-timeline', label: 'Document Timeline', hash: '#document-workbench/document-timeline' },
    { id: 'context-gauge', label: 'Context Gauge', hash: '#document-workbench/context-gauge' },
    { id: 'doc-picker', label: 'Doc Picker', hash: '#document-workbench/doc-picker' },
    { id: 'brainstorm-options', label: 'Brainstorm Options', hash: '#document-workbench/brainstorm-options' },
    { id: 'brainstorm-picker', label: 'Brainstorm Picker', hash: '#document-workbench/brainstorm-picker' },
    { id: 'workspace-status', label: 'Workspace Status', hash: '#document-workbench/workspace-status' },
  ],
},
```

- [ ] **Step 6: Update main.ts imports**

Add the 9 page imports to the bootstrap sequence in `examples/src/main.ts`:

```typescript
await import('./pages/document-diff-page.js');
await import('./pages/debate-feed-page.js');
await import('./pages/review-tracker-page.js');
await import('./pages/document-timeline-page.js');
await import('./pages/context-gauge-page.js');
await import('./pages/doc-picker-page.js');
await import('./pages/brainstorm-options-page.js');
await import('./pages/brainstorm-picker-page.js');
await import('./pages/workspace-status-page.js');
```

- [ ] **Step 7: Run the examples dev server and verify**

Run: `cd /Users/mdproctor/claude/casehub/blocks-ui/examples && npx vite --port 3000`

Open browser to `http://localhost:3000`. Verify:
- "Document Workbench" category appears in sidebar
- Each page renders the component with mock data
- No console errors
- Theme toggle (light/dark) works on the new panels

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add examples/
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "feat(drafthouse#101): add Document Workbench showcase gallery — 9 pages with mock data"
```

---

### Task 6: DraftHouse consumer migration

Delete local panels from DraftHouse and import from the extracted package.

**Files:**
- Delete: `drafthouse/server/runtime/src/main/webui/src/panels/channel-feed.ts`
- Delete: `drafthouse/server/runtime/src/main/webui/src/panels/document-diff.ts`
- Delete: `drafthouse/server/runtime/src/main/webui/src/panels/review-tracker.ts`
- Delete: `drafthouse/server/runtime/src/main/webui/src/panels/document-timeline.ts`
- Delete: `drafthouse/server/runtime/src/main/webui/src/panels/context-gauge.ts`
- Delete: `drafthouse/server/runtime/src/main/webui/src/panels/doc-picker.ts`
- Delete: `drafthouse/server/runtime/src/main/webui/src/panels/brainstorm-options.ts`
- Delete: `drafthouse/server/runtime/src/main/webui/src/panels/brainstorm-picker.ts`
- Delete: `drafthouse/server/runtime/src/main/webui/src/panels/workspace-status.ts`
- Modify: `drafthouse/server/runtime/src/main/webui/src/index.ts` (update imports)
- Modify: `drafthouse/server/runtime/src/main/webui/package.json` (add dependency)

**Interfaces:**
- Consumes: All elements from `@casehubio/blocks-ui-document-workbench`

- [ ] **Step 1: Add dependency to package.json**

In `drafthouse/server/runtime/src/main/webui/package.json`, add to `dependencies`:
```json
"@casehubio/blocks-ui-document-workbench": "*"
```

And add resolution in `resolutions`:
```json
"@casehubio/blocks-ui-document-workbench": "portal:./.casehub-packages/packages/blocks-ui-document-workbench"
```

Note: The portal resolution won't work until the blocks-ui package is published as a Maven SNAPSHOT WebJar. For local development, use a Vite alias or esbuild alias instead. Check the build tooling (`esbuild.config.mjs`) and add the appropriate alias.

Alternative for esbuild: check if drafthouse uses esbuild (it does — `esbuild.config.mjs`). Add an alias mapping `@casehubio/blocks-ui-document-workbench` to `../../../casehub/blocks-ui/components/document-workbench/src`. Read the esbuild config to determine the exact alias syntax.

- [ ] **Step 2: Update index.ts imports**

Replace the local panel imports:

```typescript
// Before:
import "./panels/document-diff.js";
import "./panels/channel-feed.js";
import "./panels/review-tracker.js";
import "./panels/context-gauge.js";
import "./panels/doc-picker.js";
import "./panels/document-timeline.js";
import "./panels/workspace-status.js";
import "./panels/brainstorm-options.js";
import "./panels/brainstorm-picker.js";

// After:
import "@casehubio/blocks-ui-document-workbench";
```

The barrel import registers all custom elements as side effects.

Note: The tag rename from `channel-feed` to `debate-feed` means any `registerPanel('channel-feed', ...)` calls or layout references in index.ts need updating to `debate-feed`. Search the full index.ts for `channel-feed` string references.

- [ ] **Step 3: Update panel registration references**

Search `index.ts` for any `registerPanel` call or layout reference using `'channel-feed'` and replace with `'debate-feed'`. Use `ide_search_text` to find all occurrences across the webui source.

- [ ] **Step 4: Delete local panel files**

Delete all 9 files from `drafthouse/server/runtime/src/main/webui/src/panels/`:
```bash
rm /Users/mdproctor/claude/casehub/drafthouse/server/runtime/src/main/webui/src/panels/channel-feed.ts
rm /Users/mdproctor/claude/casehub/drafthouse/server/runtime/src/main/webui/src/panels/document-diff.ts
rm /Users/mdproctor/claude/casehub/drafthouse/server/runtime/src/main/webui/src/panels/review-tracker.ts
rm /Users/mdproctor/claude/casehub/drafthouse/server/runtime/src/main/webui/src/panels/document-timeline.ts
rm /Users/mdproctor/claude/casehub/drafthouse/server/runtime/src/main/webui/src/panels/context-gauge.ts
rm /Users/mdproctor/claude/casehub/drafthouse/server/runtime/src/main/webui/src/panels/doc-picker.ts
rm /Users/mdproctor/claude/casehub/drafthouse/server/runtime/src/main/webui/src/panels/brainstorm-options.ts
rm /Users/mdproctor/claude/casehub/drafthouse/server/runtime/src/main/webui/src/panels/brainstorm-picker.ts
rm /Users/mdproctor/claude/casehub/drafthouse/server/runtime/src/main/webui/src/panels/workspace-status.ts
```

- [ ] **Step 5: Build and verify**

Build DraftHouse to verify the migration compiles:
```bash
/opt/homebrew/bin/mvn -f /Users/mdproctor/claude/casehub/drafthouse/server/pom.xml package -DskipTests -q -o 2>&1 | tail -10
```

If the build uses esbuild for the webui bundling, also run:
```bash
cd /Users/mdproctor/claude/casehub/drafthouse/server/runtime/src/main/webui && node esbuild.config.mjs 2>&1 | tail -10
```

Expected: Build succeeds with no errors.

- [ ] **Step 6: Run E2E tests**

```bash
/opt/homebrew/bin/mvn -f /Users/mdproctor/claude/casehub/drafthouse/server/pom.xml install -DskipTests && /opt/homebrew/bin/mvn -f /Users/mdproctor/claude/casehub/drafthouse/server/pom.xml test -pl runtime 2>&1 | tail -30
```

Expected: All tests pass. The E2E tests exercise the panels in the browser — this validates the extraction didn't break anything.

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/drafthouse add -A
git -C /Users/mdproctor/claude/casehub/drafthouse commit -m "feat(#101): migrate to @casehubio/blocks-ui-document-workbench — delete local panels

Refs #101"
```
