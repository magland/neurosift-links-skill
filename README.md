# neurosift-links skill

A [Claude Agent Skill](https://docs.claude.com/en/docs/claude-code/skills) for
**constructing URLs that open [Neurosift](https://neurosift.app) on a specific
view** of DANDI / NWB data. Give Claude a dandiset id, an asset path (or download
URL), and optionally a neurodata object path, and it produces a clickable link
that lands exactly where you mean — the DANDI browser, a dandiset page, an NWB
file, or a specific object or visualization inside that file (a Units raster, a
PSTH, a spatial-series X/Y plot, a dynamic table, an imaging series, …).

Scope: **DANDI and NWB only.** The host is always `https://neurosift.app`.

> This skill is purely about **turning identifiers into URLs**. To *find* the
> identifiers — dandisets, NWB files, neurodata object paths — use the companion
> [neurosift-dandiset](https://github.com/magland/neurosift-dandiset-skill)
> (deep-dive on one dandiset/file) and
> [neurosift-datasets](https://github.com/magland/neurosift-datasets-skill)
> (search across datasets) skills, or the DANDI REST API.

## What it can do

- Link to the landing page, the DANDI browser, or a dandiset (with version)
- Link to an NWB file two ways: by asset path (Neurosift resolves the download
  URL) or by asset download URL
- Open an NWB file focused on a specific neurodata object (`tab=/path`)
- Force a specific visualization (`tab=view:Raster|/units`), including views that
  combine a primary and secondary object (PSTH, TrialAlignedSeries)
- Open multiple objects in one tab
- Add options like `embedded=1` (hide app bar, for iframes)

Example prompts:

- "Give me a Neurosift link to dandiset 001637."
- "Link to that NWB file opened to its units raster."
- "Make a Neurosift URL showing a PSTH of /units aligned to the trials table."

## Install

**As a personal skill (all your Claude Code sessions):**

```bash
git clone <this-repo> ~/.claude/skills/neurosift-links
```

**As a project skill (one repo):**

```bash
git clone <this-repo> /path/to/your/project/.claude/skills/neurosift-links
```

The directory name must be `neurosift-links` (it matches the skill's `name`).
Claude loads the skill automatically when you ask for a Neurosift link.

## Contents

| Path | Purpose |
|------|---------|
| `SKILL.md` | Skill definition + the core URL patterns Claude follows |
| `reference/url-patterns.md` | Full route/parameter grammar, the complete plugin/view catalog, secondary-path and multi-object tabs, worked examples |

## No dependencies

This skill is pure documentation — no scripts, no packages, nothing to install
beyond dropping the folder in place.
