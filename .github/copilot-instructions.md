# FlightLog Pro — Project Guidelines

## Project Summary

**FlightLog Pro** is a browser-based Electronic Flight Folder (EFB) for pilots.  
It opens `.effarchive` / ZIP exports from an EFB mobile app, parses the embedded backup files,
and provides a read/interactive representation of the flight plan.

Full project overview: see [readme.md](../readme.md).

---

## Architecture

### Single-File Application — the golden rule

> All code, styles, and templates live in **`index.html`**. There are no other source files.  
> Do **not** create new `.js`, `.ts`, `.css`, or `.html` files. Every change goes into `index.html`.

- No build system, no npm, no TypeScript, no bundler.
- Libraries are loaded from CDN `<script>` tags: React 18, ReactDOM, Tailwind CSS (browser build), JSZip, PDF.js.
- The app runs by opening `index.html` directly in a browser (file:// or local server).

### React without JSX

All UI is written with `React.createElement` aliased as `h`:

```js
// ✅ Correct
h('div', { className: "..." }, h(MyComponent, { prop: value }))

// ❌ Never use JSX
<div className="..."><MyComponent prop={value} /></div>
```

### Component structure

| Component | Purpose |
|-----------|---------|
| `Icon` | SVG icons from inline `iconPaths` map |
| `WelcomeScreen` | Upload screen, shown before any archive is loaded |
| `RouteHeader` | Flight identification header (callsign, times, flight time) |
| `WaypointCard` | One card per waypoint: Planned / Actual / Diff rows with editable inputs |
| `WaypointDetailModal` | Waypoint popup: full details + editable Planned/Actual/Diff controls synced with the card |
| `FlightSummaryPage` | Summary tab: aircraft info, airports, crew, OptiClimb, ATC text, remarks |
| `PdfViewer` | Renders multi-page PDFs via PDF.js onto canvas elements |
| `DocumentViewer` | Routes to correct sub-viewer based on file extension |
| `DocumentsPage` | Document list + active viewer |
| `App` | Root: manages all state, file loading lifecycle, tab routing |

### Tabs (`selectedPage` state)

| Value | Label | Content |
|-------|-------|---------|
| `'summary'` | Flight Summary | `FlightSummaryPage` (flight.backup data) |
| `'OFP'` | OFP | `WaypointCard` list; fuel check already implemented |
| `'documents'` | Documents | `DocumentsPage` |

---

## Data & Parsing

### Archive structure

| File | Required | Parser function |
|------|----------|-----------------|
| `routes.backup` | ✅ | `processFlightData()` → sorts waypoints by `sequenceId` |
| `flight.backup` | optional | `processFlightSummaryData()` |
| `crewMembers.backup` | optional | `JSON.parse()` → expects array |
| `documents.backup` | optional | `processDocumentMetadata()` → `[{filename, extension}]` |
| Document blobs | optional | Extracted by filename from archive, typed via `getDocumentMimeType()` |

### Waypoint fields

Base fields: `title`, `passingByDate` (ISO), `fuel.{remaining,cumulated,unit}`.  
19 optional columns defined in `STATIC_COLUMNS` (paths like `averages.altitude`, `tracks.segment.true`).  
Special paths: `coordinates` (pseudo-field combining lat/lon), `tracks.*.*` (extracted via `extractTrackValue()`).

### Key utility functions

| Function | Purpose |
|----------|---------|
| `getNestedValue(obj, path)` | Dot-notation access + special-case delegates |
| `extractTrackValue(wp, path)` | Finds `{value,unit,type}` in `tracks[segment][array]` |
| `formatWaypointValue(value)` | Handles objects, arrays, numbers, strings |
| `formatNumber(value)` | `Intl.NumberFormat` with 9 decimal places |
| `formatUnit(unit, value)` | Normalizes "deg" → "°", etc. |
| `sanitizeRichText(html)` | DOMParser-based whitelist sanitizer (b, strong, br, ul, ol, li, p) |
| `parseDurationToMinutes(iso)` | Converts `PT1H30M` → 90 |

---

## UI Conventions

### Styling

- Use **Tailwind utility classes** directly in `className`. No custom CSS.
- Spacing scale: `p-3 sm:p-6`, `rounded-3xl`, `shadow-sm` — mobile-first, Apple-inspired.
- Follow the existing `UI_STYLES` constants for repeated patterns:

```js
UI_STYLES.header   // section headers
UI_STYLES.label    // row labels
UI_STYLES.value    // plain values
UI_STYLES.rowHeight // flex row wrapper
UI_STYLES.input    // editable inputs
UI_STYLES.button   // ▲/▼ stepper buttons
```

### New UI sections

Wrap new cards in: `h('div', { className: "bg-white p-4 sm:p-8 rounded-2xl sm:rounded-[2rem] shadow-sm" }, ...)`

Wrap new row grids in: `h('div', { className: "contents" }, ...)` inside a CSS grid container.

### Diff coloring convention

- ` getDifferenceColor(diff)` returns `text-red-500` for positive, `text-emerald-500` for negative, `text-gray-400` for zero.
- For fuel remaining, negate the sign: `getDifferenceColor(-fuelRemainingDifference)` (more remaining is good).

### Waypoint popup behavior

- Clicking a waypoint title opens `WaypointDetailModal`.
- The popup includes Time / Used / Remaining Planned, Actual, and Diff fields with the same input and stepper logic as `WaypointCard`.
- `actualData` state is owned by `WaypointCard` and passed into `WaypointDetailModal`, so edits are synchronized in both views.
- Popup close actions: backdrop click, close button, or Escape key.

---

## Domain Notes (Aviation)

- **Fuel values** in the archive use the `unit` from `fuel.unit` (typically `"KG"` or `"LBS"`). Always display the unit.
- **FUEL_STEP = 100** — increment/decrement step for fuel inputs.
- **Times** are always UTC; display with trailing `Z` (e.g., `14:30Z`).
- **Tracks** in `routes.backup` are stored as arrays: `tracks.segment = [{value, unit, type: "true"}, {value, unit, type: "magnetic"}]`.
- **sequenceId** determines waypoint order, not array position.
- **OFP** = Operational Flight Plan — the waypoint list page.
- **ETOPS** = Extended-range Twin-engine Operational Performance Standards (long over-water flights).

---

## What's Done / What Can Be Extended

| Feature | Status |
|---------|--------|
| Archive import (effarchive/ZIP) | ✅ Done |
| Waypoint OFP with fuel check | ✅ Done |
| 19 configurable extra columns | ✅ Done |
| Flight Summary (flight.backup) | ✅ Done |
| Documents tab (PDF, images, text) | ✅ Done |
| Crew roster display | ✅ Done |
| OptiClimb / ATC text | ✅ Done |

Logical areas for future extension:
- Additional backup files from the archive (e.g., `notams.backup`, `hazards.backup`, `delays.backup`, `info.backup`)
- New tabs for NOTAMs, hazards, or delay info
- Print / export view
- Persist actual flight data (localStorage)
