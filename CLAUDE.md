# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm start          # dev server (http://localhost:8080)
npm run build      # production build
npm run lint       # eslint
npm run data       # serve local SVG data on port 8085 (requires http-server)
```

There are no tests.

To use a local graph SVG, pass `graph=http://127.0.0.1:8085/graph.svg` in the URL query string (see commented line in `appState.js`).

## Architecture

This is a **Vue 2** app that streams a large SVG file from a remote URL and renders it as an interactive WebGL map of subreddit communities.

### Data flow

1. `appState.js` determines which SVG URL to load (versioned remote or custom `graph=` query param). Camera position and the focused subreddit are persisted to the URL via the `query-state` library — there is no Vuex or other store.
2. `App.vue` mounts a canvas, instantiates `createStreamingSVGRenderer`, and wires up the event bus.
3. `createStreamingSVGRenderer` (`src/lib/createStreamingSVGRenderer.js`) is the core engine: it uses `createSVGLoader` + `streaming-svg-parser` to progressively parse the SVG as it downloads, converting each `<circle>` (node) and `<path>` (cluster boundary) into WebGL draw calls.
4. Rendering happens via the `w-gl` scene. WebGL geometry is managed by typed collection classes (`PointCollection`, `PolyLineCollection`, `MSDFTextCollection`, `ColoredTextCollection`, `LineCollection`, `ArrowCollection`) in `src/lib/`.

### Scene layer system

`createSceneLayerManager` (`src/lib/createSceneLayerManager.js`) manages z-ordering by inserting WebGL elements at explicit layer indices (`LayerLevels` in `constants.js`): Polygons → Edges → Nodes → Labels → Text. Named groups (e.g. `NamedGroups.MainGraph`) allow bulk hide/restore of related elements.

### Pointer / interaction handling

`createPointerEventsHandler` (`src/lib/createPointerEventsHandler.js`) builds an **RBush** spatial index of all nodes for fast nearest-neighbor click and hover lookup. It delegates scene transform events to update line widths proportionally to camera zoom.

### Special view modes

- **Subgraph / "Show Related"**: `createSubgraphVisualizer` extracts 1st+2nd-degree neighbors into a new `ngraph.graph`, runs a force layout, and overlays it on the scene. Pointer events are paused on the main graph while subgraph is active.
- **Street view**: `createStreetView` takes over pointer events and changes the camera perspective to walk along subreddit connections.

### Event bus

All cross-component communication goes through `src/lib/bus.js` (a thin `ngraph.events` wrapper). Key events: `show-subreddit`, `focus-node`, `unfocus`, `show-tooltip`, `progress`, `exit-subgraph`, `append-dom`, `remove-dom`.

### Styling

Components use **Stylus** (`.styl`). Shared design tokens (colors, panel widths, breakpoints) live in `src/vars.styl`.

### Build notes

- `vue.config.js` sets `publicPath: ''` (relative assets) and resolves `tinyqueue` explicitly due to a packaging quirk.
- `.worker.js` files are processed by `worker-loader`.
