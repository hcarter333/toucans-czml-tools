# CesiumLayerPanel

A tiny, dependency-free layer panel for **CesiumJS** that automatically lists your loaded **DataSources** (CZML / GeoJSON / KML / CustomDataSource) and lets users toggle their visibility with a single, delegated click handler. It stays in sync as sources are added, removed, or finish loading—no manual wiring per layer.

> **Tested with:** CesiumJS 1.120
> **Controls:** Only Cesium **DataSources**. (Imagery layers and 3D Tiles are out of scope for this class.)

---

## Features

* 🔌 **Drop-in**: Instantiate once; it automatically tracks viewer data sources as you add/remove DataSources.
* 🧭 **Auto-discovers** DataSources via `viewer.dataSources`.
* ♻️ **Auto-updates** on `dataSourceAdded`/`dataSourceRemoved`, and per-source `loadingEvent`/`changedEvent`.
* 🖱️ **Single, delegated** click handler (no per-row listeners).
* 🧰 **Filterable** list (e.g., show only “wedge/overlay” sources you tag).
* 🧼 No CSS/JS frameworks; minimal DOM generated on demand.

---

## Quick Start

### 1) Include Cesium

```html
<script src="https://cesium.com/downloads/cesiumjs/releases/1.120/Build/Cesium/Cesium.js"></script>
<link href="https://cesium.com/downloads/cesiumjs/releases/1.120/Build/Cesium/Widgets/widgets.css" rel="stylesheet">
```

### 2) Load the panel class (ES Module)

> **Important:** Module scripts must be served with a JavaScript MIME type. Use a host like jsDelivr or GitHub Pages; `raw.githubusercontent.com` returns `text/plain` and will be blocked by browsers.
>   
> **Perhaps even more Important:** Not looking to host a JavaScript module? Just copy the class [code]([url](https://github.com/hcarter333/toucans-czml-tools/blob/main/CesiumLayerPanel.js) and drop it in to the top of your JavaScript  


```html
<script type="module">
  import CesiumLayerPanel from 'https://cdn.jsdelivr.net/gh/<you>/<repo>@main/path/to/CesiumLayerPanel.js';

  const viewer = new Cesium.Viewer('cesiumContainer', {
    terrain: Cesium.Terrain.fromWorldTerrain()
  });

  // Create the panel (lists ALL DataSources by default)
  const panel = new CesiumLayerPanel(viewer, { title: 'Layers' });

  // Example: load a wedge CZML and add it
  const ds = await Cesium.CzmlDataSource.load('shell_wedge_MODELED_500m.czml');
  ds.name = 'Shell Wedge — 500 m';
  viewer.dataSources.add(ds);
</script>
```

### (Optional) Filter to only show wedges/overlays

Tag the sources you want to control, then filter:

```js
// Tag when you load them
const wedge = await Cesium.CzmlDataSource.load('interference_200yd_overlay.czml');
wedge.name = 'Interference — 200 yd';
wedge.__wedge = true; // ← your tag
viewer.dataSources.add(wedge);

// Only show tagged sources in the panel
const panel = new CesiumLayerPanel(viewer, {
  title: 'Wedges',
  filter: (src) => src.__wedge === true
});
```

---

## Installation Options

### ES Module (recommended)

Host `CesiumLayerPanel.js` on a server that serves `application/javascript` (e.g., jsDelivr or GitHub Pages), then import with `type="module"` as shown above.

At the end of your module file, ensure you **export** the class:

```js
export default CesiumLayerPanel;
```

### Non-module (UMD/IIFE)

If you prefer `<script src>`, provide a UMD build that assigns a global:

```html
<script src="https://cesium.com/.../Cesium.js"></script>
<script src="https://cdn.jsdelivr.net/gh/<you>/<repo>@main/path/to/CesiumLayerPanel.umd.js"></script>
<script>
  const viewer = new Cesium.Viewer('cesiumContainer', { terrain: Cesium.Terrain.fromWorldTerrain() });
  const panel = new CesiumLayerPanel(viewer, { title: 'Layers' });
</script>
```

(UMD build isn’t included here by default; it’s just the same class wrapped to set `window.CesiumLayerPanel`.)

---

## API

### `new CesiumLayerPanel(viewer, opts?)`

Create and attach a live control panel.

**Parameters**

* `viewer: Cesium.Viewer` — Your Cesium viewer (required).
* `opts?: object`

  * `panel?: string | HTMLElement` — A container element or CSS selector. If not found, the panel is created and appended to `document.body`. **Default:** `'#controlPanel'`.
  * `listSelector?: string` — Selector for the list container inside the panel. If missing, it’s created. **Default:** `'#dsList'`.
  * `filter?: (src) => boolean` — Return `true` to include a `DataSource` in the panel. **Default:** include all.
  * `title?: string` — Panel title. **Default:** `'Data Sources'`.

**Example**

```js
const panel = new CesiumLayerPanel(viewer, {
  panel: '#myPanel',          // or pass an HTMLElement
  listSelector: '#myList',
  title: 'Wedge Layers',
  filter: (src) => src.__wedge === true
});
```

---

### `panel.refresh(): void`

Forces a re-render of the UI. Usually unnecessary; the panel reacts to data source events automatically.

### `panel.getVisibleSources(): Cesium.DataSource[]`

Returns all **currently visible** `DataSource`s (after applying your `filter`, if any).

```js
const visible = panel.getVisibleSources();
console.log('Visible layer names:', visible.map(s => s.name));
```

### `panel.destroy(): void`

Detaches DOM and Cesium event listeners and clears internal state.

> Note: This does not remove the panel element from the DOM; it just stops it from functioning. You can remove the element manually if desired.

---

## How It Works

* Iterates `viewer.dataSources` using `collection.length` and `collection.get(i)` (Cesium’s `DataSourceCollection` does not provide `.values`).
* Renders one labeled checkbox per included `DataSource`. Toggling a checkbox sets `source.show`.
* Schedules a re-render on:

  * `viewer.dataSources.dataSourceAdded`
  * `viewer.dataSources.dataSourceRemoved`
  * `source.loadingEvent` (if present)
  * `source.changedEvent` (if present)
* Uses a **single** event listener on the panel (`click` delegation).

---

## Customizing the Panel Container

Provide your own container and the class will use it:

```html
<div id="controlPanel" style="position:absolute;top:10px;left:10px;z-index:1000">
  <strong>Data Sources</strong>
  <div id="dsList"></div>
</div>
```

```js
const panel = new CesiumLayerPanel(viewer, {
  panel: '#controlPanel',
  listSelector: '#dsList',
});
```

If the container or list is missing, they’ll be created with minimal default styles.

---

## Limitations & Notes

* **Controls DataSources only.** Imagery layers (`viewer.imageryLayers`) and 3D Tiles (`viewer.scene.primitives`) are not listed by this class. (You can extend the class similarly if you need to toggle those.)
* **Tagging for filters.** If you use `opts.filter`, remember to tag your target sources (e.g., `src.__wedge = true`) or write an appropriate predicate.
* **Names.** If `source.name` is set, it’s used as the label. Else, falls back to the constructor name (`'GeoJsonDataSource 1'`, etc.).
* **Rendering.** Call `viewer.scene.requestRender()` after toggling to update immediately in request-render scenes.

---

## Troubleshooting

**The panel is empty.**

* Ensure you add DataSources via `viewer.dataSources.add(ds)`.
* Confirm you’re not filtering them out (`opts.filter`).
* Verify you’re loading *DataSources* (CZML/GeoJSON/KML), not entities directly (`viewer.entities`) or imagery/tiles.

**I get “Expected a JavaScript module but server responded text/plain”.**

* Use a host that serves JavaScript with the correct MIME type (e.g., jsDelivr or GitHub Pages). Don’t use `raw.githubusercontent.com` for `<script type="module">`.

**A wedge didn’t show up.**

* Confirm it’s a CZML `DataSource` that you added to `viewer.dataSources`.
* If you’re filtering (e.g., by `__wedge`), ensure you set `ds.__wedge = true`.

---
