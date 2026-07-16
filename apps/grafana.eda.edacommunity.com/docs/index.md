# Grafana Application

-{{% import 'icons.html' as icons %}}-

| <nbsp> {: .hide-th } |                                         |
| -------------------- |-----------------------------------------|
| **Group/Version**    | -{{ app_group }}-/-{{ app_api_version }}-   |
| **Catalog**          | [kkayhan/eda-catalog/grafana][manifest] |
| **Source Code**      | [kkayhan/edaapp_grafana][src]           |

[manifest]: https://github.com/kkayhan/eda-catalog/tree/main/apps/grafana.eda.edacommunity.com
[src]: https://github.com/kkayhan/edaapp_grafana

Turn any EDA-managed fabric into a live Grafana **topology map** — automatically.
Grafana bundles a Prometheus + Grafana stack (no Loki — EDA has native logging),
auto-discovers every namespace that has a fabric, and generates one live topology
dashboard per namespace straight from the `TopoNode` / `TopoLink` CRs.

* **Role-based layout** — border-leaf switches on the top row, spines in the
  middle, leaf switches at the bottom (from the `eda.nokia.com/role` label or the
  `Fabric` CR). Empty rows collapse; the canvas scales to any fabric size.
* **Live link telemetry** — every inter-switch link is a dual-direction arrow
  coloured by throughput with a live bits/sec label per direction, plus a port dot
  coloured by oper-state (green up / red down), refreshed every ~5s.
* **Edge interfaces, not host clutter** — each leaf lists its host-facing
  interfaces with live in/out bps instead of drawing host nodes.
* **Read-only, no login** — land on a menu, pick a fabric, view it. Grafana is
  anonymous read-only behind EDA's own authenticated proxy.
* **Change-aware** — dashboards regenerate when a fabric's structure changes and
  self-heal if Grafana restarts.

After install, open the read-only Grafana (while signed into the EDA UI) at
`https://<your-eda-address>/core/httpproxy/v1/grafana/?kiosk`, and pick a namespace
from the main menu.

Requires the `prom.eda.nokia.com` Prometheus exporter app (installed automatically
as a dependency).

The application provides the following components:

/// tab | Resources

<div class="grid" markdown>
<div markdown>

* **GrafanaConfig** — cluster-scoped global tuning (refresh interval, excluded
  namespaces, role tiers, thresholds) and live status. You normally don't edit it —
  auto-discovery drives everything.

</div>
</div>
///
