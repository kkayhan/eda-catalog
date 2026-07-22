# Changelog

All notable changes to the EDA User Audit app are documented here.

Versions follow the EDA release the app is built and validated against, with
an incrementing build suffix: `v<eda-release>-<n>` (e.g. `v26.4.3-1`,
`v26.4.3-2`). When the target EDA release changes, the suffix restarts at `-1`.

## v26.4.1-6

- New: read-only SFTP endpoint for audit log collection, alongside the existing
  HTTP API. An OpenSSH sidecar (`useraudit-sftp` image) joins the controller pod,
  serving the log volume read-only to user `audit` — chroot-jailed, no shell,
  uploads/deletes refused at three layers (read-only mount, `internal-sftp -R`,
  chroot). Exposed on port 22522 by default via a `LoadBalancer` Service that
  joins EDA's shared MetalLB VIP (`metallb.universe.tf/allow-shared-ip`);
  service type and port are configurable at install time (new app settings
  `sftpPort`, `sftpServiceType`).
- Credentials: password authentication. The password is auto-generated on first
  start and persisted in the `useraudit-sftp` Secret together with the SSH host
  keys (stable host identity across restarts). Retrieve it with
  `kubectl -n eda-system get secret useraudit-sftp -o jsonpath='{.data.password}' | base64 -d`.
- CRD status now reports the live endpoint in `status.sftpEndpoint`.
- The sidecar populates its chroot jail with the `/dev` nodes and account files
  `internal-sftp` requires after chroot, so SFTP starts correctly on runtimes
  (e.g. Talos) whose containers don't pre-populate a usable `/dev`.

## v26.4.1-5

- Robustness hardening of the v26.4.1-4 stored-secret path (no new features). Four
  fixes, all validated on a live 26.4.1 cluster including an in-pod simulation of a
  cluster whose Keycloak is routed at the non-default base:
  - Run Keycloak base discovery on the stored-secret fast path too. In v26.4.1-4 the
    cached-secret path skipped discovery, so on a cluster where Keycloak is routed at
    `/core/proxy/v1/identity` (not the default `/core/httpproxy/v1/keycloak`) any
    restart would send the token request to the unprobed default and 404-loop
    forever. Discovery now runs before every token acquisition.
  - Build Keycloak admin URLs only after the base is pinned. They were computed
    before the first token call pinned the base, so the first admin request of the
    process used the default base.
  - Never pin a base from a failed probe. A transient eda-api outage at pod start no
    longer freezes a possibly-wrong base for the process lifetime — the base is
    pinned only on a successful probe and re-probed on the next call after a failure.
    The probe timeout is tightened so a full outage stays cheap.
  - Restore role-drift self-heal. A `403` on the EDA transaction API or a Keycloak
    admin endpoint now triggers a one-shot full re-provision that re-grants the
    realm and realm-management roles, so a service account whose roles were stripped
    recovers on the next poll instead of failing until manual intervention.

## v26.4.1-4

- Remove the runtime dependency on the Keycloak master-admin credential. After the
  one-time provisioning in v26.4.1-3, the controller now fetches the `eda-useraudit`
  client secret and persists it to a Kubernetes secret `eda-useraudit-client`; on
  every subsequent start it reads that stored secret and authenticates with
  `client_credentials` directly. The master admin (`keycloak-admin-secret`) is used
  only for first-run provisioning or self-heal — so the audit loop keeps working even
  if the master-admin password is later rotated or goes stale.
- Grant the service account the `realm-management` client roles `manage-realm`,
  `view-events`, and `view-users` at provisioning time, so a single service-account
  token drives both the EDA transaction API and the Keycloak admin event/user
  endpoints (the login/admin-event feeds) without a second credential.
- Uninstall now also removes three runtime-created objects that are not shipped as
  manifests: the `useraudit-state` ConfigMap, the `eda-useraudit-client` Secret, and
  the `eda-useraudit` Keycloak client.

## v26.4.1-3

- **Stop assuming any password.** v26.4.1-2 authenticated to the EDA API with an
  `eda`-realm password grant that read the `eda-realm-auth-secret` — an EDA bootstrap
  seed fixed at `admin/admin` and never re-synced — so the audit loop returned
  `HTTP 401 invalid_grant` on any cluster whose EDA admin password had been changed.
  The controller now self-provisions a dedicated confidential Keycloak service-account
  client `eda-useraudit` on first run (using the Keycloak master admin it already
  reads), grants its service account the `edarole_system-administrator` realm role,
  and authenticates via `client_credentials`. No password is read or assumed; the
  identity is dedicated, auditable, and revocable. RBAC no longer needs
  `eda-realm-auth-secret`.

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
