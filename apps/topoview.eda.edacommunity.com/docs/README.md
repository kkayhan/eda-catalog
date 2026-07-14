# EDA TopoView

Turn any EDA-managed fabric into a live Grafana **topology map** — automatically.
Install once and TopoView bundles a Prometheus + Grafana stack (no Loki — EDA has
native logging), auto-discovers every namespace that has a fabric, and generates a
live topology dashboard for each one straight from the `TopoNode` / `TopoLink` CRs.

Border-leaf switches render on the top row, spines in the middle, leaves at the
bottom, with dual-direction links showing per-direction throughput and up/down
state, and each leaf's edge (host-facing) interfaces listed with live in/out bps.
Grafana is read-only (no login): open it, pick a namespace from the menu, and see
its fabric. When the topology changes, the dashboard regenerates automatically.

## How to use

1. Install from the EDA App Store (requires the `prom.eda.nokia.com` exporter app).
2. Open `https://<your-eda-address>/core/httpproxy/v1/grafana/?kiosk`.
3. Pick a namespace from the menu — that fabric's live topology appears.

Source: https://github.com/kkayhan/edaapp_grafana
