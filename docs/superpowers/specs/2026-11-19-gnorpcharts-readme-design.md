# GnorpCharts Front-Page README — Design

**Date:** 2026-11-19
**Status:** Approved (the design below was approved by the user before this spec was written)
**Repo:** `GnorpLabs/GnorpCharts` (`/home/mack/repos/kube/GnorpCharts`), default branch `main`
**Branch:** `agent/readme-frontpage` (from `origin/main`)
**Memory project:** `gnorpcharts`
**Intended location:** `GnorpCharts/docs/superpowers/specs/2026-11-19-gnorpcharts-readme-design.md` — see "Document location" below.

## Intent / why

`README.md` in GnorpCharts is currently a single line — `# GnorpCharts` — while the repo
publishes 15 active Helm charts to a live GitHub Pages index at
`https://gnorplabs.github.io/GnorpCharts`. An outside Helm user who lands on the repo has no way
to learn the `helm repo add` URL, which charts exist, or that two chart names are deprecated but
still resolvable from the index. This is a **full rewrite of a stub**, not an edit of an existing
document.

## Goals

- Give an outside Helm user the three things they came for: what this repo is, how to add the
  repo, and which charts they can install.
- State the per-chart configuration entry point (`helm show values`) instead of duplicating
  values documentation in the README.
- Warn about the two deprecated chart names that are still installable from the published index.
- Stay lean: four sections, no decoration, nothing that rots on every chart release.

## Non-goals

See "Out of scope" below. Each exclusion was decided by the user and must not be re-added by a
future editor without a new decision.

## Audience and tone

**Audience:** outside Helm users who found the repo and want to install a chart.
**Tone:** lean. Exactly four sections, in the order below.

## Decided design

### 1. Title and pitch

- A single H1: `# GnorpCharts`.
- At most two sentences: a Helm chart repository from GnorpLabs covering self-hosted media
  automation, dedicated game servers, and AI/agent infrastructure, developed against k3s.
- **No badges.**
- **No license section.** There is no `LICENSE` file in this repo, so claiming a license would
  be false.

### 2. Install

A verified-correct snippet, which must appear as a fenced block with the `bash` language tag:

```bash
helm repo add gnorpcharts https://gnorplabs.github.io/GnorpCharts
helm repo update
helm search repo gnorpcharts
```

Plus **one** sentence telling readers that `helm show values gnorpcharts/<chart>` is the
per-chart configuration reference. That sentence intentionally **replaces** per-chart values
documentation in the README.

The repo URL is confirmed live: `https://gnorplabs.github.io/GnorpCharts/index.yaml` returns a
valid Helm index, served from the `gh-pages` branch.

### 3. Chart index — 15 active published charts, three tables

Columns are **Chart** and **Description** only.

**No version column — deliberate.** `index.yaml` is the source of truth for versions, and
hardcoded versions in the README rot on every chart release. A future editor must not
"helpfully" add a version column.

#### Media automation (9)

| Chart | Description |
| --- | --- |
| `kube-plex` | Plex Media Server with distributed transcoding (upstream `ressu/kube-plex`) |
| `sonarr` | Smart PVR for TV via newsgroups and BitTorrent |
| `radarr` | Movie collection manager |
| `prowlarr` | Indexer manager and proxy for the *arr apps |
| `seerr` | Media request and discovery manager (Jellyseerr/Overseerr lineage) |
| `tautulli` | Plex monitoring, analytics and notifications |
| `kometa` | Plex metadata, collection and artwork automation (formerly PMM) |
| `imagemaid` | Plex image/cache cleanup utility |
| `clonarr` | Visual TRaSH Guides sync for Radarr and Sonarr |

#### Game servers (2)

| Chart | Description |
| --- | --- |
| `astroneer` | Astroneer dedicated server via AstroTuxLauncher |
| `satisfactory` | Satisfactory dedicated server |

#### AI and platform (4)

| Chart | Description |
| --- | --- |
| `agentmemory` | Persistent memory server for AI coding agents |
| `mcp-gateway` | Universal MCP gateway with Meta-MCP |
| `omniroute` | Unified AI proxy for many LLM providers |
| `portainer` | Container management UI |

#### Chart.yaml descriptions (known inconsistency, not fixed here)

Seven published charts carry the placeholder `description: A Helm chart for Kubernetes` in their
`Chart.yaml`: `imagemaid`, `kometa`, `overseerr`, `prowlarr`, `radarr`, `seerr`, `tautulli`. The
user decided to write accurate human one-liners in the README **now** and to leave every
`Chart.yaml` untouched in this change. The README descriptions above are therefore intentionally
better than the index descriptions for those charts. See "Follow-up".

### 4. Deprecations

A GitHub alert block using `> [!IMPORTANT]`, followed by a two-row table:

| Deprecated | Use instead |
| --- | --- |
| `recyclarr` | `clonarr` |
| `overseerr` | `seerr` |

**Rationale to state in the README:** both names remain resolvable from the published Helm index,
so a user can still install them by accident — which is exactly why the warning is needed.
`overseerr` is already gone from `main` (it now lives at `charts/deprecated/overseer`) and
`recyclarr` is marked `deprecated: true` in its `Chart.yaml`. Neither appears in the active chart
index above.

## Out of scope (user declined — do not re-add)

- **The six branch-only charts** `cert-manager`, `gpu-operator`, `grafana`, `home-assistant`,
  `palworld`, `plex`. None of them is a publishable chart on `origin/main` (the paths that exist
  there hold vendored upstream copies with no top-level `Chart.yaml`, which is why
  chart-releaser never published them) and none appears in `index.yaml`. Listing them would send
  users into `chart not found`.
- **Contributing / release-flow section.** There is no `CONTRIBUTING.md` in this repo.
- **Quick-start install example** beyond the three `helm repo` commands.
- **Badges.**
- **Requirements / "tested on" section.**
- **License section** (no `LICENSE` file exists).
- **Any edit to `Chart.yaml`, `values.yaml`, templates, or workflows.** This change touches
  `README.md` only, plus the planning docs under `docs/superpowers/`.

## Follow-up (explicitly deferred)

**Clean up the seven placeholder `Chart.yaml` descriptions** (`imagemaid`, `kometa`, `overseerr`,
`prowlarr`, `radarr`, `seerr`, `tautulli`) so the published index matches the README wording.
Excluded from this change because editing `Chart.yaml` requires a chart version bump, and a
`charts/**` change on merge triggers `release-charts.yml` — a real chart release. That deserves
its own PR with its own CI impact.

## CI impact (verified from `.github/workflows/`)

The only workflow in this repo is `release-charts.yml`: `helm/chart-releaser-action@v1.6.0`,
triggered on push to `main` filtered to paths `charts/**`, plus `workflow_dispatch`. This change
touches only `README.md` and files under `docs/superpowers/`, so:

- **Push to `agent/readme-frontpage`:** no workflow runs (there is no `pull_request` trigger and
  no non-`main` branch trigger).
- **On the PR:** nothing runs.
- **On merge to `main`:** nothing runs, because the path filter `charts/**` matches neither
  `README.md` nor `docs/**`.
- No chart is released, no secrets are used, no self-hosted runner is involved.

**This is a documentation-only, zero-CI-impact change.** The PR description must open with a
`CI impact` section stating exactly this.

## Constraints

- Branch `agent/readme-frontpage`, based on `origin/main`. One PR. A human merges; never push to
  `main`, never dispatch `release-charts.yml`.
- Chart names in the README must match the published index character-for-character; a typo
  produces `chart not found` for a reader.
- Markdown quality follows the `pro-markdown` preflight (see the plan's verification step).

## Document location

This spec and its plan belong in the GnorpCharts checkout at
`GnorpCharts/docs/superpowers/{specs,plans}/`. The planning agent's write permission is scoped to
the session root's `docs/superpowers/`, so the files were written to
`/home/mack/repos/kube/docs/superpowers/` instead. Move both files into
`GnorpCharts/docs/superpowers/` before committing them on `agent/readme-frontpage`; the plan's
Task 1 covers this.

## What is left to the builder

- The exact two sentences of the pitch, the single `helm show values` sentence, and the
  alert-block wording — within the constraints above.
- Markdown mechanics: table padding, heading levels under the H1, placement of the alert block.

## Open questions

None blocking.
