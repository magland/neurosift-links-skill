# Neurosift URL patterns — full reference

Base host: `https://neurosift.app`. Everything here is about **DANDI + NWB**.

This document is the exhaustive version of the patterns summarized in
[../SKILL.md](../SKILL.md). Read it when you need a plugin name, the exact `tab`
grammar, or an edge case.

---

## 1. Routes

| Route | What it shows | Parameters |
|-------|---------------|------------|
| `/` | Landing page | none |
| `/dandi` | DANDI browser / search | none that can be preset (search state is in-app only) |
| `/dandiset/<dandisetId>` | One dandiset (metadata + file browser) | `dandisetVersion` |
| `/nwb` | One NWB file | `url` **or** (`dandisetId` + `path`); plus `dandisetId`, `dandisetVersion`, `tab`, `embedded`, `lindi` |

`<dandisetId>` is the 6-digit DANDI id, e.g. `001637`.

### Version

- `dandisetVersion` is either `draft` or a published version like `0.240101.1234`.
- If omitted, the NWB viewer treats it as `draft`. Include it when you can.

---

## 2. The `/nwb` file view

### 2a. Which file — two forms

**Path-based** (preferred when you know the asset path within the dandiset).
Neurosift queries the DANDI API (`assets/?glob=<path>`) and resolves the download
URL itself:

```
https://neurosift.app/nwb?dandisetId=<id>&dandisetVersion=<version>&path=<assetPath>
```

- `<assetPath>` is the file's path inside the dandiset, e.g.
  `sub-820454/sub-820454_ses-ecephys-820454-2025-11-04-14-59-22_ecephys.nwb`.
  Write it literally (a `/` in the path is fine).
- The `path` must **exactly match** an asset in that dandiset/version — Neurosift
  resolves it via `GET /dandisets/<id>/versions/<version>/assets/?glob=<path>`.
  A wrong or nonexistent `path` yields no match, so no `url` is ever set and the
  viewer shows **"No NWB file URL provided."** If you see that message with a
  path-based link, the `path` (or `dandisetId`/`dandisetVersion`) is wrong —
  confirm it against the DANDI API rather than guessing file names.

**URL-based** (when you already have the asset download URL):

```
https://neurosift.app/nwb?url=<downloadUrl>&dandisetId=<id>&dandisetVersion=<version>
```

- `<downloadUrl>` for a DANDI asset is
  `https://api.dandiarchive.org/api/assets/<asset_id>/download/`.
- **Do not URL-encode `<downloadUrl>`.** Insert it as a plain URL. Neurosift
  decodes/normalizes it internally (it even repairs `+`→space damage), so the
  readable form is what you want.
- `dandisetId` / `dandisetVersion` are optional here but drive the left sidebar
  (dandiset metadata) and the version switcher, so include them for DANDI files.
- The `url=` form works for **any** remote NWB or LINDI file. For a non-DANDI
  file, just give `url=` and omit `dandisetId`/`dandisetVersion`.

If both `url` and `path` are somehow present, `url` wins.

### 2b. Optional parameters

| Param | Effect |
|-------|--------|
| `tab` | Open the file focused on a specific object/view (see §3). |
| `embedded=1` | Hide the top app bar (for iframe embedding). |
| `lindi=0` | Do not attempt to use a LINDI version of the file. |

---

## 3. The `tab` parameter

`tab` selects what the main panel of the NWB viewer shows. Five shapes:

### 3a. A neurodata object path (the common case)

```
&tab=/processing/eye_tracking
```

Opens that object and shows **every** view whose plugin can handle it, stacked
vertically in the registry order of §4 (so the first applicable plugin appears at
the top). **Use this unless you want to pin the file to one specific view.**
Paths are written literally, no encoding.

### 3b. A fixed (whole-file) tab

`tab` may name one of the built-in views:

| `tab` value | View |
|-------------|------|
| `widgets` | Default hierarchy/overview (this is the default; omit `tab` to get it) |
| `hdf5` | Raw HDF5 tree |
| `specifications` | The file's embedded NWB schema |
| `python-usage` | Auto-generated Python usage script |
| `timeseries-alignment` | Timeseries alignment overview |
| `video-widget` | Videos (only present if the file has external videos) |
| `icephys` | Intracellular ephys (only present if the file has icephys tables) |

`video-widget` and `icephys` fall back to `widgets` if the file lacks that data.

### 3c. A specific plugin view of one object

```
&tab=view:<PluginName>|<path>
```

Forces the `<PluginName>` visualization for `<path>`. Example — spike raster of
the units table:

```
&tab=view:Raster|/units
```

Use this only to override the auto-selected view. `<PluginName>` must be one of
the names in §4 and must actually be able to handle that object's type.

### 3d. A plugin view that needs secondary object(s)

Some views combine a primary object with one or more secondary objects. Append
secondary paths to the primary path with the `^` separator:

```
&tab=view:<PluginName>|<primaryPath>^<secondaryPath1>^<secondaryPath2>
```

The two views that require a secondary path:

| Plugin | Primary object | Secondary object |
|--------|----------------|------------------|
| `PSTH` | a `TimeIntervals` table (e.g. trials) | a `Units` table |
| `TrialAlignedSeries` | a `TimeIntervals` table | a `RoiResponseSeries`, `FiberPhotometryResponseSeries`, or `MicroscopyResponseSeries` |

Example — PSTH of `/units` aligned to `/intervals/trials`:

```
&tab=view:PSTH|/intervals/trials^/units
```

(The primary object is the trials table; the units table is the secondary.)

### 3e. Multiple objects in one tab

A JSON array of entries opens a combined multi-object view:

```
&tab=["/acquisition/ElectricalSeries","/units"]
```

Each array entry is either:

- a bare path — `"/units"` — auto-selected view, or
- `PluginName|path` — e.g. `"Raster|/units"`.

**Important prefix difference:** inside the array, a plugin is written
`PluginName|path` with **no** `view:` prefix. The `view:` prefix is used **only**
for the single-object form (§3c/§3d). Secondary paths inside array entries
(`PluginName|path^secondary`) are parsed but not fully wired up for every view —
for combined views that need a secondary object, prefer the single-object
`view:` form (§3d).

Write the array literally in the URL (brackets, quotes, commas). Do not
URL-encode it.

---

## 4. Plugin / view catalog

`<PluginName>` values usable in `tab=view:<PluginName>|...` (and, without the
`view:` prefix, inside a multi-object array). "Applies to" is the NWB
`neurodata_type` (and required structure) the plugin can handle. When you use a
bare object path (§3a), Neurosift renders **all** matching plugins from this
list, stacked in this order (the first applicable one appears at the top).

| PluginName | Label in UI | Applies to (neurodata_type / structure) |
|------------|-------------|------------------------------------------|
| `Events` | Events | `Events` (ndx-events) with a `timestamps` dataset |
| `IntervalSeries` | Interval Series | `IntervalSeries` |
| `BehavioralEvents` | BehavioralEvents | `BehavioralEvents` |
| `dynamic-table` | (table) | any group with a `colnames` attribute — i.e. any `DynamicTable` (Units, trials, electrodes, epochs, …) |
| `TwoPhotonSeries` | TwoPhotonSeries | `TwoPhotonSeries`, `OnePhotonSeries`, or `ImageSeries` with inline `data` |
| `SpatialSeriesXY` | XY | `SpatialSeries` (and subtypes) whose `data` is 2-D with 2 columns (x,y) |
| `SimpleTimeseries` | Timeseries | any group with a `data` dataset (1-D or 2-D) + `timestamps`/`starting_time`, no `external_file` — the general timeseries line plot |
| `PSTH` | PSTH | primary `TimeIntervals` + secondary `Units` (needs secondary path; see §3d) |
| `Image` | Image | an `Image` dataset, or an `Images` group |
| `ImageSegmentation` | ImageSegmentation | `ImageSegmentation` |
| `PlaneSegmentation` | PlaneSegmentation | `PlaneSegmentation` (child of an `ImageSegmentation`) |
| `TimeIntervals` | TimeIntervals | `TimeIntervals` or `OptogeneticPulsesTable` with `start_time` + `stop_time` |
| `TrialAlignedSeries` | TrialAlignedSeries | primary `TimeIntervals` + secondary response series (needs secondary path; see §3d) |
| `PoseEstimation` | Pose overlay | `PoseEstimation` (ndx-pose) containing `PoseEstimationSeries` |
| `ExternalFileVideo` | Video | `ImageSeries` with an `external_file` dataset |
| `ImageSeriesMp4` | MP4 Video | `TwoPhotonSeries` or `OnePhotonSeries` |
| `Raster` | Raster | `Units` (spike raster) |
| `FigpackRasterPlot` | Figpack Raster | `Units` (figpack-based raster) |
| `FigpackVideoPreview` | Video Preview | certain `ImageSeries`/imaging types (figpack-based) |
| `FigpackPoseEstimation` | Pose Estimation | `PoseEstimation` (figpack-based) |
| `default` | (default) | any object — raw/default object view |
| `PythonScript` | (Python) | any object — shows a Python usage snippet |

Notes:

- Names are **case-sensitive** and exactly as written above (e.g.
  `SpatialSeriesXY`, `dynamic-table`).
- The most commonly linked views: `Raster` and `FigpackRasterPlot` (Units),
  `dynamic-table` and `TimeIntervals` (tables/intervals), `SimpleTimeseries`
  (generic signals), `SpatialSeriesXY` (position), `TwoPhotonSeries` (imaging),
  `ImageSegmentation`/`PlaneSegmentation` (ROI masks), `PSTH` (spikes × trials).

---

## 5. Worked examples

These are **real and verified** against dandiset `001637`, version `draft`, file
`sub-820454/sub-820454_ses-ecephys-820454-2025-11-04-14-59-22_ecephys.nwb`
(asset id `425e5654-6a0b-4995-abe1-077fb886d138`). The object paths
(`/units`, `/processing/eye_tracking`, `/intervals/spontaneous_presentations`,
`/processing/running/running_speed`) all exist in this file. When you adapt these
for another file, swap in a `path` and object paths that actually exist there.

Open the file (path-based):

```
https://neurosift.app/nwb?dandisetId=001637&dandisetVersion=draft&path=sub-820454/sub-820454_ses-ecephys-820454-2025-11-04-14-59-22_ecephys.nwb
```

Open it straight to the eye-tracking processing group (all applicable views):

```
https://neurosift.app/nwb?dandisetId=001637&dandisetVersion=draft&path=sub-820454/sub-820454_ses-ecephys-820454-2025-11-04-14-59-22_ecephys.nwb&tab=/processing/eye_tracking
```

Open it to a spike raster of the units table (forced view):

```
https://neurosift.app/nwb?dandisetId=001637&dandisetVersion=draft&path=sub-820454/sub-820454_ses-ecephys-820454-2025-11-04-14-59-22_ecephys.nwb&tab=view:Raster|/units
```

Open it to a PSTH of `/units` aligned to `/intervals/spontaneous_presentations`:

```
https://neurosift.app/nwb?dandisetId=001637&dandisetVersion=draft&path=sub-820454/sub-820454_ses-ecephys-820454-2025-11-04-14-59-22_ecephys.nwb&tab=view:PSTH|/intervals/spontaneous_presentations^/units
```

Open it with two objects side by side:

```
https://neurosift.app/nwb?dandisetId=001637&dandisetVersion=draft&path=sub-820454/sub-820454_ses-ecephys-820454-2025-11-04-14-59-22_ecephys.nwb&tab=["/processing/running/running_speed","/units"]
```

By asset download URL instead of path (equivalent to the file above):

```
https://neurosift.app/nwb?url=https://api.dandiarchive.org/api/assets/425e5654-6a0b-4995-abe1-077fb886d138/download/&dandisetId=001637&dandisetVersion=draft&tab=/processing/eye_tracking
```

Embedded (no app bar), for an iframe:

```
https://neurosift.app/nwb?dandisetId=001637&dandisetVersion=draft&path=sub-820454/sub-820454_ses-ecephys-820454-2025-11-04-14-59-22_ecephys.nwb&tab=/units&embedded=1
```
