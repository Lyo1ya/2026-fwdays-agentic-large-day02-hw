# AGENTS.md

## Purpose

This file guides coding agents working in this repository. It is sourced from the Memory Bank, especially:
- `docs/memory/techContext.md`
- `docs/memory/systemPatterns.md`

Prefer these docs when details here are missing.

## Project Structure

Excalidraw is a **monorepo** with a clear separation between the core library and the application:

- **`packages/excalidraw/`** - Main React component library published to npm as `@excalidraw/excalidraw`
- **`excalidraw-app/`** - Full-featured web application (excalidraw.com) that uses the library
- **`packages/`** - Core packages: `@excalidraw/common`, `@excalidraw/element`, `@excalidraw/math`, `@excalidraw/utils`
- **`examples/`** - Integration examples (NextJS, browser script)

## Architecture Rules (High Priority)

- Rendering is Canvas 2D based (not DOM drawing libraries).
- State updates should follow existing action architecture (`actionManager.dispatch()` patterns in `@excalidraw/excalidraw`).
- Preserve `AppState` as central app-state contract.
- Use existing element lifecycle helpers (`newElement`, `newElementWith`, `mutateElement`) and versioning semantics.
- Keep collaboration semantics intact:
  - E2E encryption (AES-GCM), key remains client-side (URL fragment).
  - Reconciliation/version conflict flow (`reconcileElements`) must stay backward compatible.
- Respect tuple-based geometry conventions in math (`[x, y]` point forms; branded types).

## Core Technical Context

- Node.js: >= 18
- TypeScript: 5.9.x (strict mode)
- Package manager: Yarn classic (`1.22.x`)
- App tooling: Vite
- Tests: Vitest + Testing Library + jsdom
- Lint/format: ESLint + Prettier

## Common Commands

- Dev: `yarn start`
- Build app: `yarn build`
- Build packages: `yarn build:packages`
- Test app: `yarn test:app`
- Full checks: `yarn test:all`
- Lint: `yarn test:code`
- Typecheck: `yarn test:typecheck`
- Auto-fix: `yarn fix`

## Collaboration and Persistence

- Collaboration stack uses WebSocket relay + Firebase persistence.
- Do not alter message/event semantics (`SCENE_INIT`, `SCENE_UPDATE`, presence events) without tests and compatibility review.
- Do not expose secrets, room keys, encryption keys, access tokens, or auth headers in logs/errors.

## Agent Workflow Expectations

1. Read relevant Memory Bank files before major edits.
2. Keep changes minimal and package-bound.
3. Add/update tests close to changed behavior.
4. Run targeted verification first, then broader checks when needed.
5. Prefer backward-compatible changes in serialization, restore, rendering, and collab flows.

## Testing Guidance

- Co-located tests (`*.test.ts` / `*.test.tsx`) are preferred.
- Use existing helpers in `packages/excalidraw/tests/helpers/` for interaction-heavy tests.
- Maintain or improve effective coverage on changed areas.

## References

- `docs/memory/techContext.md`
- `docs/memory/systemPatterns.md`
- `docs/memory/activeContext.md`
