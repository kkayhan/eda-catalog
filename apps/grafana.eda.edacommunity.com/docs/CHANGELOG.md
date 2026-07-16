# Changelog

## v26.4.1-5

- **Renamed the app from "TopoView" to "Grafana".** No functional change — purely an
  identity rename. Group `topoview.eda.edacommunity.com` → `grafana.eda.edacommunity.com`,
  CRD kind `TopoViewConfig` → `GrafanaConfig`, images `topoview-{app,controller,grafana}`
  → `grafana-{app,controller,server}`, controller objects `eda-topoview` → `eda-grafana`.
  Because the API group changes, existing `TopoViewConfig` resources are replaced by
  `GrafanaConfig` on reinstall.

## v26.4.1-4

- **Show server-facing LAGs on leaf switches.** EDA models access LAGs (dual-homed
  server connections) as `interfaces.eda.nokia.com` Interface CRs of `spec.type: lag`
  — **not** as edge TopoLinks. Grafana only read TopoLinks, so these access ports were
  invisible on the leaves (only the border leaves, whose edge ports *do* have edge
  TopoLinks, rendered). The controller now also reads Interface CRs and lists each
  **active** LAG's member ports as host-facing edge cards under its leaf, so live
  throughput shows for them like any other port.
  - **Active only**: LAGs whose `status.operationalState` is `down` are hidden;
    `up`/`degraded` are shown (matches the operator's choice for a fabric with ~44 LAGs).
  - **Display vs telemetry**: the card shows the LAG's name (e.g. `lag1`) while keying
    in/out bps + oper-state on the physical member port (`ethernet-1/N`), which the EQL
    feed already reports — no new telemetry, no datasource change.
  - **No double-listing**: a member port that is already an inter-switch endpoint, or
    already surfaced via an edge TopoLink, is not repeated (dedup by node+interface).
  - Leaf edge lists stay a single column beneath the leaf (canvas grows to fit); the
    border-leaf/spine side-grid layout is unchanged. Namespaces without LAGs render
    byte-for-byte as before. Requires the `interfaces` list RBAC already granted by the
    app's ClusterRole.

## v26.4.1-3

- **Fix blank link/interface throughput on clusters that route Keycloak at
  `/core/proxy/v1/identity`.** The controller hardcoded the Keycloak base
  `/core/httpproxy/v1/keycloak`; on EDA deployments that expose identity at
  `/core/proxy/v1/identity` instead, every token request 404'd, so the EQL
  telemetry fetch failed and all in/out bps rendered blank (the topology still
  drew). `auth.py` now **probes the candidate Keycloak bases once and pins the
  working one** (`_ensure_kc_base`) — no route assumption.
- **Password-free EDA API auth via a self-provisioned service account.** Replaced
  the `eda-realm-auth-secret` password grant (an `admin/admin` bootstrap seed that
  401s once the operator changes the EDA admin password) with a dedicated
  `eda-grafana` Keycloak service-account client (client_credentials), provisioned
  via the KC master admin. Self-healing and revocable. RBAC no longer needs
  `eda-realm-auth-secret`. Same approach proven by `edaapp_UserAudit`.
- Rendering unchanged from v26.4.1-2 (side-placed border-leaf/spine edge lists).

## v26.4.1-2

- **Edge-interface lists moved off the links for upper-tier switches.** Border-leaf
  (and spine) host-facing interfaces were listed in a single column *below* the
  switch — directly on top of its downward inter-switch links, which made them
  overlap and hard to read. They are now placed to the **side** of the switch
  (outer left/right half) as a compact grid: at most 3 rows, growing into more
  columns as ports are added, so the list expands sideways into free canvas space
  instead of over the links. Left-half switches get a left card, right-half a right
  card; the canvas widens automatically if a side card would spill past the margin.
- **Leaf switches are unchanged** — their edge interfaces still grow as a single
  vertical column beneath each leaf (nothing sits below a leaf to collide with).
- Rendering only; no data, CRD, or datasource changes. Fully compatible with the
  same EDA releases as v26.4.1-1 (validated on a live 26.4.1 cluster).

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
