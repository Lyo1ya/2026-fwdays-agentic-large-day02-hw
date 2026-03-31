# Progress — Excalidraw

> Cross-ref: [domain-glossary.md](../product/domain-glossary.md) · [architecture.md](../technical/architecture.md) · [code-vs-docs.md](../technical/code-vs-docs.md) · [dev-setup.md](../technical/dev-setup.md)

> Feature completion, package maturity, test coverage, and documentation status.
> Last updated: 2026-03-29

---

## 1. Package Versions & Maturity

| Package | Version | Status | Notes |
|---------|---------|--------|-------|
| `@excalidraw/common` | 0.18.0 | Stable | Constants, emitter, utils — rarely changes |
| `@excalidraw/math` | 0.18.0 | Stable | 2D geometry primitives, branded tuple types |
| `@excalidraw/element` | 0.18.0 | Stable | Element model, scene, store, bindings |
| `@excalidraw/excalidraw` | 0.18.0 | Stable | Core React component + full action system |
| `@excalidraw/utils` | 0.1.2 | Early | Export & conversion utilities, independent version |
| `excalidraw-app` | 1.0.0 | Stable | Standalone web app with collab |

---

## 2. Feature Completion

### 2.1 Drawing & Elements — ✅ Complete

| Feature | Status | Source |
|---------|--------|--------|
| Rectangle, Diamond, Ellipse | ✅ | `shapes.tsx` — SHAPES array, `newElement.ts` |
| Arrow (straight, curved, elbow) | ✅ | `elbowArrow.ts`, `arrows/` directory |
| Line (multi-point) | ✅ | `linearElementEditor.ts` |
| Freedraw | ✅ | `perfect-freehand` integration |
| Text (auto-resize, wrapping) | ✅ | `textElement.ts`, `textWrapping.ts` |
| Image (embed, compress, crop) | ✅ | `image.ts`, `cropElement.ts` |
| Frame & MagicFrame | ✅ | `frame.ts`, `actionFrame.ts` |
| Embeddable (iframe, video) | ✅ | `embeddable.ts`, `actionEmbeddable.ts` |
| Eraser | ✅ | `eraser/` directory |
| Laser pointer | ✅ | `laser-trails.ts` |

### 2.2 Selection & Manipulation — ✅ Complete

| Feature | Status | Source |
|---------|--------|--------|
| Box selection | ✅ | `App.tsx` pointer handlers |
| Lasso selection | ✅ | `lasso/` directory |
| Grouping / ungrouping | ✅ | `groups.ts`, `actionGroup.tsx` |
| Alignment (6 directions) | ✅ | `align.ts`, `actionAlign.tsx` |
| Distribution (H/V) | ✅ | `distribute.ts`, `actionDistribute.tsx` |
| Flip (H/V) | ✅ | `actionFlip.ts` |
| Z-ordering (4 directions) | ✅ | `zindex.ts`, `actionZindex.tsx` |
| Duplicate | ✅ | `duplicate.ts`, `actionDuplicateSelection.tsx` |
| Element locking | ✅ | `actionElementLock.ts` |
| Shape conversion (rect↔diamond↔ellipse) | ✅ | `ConvertElementTypePopup.tsx` |
| Copy/paste styles | ✅ | `actionStyles.ts` |
| Resize & transform | ✅ | `resizeElements.ts`, `transformHandles.ts` |

### 2.3 Data & Export — ✅ Complete

| Feature | Status | Source |
|---------|--------|--------|
| Export PNG | ✅ | `ImageExportDialog.tsx`, `data/index.ts` |
| Export SVG | ✅ | `staticSvgScene.ts`, font subsetting |
| Export JSON (.excalidraw) | ✅ | `JSONExportDialog.tsx` |
| Copy PNG to clipboard | ✅ | `actionClipboard.tsx` |
| Embed scene in PNG/SVG | ✅ | `exportEmbedScene` option |
| Import from JSON/blob | ✅ | `data/blob.ts`, `restore.ts` |
| Paste chart (CSV/TSV) | ✅ | `PasteChartDialog.tsx`, `charts/` |

### 2.4 Collaboration — ✅ Complete

| Feature | Status | Source |
|---------|--------|--------|
| WebSocket rooms (Socket.IO) | ✅ | `Portal.tsx` |
| E2E encryption (AES-GCM) | ✅ | `Portal._broadcastSocketData()` |
| Element reconciliation | ✅ | `reconcile.ts` — `reconcileElements()` |
| Remote cursors | ✅ | `InteractiveCanvas.tsx` |
| Follow mode | ✅ | `FollowMode/`, `actionNavigate.tsx` |
| User presence (idle, in-call) | ✅ | `UserList.tsx`, `IDLE_STATUS` protocol |
| Firebase persistence | ✅ | `excalidraw-app/data/firebase.ts` |
| Tab sync (BroadcastChannel) | ✅ | `excalidraw-app/data/tabSync.ts` |
| Multiplayer undo/redo | ✅ | `Store` + `History` delta system |

### 2.5 UI & UX — ✅ Complete

| Feature | Status | Source |
|---------|--------|--------|
| Command palette | ✅ | `CommandPalette/` — fuzzy search, categories |
| Scene search (Ctrl+F) | ✅ | `SearchMenu.tsx` |
| Keyboard shortcuts (60+ actions) | ✅ | `shortcuts.ts` — `ActionName` union |
| Context menu | ✅ | `ContextMenu.tsx` |
| Welcome screen | ✅ | `welcome-screen/` — customizable hints |
| Help dialog | ✅ | `HelpDialog.tsx` |
| Color picker + eye dropper | ✅ | `ColorPicker/`, `EyeDropper.tsx` |
| Font picker | ✅ | `FontPicker/` |
| Stats panel (editable) | ✅ | `Stats/` |
| Sidebar (library, custom tabs) | ✅ | `Sidebar/`, `DefaultSidebar.tsx` |
| Light/dark theme | ✅ | `DarkModeToggle.tsx`, `THEME` constant |
| Mobile menu | ✅ | `MobileMenu.tsx`, `MobileToolBar.tsx` |
| Zen mode | ✅ | `actionToggleZenMode.tsx` |
| View mode (read-only) | ✅ | `actionToggleViewMode.tsx` |
| Grid + snap mode | ✅ | `actionToggleGridMode.tsx`, `snapping.ts` |
| PWA (offline support) | ✅ | `vite-plugin-pwa`, `service-worker.js` |
| i18n (59 locale files) | ✅ | `packages/excalidraw/locales/` |

### 2.6 AI & Conversion — ✅ Complete

| Feature | Status | Source |
|---------|--------|--------|
| Mermaid → Excalidraw | ✅ | `mermaid.ts`, `TTDDialog/MermaidToExcalidraw.tsx` |
| Text to Diagram (AI) | ✅ | `TTDDialog/TextToDiagram.tsx` — requires host handler |
| Diagram to Code plugin | ✅ | `DiagramToCodePlugin/` |

---

## 3. Test Coverage

### 3.1 Test File Distribution

| Package | Test files | Key areas covered |
|---------|-----------|-------------------|
| `packages/excalidraw` | 60 | Actions, clipboard, history, selection, rendering, wysiwyg |
| `packages/element` | 23 | Binding, resize, elbow arrows, frames, z-index, text, collision |
| `packages/math` | 7 | Point, vector, segment, line, ellipse, curve, range |
| `packages/common` | 6 | Colors, utils, keys, URL, queue, appEventBus |
| `packages/utils` | 4 | Export, geometry, withinBounds |
| `excalidraw-app` | 3 | Collab, mobile menu, language list |
| **Total** | **103** | |

### 3.2 Coverage Thresholds (vitest.config.mts)

| Metric | Threshold |
|--------|-----------|
| Lines | 60% |
| Branches | 70% |
| Functions | 63% |
| Statements | 60% |

### 3.3 Known Skipped/Flaky Tests

- 6 skipped flip tests — curves outside minMax points produce wrong bounding boxes
- Flaky labeled arrow version bump test (`textWysiwyg.test.tsx:335`)
- Frame `getElementsCompletelyInFrame()` fails in test env but works in browser
- History `groupIds` order not guaranteed after undo/redo

---

## 4. Documentation Status

### 4.1 Memory Bank (`docs/memory/`)

| File | Lines | Status |
|------|-------|--------|
| `projectbrief.md` | 84 | ✅ Complete |
| `productContext.md` | 188 | ✅ Complete |
| `systemPatterns.md` | 191 | ✅ Complete |
| `techContext.md` | 161 | ✅ Complete |
| `decisionLog.md` | 339 | ✅ Complete |
| `activeContext.md` | 116 | ✅ Complete |
| `progress.md` | — | ✅ Current file |

### 4.2 Technical Docs (`docs/technical/`)

| File | Status | Content |
|------|--------|---------|
| `architecture.md` | ✅ | 535 lines — layered arch, data flow, rendering |
| `dev-setup.md` | ✅ | 346 lines — prerequisites, scripts, debugging |
| `code-vs-docs.md` | ✅ | 184 lines — undocumented behavior, tech debt |

### 4.3 Product Docs (`docs/product/`)

| File | Status | Content |
|------|--------|---------|
| `domain-glossary.md` | ✅ | 363 lines — all domain types and concepts |

---

## 5. What's Not Done / Open Items

| Area | Status | Details |
|------|--------|---------|
| `@excalidraw/utils` versioning | ⚠️ | Still at 0.1.2 while others are 0.18.0 |
| Point tuple migration | ⚠️ | `GlobalCoord`/`LocalCoord` still exist alongside branded `Point` |
| Interactive canvas type split | ⚠️ | 3 fields leak from interactive to static canvas types |
| `isSomeElementSelected` refactor | ⚠️ | Module-level closure cache, should move to editor instance |
| Circular dependency in common | ⚠️ | `pick` duplicated in `colors.ts` |
| WASM font loading | ⚠️ | No URL-based fetching support, must embed at build time |
| `excalidraw-app` test coverage | ⚠️ | Only 3 test files for the entire app layer |

---

## 6. Build & Deployment

| Target | Status | Notes |
|--------|--------|-------|
| Vercel | ✅ Ready | `vercel.json` configured |
| Docker | ✅ Ready | Multi-stage: Node 18 → Nginx 1.27-alpine |
| npm publish | ✅ Ready | `scripts/release.js` with test/next/latest tags |
| ESM bundles | ✅ | UMD deprecated in 0.18.0 |
| PWA | ✅ | Service worker, manifest, offline fallback |

