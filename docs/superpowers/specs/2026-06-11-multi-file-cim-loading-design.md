# Multi-file CIM loading with cross-file connectivity — Design

Date: 2026-06-11
Status: Approved

## Problem

CimViewer loads a single CIM XML file at a time into a global `model`. Two gaps:

1. **Same-name reload bug.** Re-picking a file with the same filename (e.g. after
   fixing a data issue) does not refresh the view — the stale, pre-fix network is
   still shown. Root cause: `<input type="file">` only fires `change` when its
   `value` changes; re-selecting the same path leaves `value` identical, so the
   handler never runs. It is **not** a parse/caching problem.

2. **No multi-file support.** Real networks are exported by Similix as separate
   files that connect to each other. The viewer cannot show the combined network.

## Key data fact (confirmed)

Separate Similix exports connect at shared `ConnectivityNode` rdf:IDs. The CN id
is deterministic (`{ownerGID}.{seq}.CN`), so when two exports meet at the same
node they reference the *same* CN id. Verified on `\\OREDNAR01T\CIMexports`: e.g.
`194c1ed8-…f1.1.CN` appears in both `0001C1470104.xml` and `2752C1580058.xml`.

Therefore merging is a **union keyed by rdf:ID**: shared CN ids stitch the
networks together automatically. Coordinates are lat/lon in a common frame, so no
reprojection is required.

## Design

### 1. Same-name reload fix

In the file-input `change` handler, after reading the selected file(s), set
`e.target.value = ''`. This lets `change` fire again on a same-name re-pick.
This fix stands on its own, independent of the multi-file work.

### 2. Data-model refactor

Replace the single global `model = parseCim(text)` with:

- `loadedFiles`: `Array<{ name, parsed }>` — one entry per loaded file, each
  holding that file's raw `parseCim` result. Per-file results are retained so
  per-file validation stays exact.
- `model`: a **derived merged model**, rebuilt by `rebuildModel()` from
  `loadedFiles` whenever the set changes.

`rebuildModel()` produces the same shape `parseCim` returns today
(`equipment`, `terminals`, `cns`, `locations`, `termsByEq`, `termsByCN`,
`allTop`, plus `cimNamespace`/`cimVersion` from the first file) so the existing
`render`, `buildGraph`, validators, and selection logic keep working unchanged.

Merge rules:

- For `equipment`, `terminals`, `cns`, `locations`: union the per-file maps keyed
  by rdf:ID. **Same id across files = same object** (the mechanism that makes
  cross-file connectivity work). On collision, keep the first occurrence's object
  but accumulate provenance.
- Each merged object gains `sources: Set<filename>` recording which file(s) it
  came from. (For locations, points come from whichever file first defined them;
  boundary CNs are the shared objects, not locations.)
- `termsByEq` / `termsByCN` indexes are rebuilt over the unioned terminals.
- `allTop` for the merged model is the concatenation of every file's `allTop`,
  each tagged with its source filename, so cross-file checks can run over it.

### 3. Loading UX — additive + multi-select

- File input gets the `multiple` attribute.
- The `change` handler loops over `e.target.files`, parses each, appends a
  `{ name, parsed }` entry to `loadedFiles`, then calls `rebuildModel()` +
  re-render. Re-loading a filename already present **replaces** that file's entry
  (so the same-name fix also refreshes a file inside a multi-file set).
- New **"Loaded files"** sidebar section: one row per file showing
  `name — N equipment`, with a `×` button that removes that file's entry and
  rebuilds. A **"Clear all"** button empties `loadedFiles` and resets the view.
- Status line summarizes merged totals (equipment / terminals / CNs / files).

### 4. Provenance & validation

- Selected-component box shows the source filename(s) of the selected object.
- Validation/issue rows show source filename(s).
- **Per-file validation preserved.** Duplicate-rdf:ID and abstract-class checks
  run over each file's own `parsed.allTop` and are reported grouped/labelled by
  file. The known Similix within-file `PowerTransformer == PowerTransformerEnd`
  duplicate is still caught and is not confused by the merge.
- **New cross-file check.** Report rdf:IDs that appear in more than one file
  **only when they identify a different object** (the type or
  `IdentifiedObject.name` disagrees across files). Similix deliberately
  re-exports the *same* real-world object into every file that touches it —
  shared context (regions, base voltages, coordinate systems, substations) and
  boundary equipment (the boundary device plus its terminals/location/points and
  the boundary ConnectivityNodes) — with the same id, type, and name so the
  networks reconnect on merge. Those same-object repeats are expected and not
  flagged; only a genuine clash (one id, two different things) is reported.
  Additionally, shared reference/context classes (`BaseVoltage`,
  `GeographicalRegion`, `SubGeographicalRegion`, `Substation`, `VoltageLevel`,
  `Bay`, `CoordinateSystem`) are never flagged — they are catalog/hierarchy data
  that legitimately recurs in every file and Similix may label inconsistently
  (e.g. `BaseVoltage 400` as "0.4kV"/"400"/"400V").

## Out of scope (YAGNI)

- Color-by-file styling (keep type-based colors).
- Coordinate/CRS reprojection (all files share a lat/lon frame).
- Auto-loading an entire folder / drag-and-drop of a directory.

## Acceptance

- Re-picking a same-named file refreshes the view (stale network clears).
- Selecting multiple files, or loading files one after another, shows a single
  merged network with boundaries stitched at shared CN ids.
- Removing one file (× ) or "Clear all" updates the merged network correctly.
- Per-file duplicate-rdf:ID and abstract-class issues remain correct and are
  labelled by file; a new cross-file non-CN id-collision check is available.
- Selection, search-zoom, toggle-worlds, and both views work over the merged
  model exactly as before.
