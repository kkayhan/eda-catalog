# Changelog

## v26.4.1-1

- **Re-baselined to target EDA 26.4.1** (validated on a live 26.4.1 cluster); the
  version suffix restarts at `-1`. Code is backward-compatible with 26.4.3.
- Fix the follow-on `KeyError` that v26.4.3-4's stale-edge fix exposed. That fix
  dropped stale edges inside `compute_layout`, but `svggen` still iterated the
  model's full edge list and looked up `layout[endpoints]` by index — so the
  dropped edge's index raised `KeyError` (reported as `KeyError: 8` on the tt-poc
  fabric) and left the fabric `degraded` with 0 dashboards. Stale edges (a TopoLink
  endpoint with no TopoNode) are now dropped once in `build_topology`, so the
  model's edge list and the layout's endpoints stay index-consistent; the stale
  local port is surfaced as a host-facing edge port instead of being lost.

## v26.4.3-4

- Fix a dashboard-rendering crash when a TopoLink references a node that has no
  TopoNode — a stale or external link endpoint (e.g. `leaf-3` when only `leaf-1`
  and `leaf-2` exist). The layout engine assumed every edge endpoint had a
  computed position; one such edge raised `KeyError` and left the whole fabric
  at `health=degraded` with 0 dashboards. Edges whose endpoints are not in the
  drawn node set are now dropped before layout, so one stale link can no longer
  block every dashboard.

## v0.1.0

- Initial release. Auto-discovers EDA fabrics and generates a live Grafana
  topology dashboard per namespace (border-leaf / spine / leaf), with
  per-interface throughput and oper-state, and a read-only main menu.
