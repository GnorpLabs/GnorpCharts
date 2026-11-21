# GnorpCharts

A Helm chart repository from GnorpLabs covering self-hosted media automation, dedicated game
servers, and AI/agent infrastructure. Developed and tested against k3s.

## Install

```bash
helm repo add gnorpcharts https://gnorplabs.github.io/GnorpCharts
helm repo update
helm search repo gnorpcharts
```

For per-chart configuration options, run `helm show values gnorpcharts/<chart>`.

## Charts

### Media automation

| Chart | Description |
| --- | --- |
| `kube-plex` | Plex Media Server with distributed transcoding (upstream `ressu/kube-plex`) |
| `sonarr` | Smart PVR for TV via newsgroups and BitTorrent |
| `radarr` | Movie collection manager |
| `prowlarr` | Indexer manager and proxy for the `*arr` apps |
| `seerr` | Media request and discovery manager (Jellyseerr/Overseerr lineage) |
| `tautulli` | Plex monitoring, analytics and notifications |
| `kometa` | Plex metadata, collection and artwork automation (formerly PMM) |
| `imagemaid` | Plex image/cache cleanup utility |
| `clonarr` | Visual TRaSH Guides sync for Radarr and Sonarr |

### Game servers

| Chart | Description |
| --- | --- |
| `astroneer` | Astroneer dedicated server via AstroTuxLauncher |
| `satisfactory` | Satisfactory dedicated server |

### AI and platform

| Chart | Description |
| --- | --- |
| `agentmemory` | Persistent memory server for AI coding agents |
| `mcp-gateway` | Universal MCP gateway with Meta-MCP |
| `omniroute` | Unified AI proxy for many LLM providers |
| `portainer` | Container management UI |

## Deprecations

> [!IMPORTANT]
> Both deprecated names below are still resolvable from the published Helm index, so they can
> still be installed by accident. Use the replacement chart instead.

| Deprecated | Use instead |
| --- | --- |
| `recyclarr` | `clonarr` |
| `overseerr` | `seerr` |
