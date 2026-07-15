# Changelog

All notable changes to the EDA User Audit app are documented here.

Versions follow the EDA release the app is built and validated against, with
an incrementing build suffix: `v<eda-release>-<n>` (e.g. `v26.4.3-1`,
`v26.4.3-2`). When the target EDA release changes, the suffix restarts at `-1`.

## v26.4.1-2

- Fix the `HttpProxy` component missing `namespace: eda-system`. Store installs
  inject it, but a direct `kubectl apply` landed it in the `default` namespace,
  where eda-api doesn't route it, so `/core/httpproxy/v1/useraudit/…` returned 404.
- Quiet log noise introduced in v26.4.1-1: the transaction look-ahead scan probes
  not-yet-existing ids (expected 404s) — those are now logged at debug, not warning.
  Server errors (5xx) and token-request failures are still warned.
- Stop the per-poll Keycloak 500. EDA 26.4.1's Keycloak rejects the admin-events
  `resourceTypes` filter with HTTP 500; after the first rejection the app drops the
  filter and fetches all resource types (the same client-side filtering still applies),
  so admin events are collected without an error every cycle.

## v26.4.1-1

- **Re-baselined to target EDA 26.4.1** (validated on a live 26.4.1 cluster); the
  suffix restarts at `-1`. Code is backward-compatible with 26.4.3.
- Fix first-run Keycloak auth across EDA releases. The controller hardcoded the
  Keycloak base at `/core/httpproxy/v1/keycloak`; on some 26.4.1 deployments that
  path 404s (Keycloak is routed at `/core/proxy/v1/identity`), aborting init before
  any token is acquired. It now probes candidate bases once (unauthenticated
  `.well-known`) and pins the one that answers, so it works regardless of the
  cluster's Keycloak routing. Auth failures now log the exact URL + HTTP status.
- Carries forward the `?size=1` transaction-summary fix from v26.4.3-2.

## v26.4.3-2

- Fix first-run initialization against EDA **26.4.1**. `discover_current_watermark()`
  queried the transaction-summary list endpoint as `?page=0&size=1`, but the
  `page` parameter does not exist on 26.4.1 (its only required query param is
  `size`), so the request returned 404 and the audit loop never started. The call
  is now `?size=1` — honored across 26.4.x — and the watermark is taken as the
  max transaction id over the returned results (order-agnostic).

## v26.4.3-1

- Adopt the EDA-release-tied version scheme. This build targets and is
  validated against EDA **26.4.3**. Application code is unchanged from v0.8.0;
  this release re-baselines the version number to track the EDA release.
- Fix stale controller `VERSION` string (was `v0.7.1`; now matches the
  release tag).
- Bump project `builderVersion` to `v26.4.3` to match the toolbox.

## v0.8.0 and earlier

Pre-date the EDA-release-tied scheme. See git tags
`apps/useraudit.eda.edacommunity.com/v0.8.0`, `.../v0.7.1`, `.../v0.7.0`.
