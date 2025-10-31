# Paper Apps™ GOLF — Design Document

Goal
- Single-page Progressive Web App implementing Gladden Design Paper Apps GOLF ruleset (seeded courses) that runs entirely client-side from a static site (e.g., GitHub Pages). No external services, no login, no registration. Full offline functionality and export/import for sharing.

Assumptions / Defaults
- Rules follow Paper Apps GOLF ZERO PDF: d6 roll, fairway +1 boost, sand −1 penalty, water = +1 stroke and reset to previous position (parameterizable).
- Course structure: 3 courses × 18 holes (configurable). Holes are rectangular maps with grid tiles.
- Deterministic PRNG seeded by user-visible seed string for reproducible courses and die rolls.
- Single-device play modes: Solo, Pass-and-play, Shared-seed local comparison (via seed import/export). No server-side multiplayer.

High-level architecture (frontend-only)
- Tech stack: React + TypeScript (or Vue/Preact), Vite build, Tailwind CSS (optional).
- Rendering: Canvas (2D) for maps and animations; SVG export for print.
- State persistence: IndexedDB via localForage (or direct IndexedDB wrapper). All data stored locally.
- PWA: Web App Manifest + Service Worker (Workbox optional) for offline caching of app shell and assets.
- Single build artifact deployable to static hosting.

Major features
- Play modes:
  - Solo: full game saving/reloading.
  - Pass-and-play: multiple local players take turns on same device.
  - Seed mode: generate/import/export seed strings to reproduce courses and die rolls; players can exchange seed string manually.
- Course editor: create and save custom courses (tile painting, set tee/pin, par).
- Deterministic rules engine: seeded PRNG for die rolls and optional deterministic events.
- Shot interaction: roll d6, touch/drag shot within allowed radius, path preview, resolve collisions and terrain effects.
- Scorecard & history: hole-by-hole strokes, per-shot history, exportable PDF/SVG.
- Export/import:
  - Export course(s) and saved sessions as JSON files (shareable).
  - Export hole sheet + scorecard to printable PDF (client-side using jsPDF or SVG->PDF).
- Offline-first: app and data fully available offline after first load.
- Settings & rule variants: configurable fairway/sand/water behaviors, animation toggles, accessibility options.
- Seed sharing: human-friendly seed strings and QR code generation for easy transfer (QR generated client-side).

Data model (client-side JSON)
- CourseSeed
  - id: string (seed)
  - name: string
  - version: number
  - courses: [{name, holes: [Hole]}]
- Hole
  - width, height (pixels or tileCount)
  - tiles: flat array or RLE encoding of Terrain enums
  - tee: {x,y}
  - pin: {x,y}
  - par: number
- GameSession
  - id, createdAt, players: [{id,name,color}], mode, seedId
  - currentCourseIndex, currentHoleIndex, currentPlayerIndex
  - holeStates: [{strokes:[Stroke], ballPos:{x,y}, totalStrokes}]
  - settings: variant rules, snapToGrid, assistOptions
- UIState (transient): current roll, preview path, undo stack

Rules engine (client-side, deterministic)
- PRNG: small seeded PRNG (xorshift32 or mulberry32) seeded from seed string (e.g., CRC32 of seed text + session id).
- rollDie(prng): returns integer 1–6.
- computeShot(start, dirVector, rollValue, activeModifiers): clamp length to rollValue + boost; compute path segment and detect collisions with tile boundaries.
- collision rules:
  - Tree: stop at collision point.
  - Water: landing in water triggers water penalty: add stroke, reset ballPos per selected variant (default: previous position).
  - OB: treat as water or nearest in-bounds placement (configurable).
- Terrain modifiers: fairway sets boost counter (+1 default), sand sets penalty (−1 default) for next roll.
- Hole detection: snap to hole when within holeRadius.
- Deterministic outputs: same seed + same user actions produce same die rolls; if deterministic die sequence is desired for comparative seed play, provide setting "Use seed RNG" (default ON). Seed RNG ensures identical die rolls across devices using same seed.

UI / UX (client-only specifics)
- Main screens:
  - Home/Dashboard: Play (Solo / Pass & Play / Seed Play), Editor, Saved Games, Import/Export, Settings.
  - Game view:
    - Top bar: course/hole, par, strokes, undo, export hole sheet.
    - Canvas: map view with pinch-zoom, pan, terrain legend toggle.
    - Controls: Roll button (shows die animation using deterministic RNG), Roll history, Draw shot via drag gesture; numeric input alternative for accessibility.
    - Scorecard drawer: current hole strokes and full round totals.
  - Editor: tile palette, brush/erase/fill, set tee/pin, save as seed.
  - Import/Export modals: Load seed string, upload JSON, download session/course JSON, export PDF.
- Shot flow (client):
  1. Tap "Roll" → PRNG yields 1–6 → show reachable radius (roll + boost).
  2. Player drags from ball to target within radius; visual preview shows path and predicted landing tile + next-shot modifier preview.
  3. Confirm shot → engine computes final endpoint considering collisions; animate ball movement; record stroke.
  4. If water/OB occurs, show resolution animation and penalty; update ballPos per rules.
- Undo: single-step undo for last stroke; full-hole replay option.
- Export/share:
  - Seed string copy button and QR code (client-side QR generator).
  - JSON export/import for full session or individual courses.
  - Printable PDF/SVG generation executed in browser; prompt for download or print.

Storage & export
- Local persistence:
  - IndexedDB stores: savedCourses, savedSessions, settings. Autosave after each stroke.
  - Storage schema versioning to support migrations.
- Manual export/import only:
  - Export JSON for sessions/courses to share via email/files.
  - Import JSON restores courses/sessions locally.
- No remote sync: data remains on device unless user manually transfers files; complies with frontend-only constraint.

PWA specifics (static-host friendly)
- Web App Manifest:
  - name, short_name, icons (multiple sizes), start_url relative to root, display: standalone, scope: /, theme_color, background_color.
- Service Worker:
  - Precache app shell (index.html, CSS, JS bundles, fonts, icons) at install.
  - Runtime cache for dynamically loaded assets (course images if any).
  - Network-first NOT used for game data; rely on IndexedDB for dynamic content.
  - Update strategy: notify user when new version available; "Install update" button that reloads app without losing current session (persist transient state to IndexedDB before reload).
- Offline first:
  - All gameplay features available offline after initial load including editor, seed generation, die rolls, exports.

Security & privacy (client-only)
- No external network calls required; all operations performed locally.
- If QR codes or files include seed strings, they contain only seed/course data and user-chosen player names; warn users about sharing names if privacy desired.
- All optional analytics disabled by default (not implemented for frontend-only build).
- Sanitize imported JSON to prevent potential misuse of stored UI strings; validate schema before importing.

Determinism & seed format
- Seed format: short human string + optional compressed payload.
  - Example seed string: "GPA-0001-5F2A3B" or base32 encoded blob representing PRNG seed and course checksum.
- PRNG algorithm: mulberry32(seedHash) or xorshift32 with 32-bit seed derived from a stable hash (FNV-1a or CRC32) of seed string.
- Seed uses:
  - Course generation (if procedural) OR exact course payload embedded in seed JSON.
  - Die roll sequence when "Use seed RNG" setting ON — ensures identical rolls across devices for shared-seed competitions.

Implementation details & components
- App shell (React + TypeScript)
  - App.tsx: routing and global contexts (SettingsContext, SessionContext, Persistence).
  - Canvas component: HoleCanvas.tsx — draw map, ball, preview path; handle touch/mouse events.
  - Rules engine module: rulesEngine.ts — PRNG, rollDie, computeShot, resolveLanding, applyModifiers.
  - Persistence module: storage.ts — IndexedDB wrappers for save/load/export/import.
  - Editor components: EditorCanvas, Palette, Toolbars.
  - Export utilities: svgExport.ts, pdfExport.ts (client-side generation).
  - Service worker registration: sw.ts and manifest.json.
- Assets:
  - Minimal icon set, font(s), terrain color palette.
  - No external fonts by default to avoid network dependency.

Testing
- Unit tests: rules engine deterministic tests, seed parsing, collision detection.
- Integration tests: simulated full-hole plays for canonical seeds; export/import roundtrip.
- Manual QA: offline install, PWA installability on Android/iOS, printing/export checks.
- Cross-browser: Chrome, Firefox, Safari (iOS PWA constraints), Edge.

Performance considerations
- Canvas rendering optimized with dirty-rectangle updates; avoid full redraw each frame when animating small elements.
- Use RLE or sparse tile storage for large custom maps to keep IndexedDB size low.
- Lazy-load editor assets only when entering editor.

Developer & build notes
- Build: Vite for fast dev and static output; configure base path for GitHub Pages (repository name).
- Single static output: index.html + assets; deploy to GitHub Pages or any static host.
- Environment flags: NODE_ENV for production; feature flags for debugging.
- Example repo structure:
  - /src
    - /components
    - /engine
    - /storage
    - /assets
    - main.tsx, App.tsx
  - /public
    - manifest.json, icons, sw.js
- CI: GitHub Actions to build and publish to gh-pages branch.

MVP checklist (frontend-only)
- [ ] Deterministic PRNG and rollDie
- [ ] Rules engine: shot validation, collision handling, terrain modifiers
- [ ] Canvas renderer for hole maps and animations
- [ ] Roll UI, drag-to-aim with preview, numeric shot input alternative
- [ ] IndexedDB persistence (save/load sessions & courses)
- [ ] Seed import/export (string + JSON)
- [ ] Course editor (basic tile painting, set tee/pin, save as seed)
- [ ] Export hole sheet & scorecard to PDF/SVG client-side
- [ ] PWA manifest + Service Worker for offline
- [ ] Undo/replay, scorecard view, pass-and-play flow
- [ ] Accessibility: keyboard/numeric shot input, ARIA labels
- [ ] Tests for rules engine and export/import

Appendix: Example JSON export (session)
{
  "version":1,
  "seedId":"GPA-0001-5F2A3B",
  "createdAt":"2025-10-31T00:00:00Z",
  "players":[{"id":"p1","name":"Alice","color":"#e53e3e"}],
  "currentCourseIndex":0,
  "currentHoleIndex":2,
  "holeStates":[
    {"holeIndex":0,"strokes":[{"from":{"x":5,"y":10},"to":{"x":8,"y":12},"roll":4,"landTerrain":"Fairway"}], "ballPos":{"x":8,"y":12},"totalStrokes":4}
  ],
  "settings":{"fairwayBoost":1,"sandPenalty":-1,"waterPenalty":1,"useSeedRng":true}
}

End.
