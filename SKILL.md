---
name: neurosift-links
description: >-
  Construct URLs that open specific Neurosift (neurosift.app) views of DANDI
  data — the DANDI browser, a dandiset page, or an NWB file opened to a
  particular neurodata object or visualization. Use whenever you want to give the
  user a clickable Neurosift link: to a dandiset, to an NWB file, or to a
  specific object or plot inside an NWB file (e.g. a Units raster, a PSTH, a
  spatial-series X/Y plot, a dynamic table, an imaging series). Assumes you
  already know the dandiset id, the asset path or asset download URL, and any
  neurodata object paths — obtain those from the neurosift-dandiset /
  neurosift-datasets skills or the DANDI API; this skill is only about turning
  them into URLs.
---

# Building Neurosift URLs

[Neurosift](https://neurosift.app) is a browser-based viewer for neuroscience
data, focused on **NWB** (Neurodata Without Borders) files in the
[DANDI Archive](https://dandiarchive.org). This skill is a reference for
constructing URLs that open Neurosift directly on a chosen view — a dandiset, an
NWB file, or a specific object/visualization inside an NWB file — so you can hand
the user a link that lands exactly where you mean.

This skill covers **DANDI and NWB only**. The base host is always
`https://neurosift.app`.

You supply the identifiers; this skill turns them into URLs. Don't invent paths;
use ones you've actually confirmed.

## Companion skills

This is one of three Neurosift skills that work together — for neuroscience data
on the DANDI Archive (plus OpenNeuro / EBRAINS for search):

- **neurosift-datasets** — find, filter, rank, and count datasets **across**
  DANDI, OpenNeuro, and EBRAINS, and see which neurodata types (Units,
  ElectricalSeries, …) a file contains. *Discovery across many datasets.*
- **neurosift-dandiset** — deep-dive a **single** dandiset or NWB file: read its
  metadata, inspect its contents, and load / visualize / analyze the data in
  Python by streaming the remote file. *Depth on one file.*
- **neurosift-links** *(this skill)* — construct **neurosift.app** URLs that open
  a dandiset, an NWB file, or a specific object/visualization in the interactive
  web viewer. *Clickable views to share.*

Typical flow: **neurosift-datasets** (discover) → **neurosift-dandiset**
(analyze) → **neurosift-links** (share an interactive view); each also works on
its own. This skill only builds URLs — get the identifiers it needs (dandiset
ids, asset paths, and neurodata object paths) from **neurosift-datasets**
(search across archives), **neurosift-dandiset** (deep-dive one file), or the
DANDI REST API. Whenever those skills report a dandiset, an NWB file, or an
object, this skill turns it into a clickable Neurosift link.

## Golden rules

- **Do not URL-encode the `url` value.** Put the asset download URL in as a
  plain, readable URL. Neurosift handles it. (Same for object paths in `tab=` —
  write them literally, e.g. `tab=/processing/eye_tracking`.)
- **Version defaults to `draft`.** If you don't know the published version, use
  `draft`. Include `dandisetVersion` when you can — a published version looks
  like `0.240101.1234`.
- **Prefer the path-based NWB form** (`dandisetId` + `dandisetVersion` + `path`)
  when you know the asset path — it's the most readable and Neurosift resolves
  the download URL itself. Use the `url=` form when you have the download URL in
  hand.
- **Emit links as markdown** with a short human label:
  `[<label>](<url>)`.

## The pages

| View | URL |
|------|-----|
| Landing page | `https://neurosift.app` |
| Browse / search DANDI | `https://neurosift.app/dandi` |
| A dandiset | `https://neurosift.app/dandiset/<dandisetId>` |
| A dandiset, specific version | `https://neurosift.app/dandiset/<dandisetId>?dandisetVersion=<version>` |
| An NWB file | see **Linking to an NWB file** below |

Notes:

- `<dandisetId>` is the 6-digit id, e.g. `001637`.
- The `/dandi` browse page holds its search state internally — there is **no**
  search query parameter to preset. Link to `/dandi` to open the browser; you
  can't pre-fill a search via the URL.
- To point at a *file*, don't use the dandiset page — link straight to `/nwb`
  (below). The dandiset page is for browsing.

## Linking to an NWB file

The NWB viewer lives at `https://neurosift.app/nwb`. There are two ways to tell
it which file to open.

**Path-based (preferred when you know the asset path).** Neurosift looks the
asset up in the DANDI API and resolves the download URL for you:

```
https://neurosift.app/nwb?dandisetId=001637&dandisetVersion=draft&path=sub-820454/sub-820454_ses-ecephys-820454-2025-11-04-14-59-22_ecephys.nwb
```

The `path` **must exactly match an asset** in that dandiset/version (Neurosift
resolves it via the DANDI `assets/?glob=<path>` API). If it doesn't match, the
viewer never gets a URL and shows **"No NWB file URL provided."** — that error
almost always means a wrong/nonexistent `path` (or wrong `dandisetId` /
`dandisetVersion`), so verify the asset path against the DANDI API. Use a path
you've actually confirmed; do not guess file names.

**URL-based (when you have the asset download URL).** Pass the DANDI asset
download URL directly as `url` (unencoded). Keep `dandisetId` and
`dandisetVersion` for context (they drive the sidebar and version switcher):

```
https://neurosift.app/nwb?url=https://api.dandiarchive.org/api/assets/425e5654-6a0b-4995-abe1-077fb886d138/download/&dandisetId=001637&dandisetVersion=draft
```

DANDI asset download URLs have the form
`https://api.dandiarchive.org/api/assets/<asset_id>/download/`.

The `url=` form also works for **any** remote NWB (or LINDI) file, not just DANDI
— just omit `dandisetId`/`dandisetVersion` if it isn't a dandiset asset.

## Opening the file to a specific object or view: the `tab` parameter

Add `&tab=...` to an `/nwb` URL to open the file focused on a particular
neurodata object or visualization instead of the default overview. This is the
most useful thing this skill does. The value is one of:

1. **A neurodata object path** — opens that object, showing every view that
   applies to it (stacked). This is what you want most of the time:

   ```
   https://neurosift.app/nwb?dandisetId=001637&dandisetVersion=draft&path=sub-820454/sub-820454_ses-ecephys-820454-2025-11-04-14-59-22_ecephys.nwb&tab=/processing/eye_tracking
   ```

2. **A fixed tab** — one of the built-in views of the whole file:
   `widgets` (the default hierarchy view), `hdf5`, `specifications` (the NWB
   schema), `python-usage`, `timeseries-alignment`, `video-widget`, `icephys`.
   Example: `&tab=hdf5`.

3. **A specific plugin view of one object** — force a particular visualization
   when several could apply: `&tab=view:<PluginName>|<path>`. Example, a spike
   raster of the units table:

   ```
   ...&tab=view:Raster|/units
   ```

4. **A view that needs a secondary object** (e.g. a PSTH needs both a trials
   table and a units table): append secondary paths with `^`:
   `&tab=view:<PluginName>|<primaryPath>^<secondaryPath>`. Example, a PSTH of
   `/units` aligned to `/intervals/trials`:

   ```
   ...&tab=view:PSTH|/intervals/trials^/units
   ```

5. **Multiple objects in one tab** — a JSON array of entries:
   `&tab=["/path1","/path2"]`. Each entry can be a bare path (auto view) or
   `PluginName|path` (note: **no** `view:` prefix inside the array). Example:

   ```
   ...&tab=["/acquisition/ElectricalSeries","/units"]
   ```

**In almost all cases, prefer form 1 (a bare object path).** Neurosift shows all
the applicable views for that object automatically. Reach for
`view:<PluginName>|...` (forms 3–4) only when you want to pin the file to one
specific visualization, or when the view *needs* a secondary object (PSTH,
TrialAlignedSeries).

For the complete list of plugin names, which neurodata type each one applies to,
and the exact grammar (including the single-object vs. multi-object prefix
difference and other edge cases), see
**[reference/url-patterns.md](reference/url-patterns.md)**.

## Optional parameters

- `embedded=1` — hides the Neurosift top app bar. Use for embedding a view in an
  iframe. Example: `...&tab=/units&embedded=1`.
- `lindi=0` — tells Neurosift not to try to use a LINDI version of the file.
  Rarely needed.

## Quick recipes

These use a real, working file (dandiset `001637`, `draft`,
`sub-820454/sub-820454_ses-ecephys-820454-2025-11-04-14-59-22_ecephys.nwb`). Let
`BASE` be its path-based `/nwb` URL:

```
https://neurosift.app/nwb?dandisetId=001637&dandisetVersion=draft&path=sub-820454/sub-820454_ses-ecephys-820454-2025-11-04-14-59-22_ecephys.nwb
```

- **A dandiset:**
  `[Dandiset 001637](https://neurosift.app/dandiset/001637)`
- **An NWB file:** `[View in Neurosift](BASE)`
- **Open to the eye-tracking group:** `BASE` + `&tab=/processing/eye_tracking`
- **Units raster:** `BASE` + `&tab=view:Raster|/units`
- **A stimulus intervals table:** `BASE` + `&tab=/intervals/spontaneous_presentations`
- **PSTH of /units aligned to those intervals:**
  `BASE` + `&tab=view:PSTH|/intervals/spontaneous_presentations^/units`

(For a spatial-series X/Y plot in some *other* file, the pattern is
`&tab=view:SpatialSeriesXY|<path-to-a-SpatialSeries>` — this particular file has
no SpatialSeries.)

## Reference

- **[reference/url-patterns.md](reference/url-patterns.md)** — full route and
  parameter grammar, the complete plugin/view catalog (names + neurodata types),
  secondary-path views, multi-object tabs, and the exact `tab` syntax rules.
