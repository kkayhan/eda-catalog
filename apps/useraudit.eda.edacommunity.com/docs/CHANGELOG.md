# Changelog

All notable changes to the EDA User Audit app are documented here.

Versions follow the EDA release the app is built and validated against, with
an incrementing build suffix: `v<eda-release>-<n>` (e.g. `v26.4.3-1`,
`v26.4.3-2`). When the target EDA release changes, the suffix restarts at `-1`.

## v26.4.1-12

Ships the **`useraudit-reader`** EDA `ClusterRole` with the app, so a machine
account can read the audit log without being an administrator.

v26.4.1-11 locked the endpoint down to the `system-administrator` group. That is
correct but blunt: the typical reader is not a person but a machine — an SFTP
relay host, a SIEM collector — and giving that a full administrator identity is
exactly what an audit trail should not require. The role grants read on
`/core/httpproxy/v1/useraudit` and nothing else: no `resourceRules` (no EDA
resource of any kind) and no `tableRules` (no EDB/EQL access). It installs with
the app and is removed on uninstall.

The **group and the user are deliberately not shipped, and cannot be**: EDA keeps
users and groups in Keycloak, not Kubernetes, so there is no resource for an app
to install. An administrator creates them once, and **chooses the password** — no
credential is generated, defaulted, or stored by the app, and nothing in the app
depends on that password. EDA also attaches roles to *groups* rather than to
users, so the setup is always: create a group holding `useraudit-reader`, create
the user, add it to the group.

Recommended setup, matching the relay pattern in the README:

1. Install (or upgrade to) this version — the role appears automatically.
2. Create a group, e.g. `useraudit-readers`, and assign it the `useraudit-reader`
   role.
3. Create user `useraudit-readonly` with a password of your choosing, and add it
   to that group.
4. Give those credentials to the collector host, which pulls with
   `logs/pull-audit-logs.sh` and serves its local copy over its own SFTP.

Resulting access to the endpoint is unchanged in shape, just wider by one
deliberate seat: no token `400`, any other authenticated user `403`, members of
`system-administrator` `200`, members of a group holding `useraudit-reader` `200`.

No change to event collection, the log format, the CRD, app settings, or the
controller's behaviour.

## v26.4.1-11

**Security fix — the audit log endpoint now requires an EDA login, and only
administrators are allowed in.**

Up to v26.4.1-10 the app's `HttpProxy` used `authType: atDestination`. That name is
misleading: it means the EDA API server authenticates *nothing* and forwards the
request as-is, leaving it to the backend. This app's file server had no
authentication of its own, so **the entire audit log — usernames, source IPs, and
full configuration diffs — was readable by anyone who could reach the EDA
address, with no credentials at all.** Verified against a live cluster: an
unauthenticated `curl` of `/core/httpproxy/v1/useraudit/logs/<file>` returned
HTTP 200 and the complete log.

The proxy now uses `authType: inApiServer`, which makes the EDA API server the
enforcement point for both checks:

| Caller | Before | Now |
|--------|--------|-----|
| No token | `200` + full log | `400` — rejected as unauthenticated |
| Valid token, non-administrator | `200` + full log | `403` — authenticated, not authorized |
| Valid token, `system-administrator` | `200` | `200` |

Authorization is EDA's own RBAC, not app code. Reaching
`/core/httpproxy/v1/useraudit/**` requires a role carrying a **URL rule** for that
path; the only role shipped with one is the default `system-administrator`
`ClusterRole` (`/**`, `readWrite`), and roles are granted through user groups —
so access means membership of the `system-administrator` group. Because EDA rules
are additive with implicit deny, an operator can grant another group access by
creating a `ClusterRole` with a narrow URL rule, without touching the app. Both
behaviours were confirmed live by adding and removing a test user from the group.

Consequences to be aware of when upgrading:

- **Existing collectors will break** until they send an `Authorization: Bearer`
  header. Anything scraping the endpoint anonymously — cron jobs, SIEM pollers,
  monitoring checks on `/healthz` — now gets HTTP 400.
- **Browsers cannot open the URL.** EDA accepts the token only in a header (not a
  cookie, not a query parameter), so the URL returns HTTP 400 rather than a login
  page. Use `curl`, `logs/pull-audit-logs.sh`, or a collector that sets headers.
- **The app can no longer see who reads the logs.** `inApiServer` strips the token
  before forwarding, so downloads are not recorded in the audit files themselves;
  that visibility lives in EDA and Keycloak.
- Kubernetes health probes are unaffected — they hit the pod directly, not the
  proxy.

`logs/pull-audit-logs.sh` now performs the full EDA authentication flow: it locates
Keycloak (handling both the `/core/httpproxy/v1/keycloak` and
`/core/proxy/v1/identity` layouts), optionally fetches the `eda` client secret,
exchanges credentials for a token, re-acquires it if it expires mid-run, and
explains 400/401/403 in terms of what to fix. Credentials come from the
environment (`EDA_USERNAME`, `EDA_PASSWORD`, optionally `EDA_CLIENT_SECRET` or a
ready-made `EDA_TOKEN`). It remains pure `bash` + `curl`.

The docs now also cover the recommended setup for log collectors: a **dedicated
account scoped to this endpoint alone** rather than an administrator. Membership
of `system-administrator` cannot be made read-only — that group exists to carry a
role granting `resourceRules: * readWrite` and `urlRules: /** readWrite`, so a
"read-only member" is a contradiction. The README walks through creating a
`useraudit-reader` ClusterRole whose only permission is a URL rule for
`/core/httpproxy/v1/useraudit`, a group carrying it, and a user in that group —
including the five `400`s the admin API returns if you try to combine those steps.

Net effect on the endpoint: no token `400`, any other authenticated user `403`,
administrators and members of the reader group `200`.

No change to event collection, the log format, the CRD, or app settings.

## v26.4.1-10

Back to **HTTP(S) only** — the in-pod SFTP sidecar introduced in v26.4.1-6 is removed.

Why: SFTP delivery belongs outside the cluster. The sidecar meant the app shipped an
SSH daemon, generated host keys, published a second Service on the EDA VIP, and carried
a default `readonly`/`readonly` credential — a lot of surface, and all of it inside the
audited system. Pulling the logs over the existing HTTPS endpoint onto a separate host,
and serving SFTP from there, is simpler and puts the audit copy **off-cluster** so it
survives loss of the cluster. Ownership of the SFTP account, password policy, and
retention moves to whoever owns that host instead of being baked into an app manifest.

Removed:
- Container `useraudit-sftp` and the `useraudit-sftp` image (`build/sftp/` deleted).
- `manifests/sftp-service.yaml` (the `LoadBalancer` Service on port 22522) and its
  manifest component entry.
- App settings `sftpPort` and `sftpServiceType`.
- CRD status field `status.sftpEndpoint`, and `k8s.read_service()`, now unused.

Unchanged: everything about how events are collected and written. The HTTP API
(`/logs`, `/logs/<file>`, `/healthz`) and the daily-file format are exactly as before.

Upgrading from v26.4.1-6…-9: applying the new manifests drops the pod to a single
container and removes the SFTP Service. The `useraudit-sftp` Secret is *not* deleted by
the upgrade — remove it by hand once nothing depends on it:
`kubectl -n eda-system delete secret useraudit-sftp`. Audit data on the PVC is untouched.
See the README for the recommended external relay pattern.

## v26.4.1-9

Audit-integrity release — fixes from an adversarial multi-agent code review, each
reproduced against a live EDA 26.4.1 cluster. No new features, no CRD/app-settings
changes, no infrastructure changes: upgrade by installing the app as usual.

- **Fixed: a transaction caught mid-execution was permanently recorded as "Failed
  transaction attempt, no changes were made" — and its real changes were never
  audited.** EDA publishes a transaction summary while it is still `queued`/`running`
  (measured: ~4 seconds before it flips to `complete`), and anything that was not
  `complete` + `success` fell into the failure branch while the watermark advanced past
  it. The audit log therefore asserted the opposite of what happened, and the actual
  configuration change became unauditable. The poll loop now treats `complete` as the
  only terminal state (per the core API's `TransactionState`), defers an in-flight
  transaction to the next cycle without writing a line or moving the watermark, and
  picks it up with its full change detail once it settles. Genuine failures
  (`complete` + `success=false`) are still recorded as failures, unchanged. A
  transaction that never completes is skipped after one hour with an explicit
  "still in state X" record, so one wedged transaction can never stall auditing.
- **Fixed: purged transactions fabricated duplicate audit records.** The per-id
  summary endpoint *rounds up* — an id that no longer exists (EDA prunes old
  transactions) answers HTTP 200 with the **next surviving** transaction's record,
  under a different id. The scan trusted the id it asked for, so after any gap it wrote
  one bogus record per purged id, each carrying a later transaction's user, timestamp
  and changes. The scan now believes the id the API returns and skips the gap.
- **Fixed: an authentication outage was silently read as "this transaction does not
  exist".** Token acquisition can itself fail with HTTP 404 (Keycloak briefly
  unrouted, or `keycloak-admin-secret` momentarily absent). That 404 propagated
  indistinguishably from the endpoint's own 404, so the poll loop skipped the id,
  advanced the watermark past it and reported healthy. Token failures now raise a
  distinct `AuthError`, which fails the cycle loudly and leaves the watermark untouched
  for a clean retry.
- **Fixed: Keycloak events were silently dropped above 500 per poll.** Both event
  queries fetched only the newest 500 with no paging and no watermark anchor, then
  advanced the watermark to the newest event seen — so any burst larger than the
  window (a login storm, or a backlog after an outage) lost precisely the events an
  auditor most wants, with no error. Both feeds now page backwards until they reach
  the persisted watermark, and log a warning if a hard ceiling is ever reached.
- **Fixed: privilege changes were never audited, and one invalid filter value disabled
  server-side filtering entirely.** `USER_FEDERATION` is not a member of Keycloak's
  `ResourceType` enum, so Keycloak rejected the whole admin-events query with HTTP 500
  (the cause of the 500 documented in v26.4.1-2) and every poll refetched all resource
  types. The invalid value is gone, and `REALM_ROLE_MAPPING`, `CLIENT_ROLE_MAPPING` and
  `GROUP_MEMBERSHIP` are now collected and formatted — so role grants/revocations and
  group-membership changes appear in the audit log with the actor, the subject and the
  role or group involved.

## v26.4.1-8

Reliability release — fixes from a full code review. No new features, no
CRD/app-settings changes.

- **Fixed: log cleanup silently stopped after the last day of a month.** The
  daily 3 AM scheduler computed "tomorrow" in a way that crashes on month
  boundaries (e.g. July 31), permanently killing the cleanup thread until the
  pod restarted. Retention and the 90%-disk guard now keep running, and the
  cleanup thread additionally survives any unexpected error.
- **Fixed: backend outages were invisible in health.** A full EDA API or
  Keycloak outage previously still reported `health: ok / All systems
  operational` — connection errors were misread as "no new transactions" / "no
  new events". Outages now surface as `degraded`/`error` with the failing
  subsystem named, and the stale-poll detector works again. Watermarks are
  untouched during an outage, so nothing is lost or duplicated on recovery.
- **Fixed: a transaction with an unparseable timestamp could permanently stall
  transaction auditing** (the poll cycle failed before the watermark advanced
  past it, then re-hit it every cycle). Unparseable timestamps now degrade to
  the observation time and processing continues.
- **Fixed: a transient API failure during first-run initialization could reset
  the transaction watermark to 0**, causing the next healthy cycle to replay the
  entire transaction history into the audit log. First-run init now retries
  instead.
- **Hardened against transient Kubernetes API errors** (state ConfigMap reads,
  CR reads/status updates): the controller now skips the cycle and retries
  instead of crashing, and a corrupt state ConfigMap can no longer crash-loop
  the pod (cache fields self-heal; corrupt watermarks are reported clearly).
- **Hardened against a full log volume:** audit-log write failures no longer
  crash the controller; they hold the watermarks (so no events are dropped),
  and report `error` health with the reason in the CRD status.
- **Fixed: watermark updates could be lost when the poll loop and the cleanup
  thread wrote state simultaneously** (a rare race causing duplicate audit
  lines the next cycle). State writes are now serialized.
- **Audit accuracy:** Keycloak realm updates are now logged as "Realm settings
  have been modified." instead of the misleading fixed text "Password policy
  has been modified." (which every fresh install triggered once via the app's
  own event enablement).
- Minor: `/logs/` file download now serves only `.log` files; internal files
  (e.g. `.healthz.json`) return 404. Removed a wasted per-transaction API call
  for resources without a namespace.

## v26.4.1-7

- Log files are now written **per day** (`EDA-user-events-YYYY-MM-DD.log`) instead
  of per month. Retention is still configured in months (`retentionMonths`), and
  pre-existing monthly files still age out correctly.
- SFTP: the default login is now **user `readonly`, password `readonly`** (was a
  random-generated password for user `audit`). The password still lives in the
  `useraudit-sftp` Secret and can be changed there; the new value is applied on the
  next controller restart.
- Docs: SFTP examples now use `-o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null`
  so clients don't persist the endpoint's host key — a fresh app reinstall can change
  that identity, and not pinning it avoids a painful `known_hosts` cleanup later.

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
