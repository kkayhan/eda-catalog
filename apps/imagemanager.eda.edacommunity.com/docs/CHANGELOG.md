# Changelog

## v26.8.2-1 — EDA 26.8 line

- Re-baselined to EDA 26.8 (Core API `v6.0.0`, built with `edabuilder v26.8.2`). The 26.4 line stays at `v26.4.2-27`.
- Fix: the web UI was blank on EDA 26.8 — the 26.8 `eda-api` proxy adds a `Content-Security-Policy: default-src 'self'` header to proxied pages that carry none, which blocked the UI's inline script and styles. The app now sends its own policy on every HTML page.
- Verified on EDA 26.8.2: SR Linux and SR OS zip uploads (image + md5 + schema profile Artifacts), URL import, license attach (EDA 26.8 also turns the license ConfigMap into a `licenses` Artifact for the node), SR-SIM upload served from the app's registry endpoint and pulled by the node, simulator boot.
- Docs: on EDA 26.8+ a node must carry the label `eda.nokia.com/security-profile: managed` (or match another NodeSecurityProfile) or EDA never mints its bootstrap certificate and an SR OS simulator exits before booting; the generated NodeProfile examples say so. Talos registry-mirror patch documented as a strategic-merge YAML patch (JSON-6902 patches are refused on multi-document machine configs).
