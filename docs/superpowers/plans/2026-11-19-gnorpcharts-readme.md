# GnorpCharts Front-Page README — Implementation Plan

> **For agentic workers:** implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax
> for tracking. Do not expand scope; the exclusions in the spec are user decisions.

**Goal:** Replace the one-line `README.md` stub in GnorpCharts with a lean four-section front
page: title and pitch, install, a 15-chart index in three tables, and a deprecations warning.

**Architecture:** One file, `README.md`, at the repo root of `GnorpLabs/GnorpCharts`. No chart,
values, template or workflow change. Content is fully specified in the design doc; the author
supplies wording within those constraints and the markdown mechanics.

**Tech Stack:** Markdown (GitHub-flavoured, including `> [!IMPORTANT]` alerts), Helm CLI for
read-only verification against the published index.

**Spec:** `docs/superpowers/specs/2026-11-19-gnorpcharts-readme-design.md` (to be moved into
`GnorpCharts/docs/superpowers/specs/` — see Task 1 Step 1).

**Repo / branch:** `/home/mack/repos/kube/GnorpCharts`, branch `agent/readme-frontpage`, based on
`origin/main`.

## Global constraints

- **Scope is one file.** `README.md` only, plus the two planning docs under
  `docs/superpowers/`. Never touch `Chart.yaml`, `values.yaml`, templates, or
  `.github/workflows/`.
- **Do not re-add the exclusions.** No badges, no license section, no contributing/release
  section, no requirements section, no quick-start beyond the three `helm repo` commands, no
  version column in the chart tables, and none of the six branch-only charts (`cert-manager`,
  `gpu-operator`, `grafana`, `home-assistant`, `palworld`, `plex`).
- **15 active charts, exactly:** media automation 9, game servers 2, AI and platform 4. The
  deprecated `recyclarr` and `overseerr` appear only in the deprecations table.
- Commit with explicit-path `git add`. Commit messages short imperative, `docs:` prefixed.
- **Every `git push` needs explicit user approval.** Open one PR and stop; a human merges. Never
  push to `main`, never run `gh workflow run`.

## Review focus

- **Chart-name fidelity:** a single typo in a chart name gives the reader `chart not found`.
  Names must match the published `index.yaml` character-for-character.
- **Working install snippet:** the `helm repo add` URL must be
  `https://gnorplabs.github.io/GnorpCharts` and the block must carry the `bash` language tag.
- **No version numbers anywhere** in the chart tables (they rot on every release).
- **Deprecations rationale present:** the README must say *why* the warning exists — both names
  are still resolvable from the published index.
- **No false claims:** no license statement, no "tested on" promises, no contributing process
  that does not exist.

## CI impact (verified from `.github/workflows/`; goes verbatim into the PR description)

The only workflow in this repo is `release-charts.yml` (`helm/chart-releaser-action@v1.6.0`), on
push to `main` filtered to paths `charts/**`, plus `workflow_dispatch`. This change touches only
`README.md` and `docs/superpowers/**`, so:

- **Push to `agent/readme-frontpage`:** nothing runs — no `pull_request` trigger, no non-`main`
  branch trigger exists.
- **On the PR:** nothing runs.
- **On merge to `main`:** nothing runs — the `charts/**` path filter matches neither `README.md`
  nor `docs/**`.
- No chart released, no secrets used, no self-hosted runner involved.

**Documentation-only, zero-CI-impact change.** The PR description must open with a `CI impact`
section stating this.

---

### Task 1: Write the front-page README

**Owner:** `docs-sync` (sole writer; also the final writer — it opens the single PR)

**Depends on:** nothing.

**Files:**

- Modify: `README.md` (full rewrite of the one-line stub `# GnorpCharts`)
- Move into place: `docs/superpowers/specs/2026-11-19-gnorpcharts-readme-design.md` and
  `docs/superpowers/plans/2026-11-19-gnorpcharts-readme.md` (currently written at the session root
  `/home/mack/repos/kube/docs/superpowers/` because the planner could not write inside the
  GnorpCharts checkout)

**Risk:** Local-only. No cluster, infra or CI contact. The only approval needed is the `git push`
(Step 7); a human merges the PR.

- [ ] **Step 1: Branch hygiene.** Confirm you are in `/home/mack/repos/kube/GnorpCharts` on
      `agent/readme-frontpage` and that the branch contains **nothing but** `origin/main` plus the
      planning commit. Run `git log --oneline origin/main..HEAD`. Expected: empty, or only this
      plan's docs commit. **If it lists `f022106 feat: httpBackends command/args/streamable_http,
      bump chart to 0.4.0`** (an unrelated mcp-gateway chart commit the planner could not avoid
      inheriting), reset the branch onto `origin/main` before doing anything else — e.g.
      `git fetch origin && git reset --hard origin/main` — then re-apply the docs. A `charts/**`
      commit on this branch would turn a zero-CI-impact PR into a chart release on merge.
      Verify after the reset: `git diff --stat origin/main..HEAD` lists only `README.md` and
      `docs/superpowers/**`.
- [ ] **Step 2: Move the planning docs into the repo.** Place the spec at
      `docs/superpowers/specs/2026-11-19-gnorpcharts-readme-design.md` and this plan at
      `docs/superpowers/plans/2026-11-19-gnorpcharts-readme.md` inside the GnorpCharts checkout.
      Expected: both paths exist under `GnorpCharts/docs/superpowers/` and neither is still
      referenced from the session root.
- [ ] **Step 3: Write `README.md`** with exactly four sections in this order, per the spec:
      (1) H1 `# GnorpCharts` plus at most two sentences of pitch (Helm chart repository from
      GnorpLabs; self-hosted media automation, dedicated game servers, AI/agent infrastructure;
      developed against k3s) — no badges, no license;
      (2) **Install** — the three-command `bash` fenced block (`helm repo add gnorpcharts
      https://gnorplabs.github.io/GnorpCharts`, `helm repo update`, `helm search repo
      gnorpcharts`) plus one sentence pointing at `helm show values gnorpcharts/<chart>` as the
      per-chart configuration reference;
      (3) **Charts** — three tables (Media automation 9, Game servers 2, AI and platform 4) with
      the Chart/Description rows copied from the spec, no version column;
      (4) **Deprecations** — a `> [!IMPORTANT]` alert explaining that both names are still
      resolvable from the published index (so they can be installed by accident), plus the
      two-row Deprecated/Use-instead table (`recyclarr` → `clonarr`, `overseerr` → `seerr`).
- [ ] **Step 4: `pro-markdown` preflight.** Apply the `pro-markdown` skill and confirm every
      item: exactly one H1; blank lines around headings, tables and fenced blocks; a language tag
      on every fence (`bash`); table pipes padded with a consistent column count per table;
      descriptive link text (no bare "here"); no trailing whitespace on any line; the file ends in
      exactly one newline. Expected: all checks pass with no exceptions.
- [ ] **Step 5: Chart-name verification against the published index (read-only).** Run
      `helm repo add gnorpcharts https://gnorplabs.github.io/GnorpCharts && helm repo update &&
      helm search repo gnorpcharts` (this also proves the snippet in Step 3 actually works).
      Expected: every one of the 15 chart names in the README appears in the output
      character-for-character, and the only published names *absent* from the README's active
      tables are `overseerr` and `recyclarr` (both present in the deprecations table instead).
      A name in the README that is not in the index, or an active chart in the index missing from
      the README, is a **fail** — fix the README, not the index.
- [ ] **Step 6: Final read-through.** Render-check the file (e.g. `gh markdown` preview or a local
      renderer) and confirm the `> [!IMPORTANT]` block is valid GitHub alert syntax and both
      tables render as tables. Then `git diff` and confirm the change set is `README.md` plus the
      two `docs/superpowers/` files and nothing else.
- [ ] **Step 7: Commit.** `git add README.md`, then
      `git add docs/superpowers/specs/2026-11-19-gnorpcharts-readme-design.md`, then
      `git add docs/superpowers/plans/2026-11-19-gnorpcharts-readme.md` (one path per command),
      then `git commit -m "docs: add front-page README with chart index and deprecations"`.
- [ ] **Step 8: Push + PR (needs approval).** Request push approval, stating the CI impact: a push
      to `agent/readme-frontpage` runs nothing, and the merge runs nothing because the only
      workflow filters on `charts/**`. Then open one PR to `main` with `gh pr create`; the
      description must **start** with the `CI impact` section above. **Do not merge** — the user
      merges. Do not open a second PR if one already exists for this branch; push to the branch
      instead.

---

## Execution order

| # | Task | Owner | Branch | Depends on | Live impact |
|---|---|---|---|---|---|
| 1 | Write the front-page README (final writer, opens the PR) | `docs-sync` | GnorpCharts `agent/readme-frontpage` | — | none; docs only |

One writer, one branch, one PR. Approvals needed: the single `git push` (Step 8) and the human
merge. No command in this plan touches the cluster, Terraform, or CI.
