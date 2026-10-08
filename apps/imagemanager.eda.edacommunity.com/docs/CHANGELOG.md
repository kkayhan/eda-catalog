# Changelog

## v26.8.2-2 — EDA 26.8 line

- Fix: an upload could stay **Failed** with `tls: failed to verify certificate: x509: certificate signed by unknown authority`. The trust-bundle ConfigMap (`imagemanager-trust-bundle`) that lets `eda-asvr` trust this app was written once and never updated, so it went stale when EDA re-keyed its internal CA or when it was left over from an earlier install (an uninstall keeps it). The app now checks it every reconcile cycle (60 s) and rewrites it only when a CA it needs is missing; `eda-asvr` then retries the Failed pulls at once, with no re-upload. Managed HTTPS Artifacts that lack `spec.trustBundle` get it set.
- Fix: the file server loaded its certificate once at start, so a pod running longer than the certificate's 30-day lifetime would serve an expired certificate and every pull would fail. New connections now use the certificate the cert-manager CSI driver renews on disk.
- Fix: the `HttpProxy` manifest names its namespace (`eda-system`), so a direct `kubectl apply` install no longer puts it in `default`.

## v26.8.2-1 — EDA 26.8 line

- Re-baselined to EDA 26.8 (Core API `v6.0.0`, built with `edabuilder v26.8.2`). The 26.4 line stays at `v26.4.2-27`.
- Fix: the web UI was blank on EDA 26.8 — the 26.8 `eda-api` proxy adds a `Content-Security-Policy: default-src 'self'` header to proxied pages that carry none, which blocked the UI's inline script and styles. The app now sends its own policy on every HTML page.
- Verified on EDA 26.8.2: SR Linux and SR OS zip uploads (image + md5 + schema profile Artifacts), URL import, license attach (EDA 26.8 also turns the license ConfigMap into a `licenses` Artifact for the node), SR-SIM upload served from the app's registry endpoint and pulled by the node, simulator boot.
- Docs: on EDA 26.8+ a node must carry the label `eda.nokia.com/security-profile: managed` (or match another NodeSecurityProfile) or EDA never mints its bootstrap certificate and an SR OS simulator exits before booting; the generated NodeProfile examples say so. Talos registry-mirror patch documented as a strategic-merge YAML patch (JSON-6902 patches are refused on multi-document machine configs).
