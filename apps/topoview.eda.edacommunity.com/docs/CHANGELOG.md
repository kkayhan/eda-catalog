# Changelog

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
