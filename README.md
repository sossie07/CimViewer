# CimViewer

A single-file web tool for inspecting IEC **CIM / CIM100 RDF/XML** power-network exports in the browser. Loads any CIM XML file, draws components on a real map *and* on a non-crossing schematic, and flags common data-quality problems (like AcLineSegments with the wrong number of Terminals).

No build step, no server, no install — just open the HTML file.

## Features

- **Two views**
  - *Geographic* — components placed on an OpenStreetMap background using their real `PositionPoint` coordinates.
  - *Relationship* — a schematic that lays every component out as a radial tree, with black dotted lines drawn along the CIM topology (`Equipment ─ Terminal ─ ConnectivityNode ─ Terminal ─ Equipment`). Nodes never overlap, so it's easy to see what's connected to what when components share a location on the map.
- **Click any component** to see every attribute from the source XML in a table — one click copies a value to the clipboard.
- **Dynamic layers** — every equipment type found in the file gets its own checkbox with a count.
- **Validation** checkboxes with an issues panel: highlights AcLineSegments with ≠2 terminals (pulsing red so they're impossible to miss), Terminals missing a ConnectivityNode, and Equipment missing a Location.
- **Search** by component name or mRID; pressing Enter zooms and selects in either view.
- **Resizable sidebar**, drag-to-pin nodes in the schematic, and zoom/pan on both views.

## Running it

1. Download [`CimViewer.html`](CimViewer.html).
2. Serve the folder with any local HTTP server. The simplest way is Python (included with most systems):
   ```
   python -m http.server 8000
   ```
3. Open <http://localhost:8000/CimViewer.html> in any modern browser (Chrome, Edge, Firefox).
4. Click **Load CIM XML file…** and pick your CIM RDF/XML export.

That's it. Nothing is uploaded anywhere — parsing happens entirely in your browser.

> **Why a local server?** OpenStreetMap's tile servers require a valid HTTP `Referer` header. If you open the HTML file directly by double-clicking it, the browser uses the `file://` protocol which sends no referer, so map tiles will show a 403 error. Serving from `localhost` fixes this. The Relationship (schematic) view works either way since it doesn't use map tiles.

### Other ways to serve locally

| Tool | Command |
|------|---------|
| Python 3 | `python -m http.server 8000` |
| Python 2 | `python -m SimpleHTTPServer 8000` |
| Node.js | `npx serve .` |
| PHP | `php -S localhost:8000` |
| VS Code | Install the "Live Server" extension, right-click the file → *Open with Live Server* |

## Supported CIM shape

- Namespaces: `http://iec.ch/TC57/CIM100#` (plus Zepben Evolve extensions).
- Any top-level CIM element with a `PowerSystemResource.Location` is recognised as equipment automatically — the type list is built from the file, so PowerTransformers, BusbarSections, custom types, etc. all appear without code changes.
- Equipment with ≥2 `PositionPoint`s is drawn as a polyline; everything else as a point marker.
- The schematic topology is reconstructed from `Terminal.ConductingEquipment` and `Terminal.ConnectivityNode` references.

## Dependencies

The page pulls these from public CDNs at load time:

- [Leaflet 1.9.4](https://leafletjs.com/) — map rendering
- [OpenStreetMap](https://www.openstreetmap.org/copyright) tiles
- [D3 v7](https://d3js.org/) — schematic layout (force + radial tree)

If you're behind a strict corporate proxy that blocks `unpkg.com`, `cdn.jsdelivr.net` or `tile.openstreetmap.org`, the relevant part of the page will fail silently. Easiest workaround: vendor the library files locally and update the `<script>`/`<link>` tags.

## Browser support

Any modern Chromium-based browser, Firefox, or Safari. The tool uses plain DOM + SVG + `fetch` + `DOMParser` — no polyfills required.

## Limitations / not yet supported

- CIM-level assets that aren't `PowerSystemResource`-located (e.g. catalog data) are skipped.
- The schematic layout builds a BFS spanning tree per connected component, so cycles in the topology are rendered without their extra edges (cycles are rare in distribution networks but common in transmission).
- Single-file tool — no persistence, no backend, no diff between files.

## Contributing

Issues and pull requests welcome. Keep the tool as a single HTML file with CDN dependencies unless there's a strong reason to add a build step.

## License

[MIT](LICENSE) — do whatever you like, no warranty.

## Support

If this tool saves you time, you can [buy me a coffee](https://buymeacoffee.com/sossie07) ☕. Entirely optional — the code stays free.
