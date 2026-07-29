# EDA User Audit

Automatically logs all EDA transactions and Keycloak authentication events into daily audit log files (`EDA-user-events-YYYY-MM-DD.log`). Provides a read-only HTTP API for viewing and downloading logs.

## What It Logs

- **EDA Transactions**: Every configuration change -- who changed what, when, from which IP, with a flattened diff of the config changes
- **Keycloak Login/Logout Events**: GUI sign-in and sign-out events with username and source IP
- **Keycloak Admin Events**: User/group/client-role/realm modifications made through Keycloak

### Sample Log Output

```
2026-04-16T12:20:22 UTC | Event=Transaction-99 | User=admin | IPADDR=10.0.0.5 | Modified=Fabric | Namespace=default | Fabric resource named fabric1 has been updated.
   interface-ethernet/ethernet-1-1/admin-state: enable -> disable

2026-04-16T12:22:40 UTC | Event=EDA-Login | User=admin | IPADDR=10.244.0.27 | The user signed-in to the EDA GUI.
```

## Prerequisites

- EDA v25.12.x or later
- Access to `ghcr.io` from the EDA cluster nodes

## Installation

### Step 1: Add the Catalog

Apply this Catalog CR to register the app in your EDA App Store:

```yaml
apiVersion: appstore.eda.nokia.com/v1
kind: Catalog
metadata:
  name: kkayhan-catalog
  namespace: eda-system
spec:
  enabled: true
  remoteType: git
  remoteURL: https://github.com/kkayhan/eda-catalog.git
  refreshInterval: 180
  title: kkayhan community catalog
```

```bash
kubectl apply -f catalog.yaml
```

### Step 2: Install from the App Store

Open the EDA GUI, navigate to the **App Store**. The **EDA User Audit** will appear under the *Monitoring* category. Click **Install**.

Or install via CLI:

```yaml
apiVersion: appstore.eda.nokia.com/v1
kind: AppInstaller
metadata:
  name: install-useraudit
  namespace: eda-system
spec:
  operation: install
  dryRun: false
  apps:
    - appId: useraudit.eda.edacommunity.com
      catalog: kkayhan-catalog
      version:
        type: semver
        value: "v26.4.1-8"
```

```bash
kubectl apply -f install.yaml
```

### Step 3: Post-Install Setup

Create the default configuration:

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: useraudit.eda.edacommunity.com/v1alpha1
kind: UserAuditConfig
metadata:
  name: default
spec:
  pollIntervalSeconds: 300
  retentionMonths: 0
EOF
```

### Step 4: Verify

```bash
# Check the pod is running
kubectl -n eda-system get pods -l eda.nokia.com/app=eda-useraudit

# Check health (needs an EDA token -- see Access control below)
curl -sk -H "Authorization: Bearer $TOKEN" \
  https://<eda-address>/core/httpproxy/v1/useraudit/healthz
```

## Usage

### Access control

The log endpoint is protected by EDA itself. The app's `HttpProxy` uses
`authType: inApiServer`, so the **EDA API server** authenticates and authorizes
every request before it reaches the app:

| Caller | Result |
|--------|--------|
| No bearer token | `HTTP 400` — rejected as unauthenticated |
| Valid token, not a `system-administrator` | `HTTP 403` — authenticated but not authorized |
| Valid token, member of `system-administrator` | `HTTP 200` |

Authorization uses EDA RBAC **URL rules**. Reaching
`/core/httpproxy/v1/useraudit/**` requires a role with a URL rule covering that
path, and the only role shipped with one is the default `system-administrator`
`ClusterRole` (`/**`, `readWrite`). Roles are granted through user groups, so
access means membership of the **`system-administrator`** user group.

To let another group read the audit log, create a `ClusterRole` with a URL rule
covering the endpoint and assign it to that group — the app itself needs no
change:

```yaml
apiVersion: core.eda.nokia.com/v1
kind: ClusterRole
metadata:
  name: audit-log-reader
  namespace: eda-system
spec:
  description: Read-only access to the EDA User Audit log endpoint.
  urlRules:
    - path: /core/httpproxy/v1/useraudit/**
      permissions: read
```

### Recommended: a dedicated read-only account for collectors

Do **not** put a log-collector account in the `system-administrator` group. That
group exists to carry the `system-administrator` ClusterRole, which grants
`resourceRules: * readWrite` and `urlRules: /** readWrite` — membership *is* full
write access to EDA, and there is no way to attenuate it per user. A cron job
that only pulls files should not hold it.

Instead give the collector its own group and user, holding the
**`useraudit-reader`** role that this app installs for you.

That role grants exactly one thing — read access to
`/core/httpproxy/v1/useraudit` — with no `resourceRules` and no `tableRules`, so
the account can reach no EDA resource and no EDB table. It is installed with the
app and removed when the app is uninstalled.

The group and the user are **not** shipped and cannot be: EDA keeps users and
groups in Keycloak, not Kubernetes, so there is no resource for the app to
install. An administrator creates them once, and **chooses the password** — the
app never sees, stores, or defaults it.

Note that EDA attaches roles to **groups**, never directly to users. Granting
`useraudit-readonly` this role therefore always means "create a group holding the
role, and put the user in it"; there is no user-level assignment.

Using EDA's admin API (`$EDA` and `$TOKEN` as above, from an administrator):

```bash
# 1. A group, then attach the shipped role (roles cannot be set when creating a group).
curl -sk -X POST -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
  "$EDA/core/admin/groups" -d '{"name":"useraudit-readers",
    "description":"Members may read the User Audit log endpoint only."}'
GUUID=$(curl -sk -H "Authorization: Bearer $TOKEN" "$EDA/core/admin/groups" \
  | jq -r '.[]|select(.name=="useraudit-readers").uuid')
curl -sk -X POST -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
  "$EDA/core/admin/groups/$GUUID/roles" -d '["useraudit-reader"]'

# 2. A user, then group membership, then a password — each is its own call.
#    Choose your own password here; nothing about it comes from the app.
curl -sk -X POST -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
  "$EDA/core/admin/users" -d '{"username":"useraudit-readonly",
    "email":"useraudit-readonly@eda.local","firstName":"UserAudit",
    "lastName":"Reader","enabled":true}'
UUUID=$(curl -sk -H "Authorization: Bearer $TOKEN" "$EDA/core/admin/users" \
  | jq -r '.[]|select(.username=="useraudit-readonly").uuid')
curl -sk -X POST -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
  "$EDA/core/admin/users/$UUUID/groups" -d "[\"$GUUID\"]"
curl -sk -X PUT  -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
  "$EDA/core/admin/users/$UUUID/resetpassword" -d '{"value":"<password>","temporary":false}'
```

The same thing is available in the UI under **System Administration → User
Management** — create the group, assign it the `useraudit-reader` role, then
create the user and add it to the group. Five constraints in the API are worth
knowing, since each returns a `400`: a group cannot be created with `roles`
(attach them afterwards), a user cannot be created with `groups` or `password`
(both are separate calls), `email`/`firstName`/`lastName` are all mandatory, and
Keycloak's name validator rejects punctuation such as parentheses in first/last
names.

Then point the collector at the new account — no Keycloak admin credentials
needed on the collector host, just the `eda` client secret:

```bash
EDA_USERNAME=useraudit-readonly EDA_PASSWORD=... EDA_CLIENT_SECRET=... \
  ./pull-audit-logs.sh https://<eda-address> /var/audit-archive
```

This is the intended shape for the **SFTP relay** described below: the relay host
holds the `useraudit-readonly` credentials, pulls with the script on a timer, and
serves its local copy over its own SFTP. The relay never needs an EDA
administrator account, and the audit copy ends up off-cluster.

What that account ends up with, verified on EDA 26.4.1:

| request | result |
|---------|--------|
| the User Audit endpoint (`/healthz`, `/logs`, `/logs/<file>`) | `200` |
| any EDA resource, e.g. `toponodes` | `403` |
| user/group administration | `403` |
| any write, anywhere — including resetting its own password | `403` |
| transaction *metadata* (`/core/transaction/v2/result/summary`) | `200` — id, timestamp, username, success |
| transaction content: input resources, node config diffs, execution detail | no content — `inputCrs: [], limitedAccess: true`, or `403` |

The transaction-metadata row is EDA's own behaviour, not a gap in the role: summaries
are not URL-rule gated, while everything with configuration content in it is gated by
resource rules the account does not have. The visible fields are the same ones the
audit log already contains, so the account learns nothing extra.

Note that EDA strips the token before forwarding, so the app cannot see who
called it and does not record log downloads. Note also that a browser cannot open
the URL directly: EDA accepts the token only in an `Authorization` header, so
pasting the URL into a browser returns `HTTP 400`, not a login page. Use `curl`,
the helper script, or a collector that can set a header.

### Getting a token

Standard EDA API authentication — exchange your EDA username and password for a
token, using the Keycloak client secret of the `eda` client (ask your EDA
administrator, or read it from Keycloak):

```bash
EDA=https://<eda-address>
TOKEN=$(curl -sk "$EDA/core/httpproxy/v1/keycloak/realms/eda/protocol/openid-connect/token" \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  --data-urlencode 'grant_type=password' \
  --data-urlencode 'client_id=eda' \
  --data-urlencode 'scope=openid' \
  --data-urlencode "client_secret=$EDA_CLIENT_SECRET" \
  --data-urlencode "username=$EDA_USERNAME" \
  --data-urlencode "password=$EDA_PASSWORD" | jq -r .access_token)
```

Tokens last about 5 minutes. On deployments that route Keycloak at
`/core/proxy/v1/identity` rather than `/core/httpproxy/v1/keycloak`, use that
path instead.

### Log Endpoints

All endpoints are accessible via the EDA HttpProxy:

```
https://<eda-address>/core/httpproxy/v1/useraudit/
```

| Endpoint | Description |
|----------|-------------|
| `/healthz` | Health status and last poll time |
| `/logs/` | List all log files with sizes and timestamps (JSON) |
| `/logs/<filename>` | Download a specific log file (plain text) |

**Examples:**

```bash
EDA=https://<eda-address>
BASE=$EDA/core/httpproxy/v1/useraudit
AUTH="Authorization: Bearer $TOKEN"

# Health check
curl -sk -H "$AUTH" $BASE/healthz

# List log files
curl -sk -H "$AUTH" $BASE/logs/

# Download a specific day's log
curl -sk -H "$AUTH" $BASE/logs/EDA-user-events-2026-05-04.log
```

### Need SFTP? Relay it from outside the cluster

The app serves **HTTP(S) only**. An in-pod SFTP sidecar existed in v26.4.1-6 …
v26.4.1-9 and was **removed in v26.4.1-10**: it required an SSH daemon, host
keys, a second Service on the EDA VIP, and a default password shipped inside the
app. Relaying from outside is both simpler and safer, and it keeps the audit
copy off-cluster so it survives loss of the cluster.

Recommended pattern — a cron job on any Linux host that can reach EDA over
HTTPS, serving its own SFTP:

1. Acquire a token as above, for a user in the `system-administrator` group.
   Tokens expire in ~5 minutes, so fetch one per run rather than caching it.
2. `GET $BASE/logs` to list; the JSON gives `name` and `size_bytes`.
3. Re-fetch only files whose `size_bytes` differs from the local copy. The
   current day's file grows; finished days never change again.
4. Download to a temp file and `mv` it into place, so a collector never reads a
   half-written file.
5. Serve that directory with the host's own SFTP (OpenSSH `internal-sftp` with
   `ChrootDirectory` works well), where the account, password policy, and
   retention are owned by the host, not by an app manifest.

`logs/pull-audit-logs.sh` in the source repo implements steps 1–4 in pure
`bash` + `curl`, credentials supplied through the environment.

### CRD Status

```bash
kubectl get userauditconfig default -o yaml
```

Reports: `health`, `subsystems` (edaApi, keycloakEvents), `lastPollTime`, `lastTransactionId`, `transactionsProcessed`, `kcEventsProcessed`, `logFiles`, `version`.

## Configuration

### UserAuditConfig CRD

```bash
kubectl edit userauditconfig default
```

| Field | Default | Range | Description |
|-------|---------|-------|-------------|
| `pollIntervalSeconds` | 300 | 60-3600 | Polling interval in seconds |
| `retentionMonths` | 0 | 0+ | Months of logs to keep (0 = unlimited) |

### App Settings

Adjustable via the EDA App Store settings:

| Setting | Default | Description |
|---------|---------|-------------|
| `controllerCpuLimit` | 200m | CPU limit |
| `controllerMemoryLimit` | 128Mi | Memory limit |
| `logStorageSize` | 500Mi | PVC size for log storage |

## Uninstalling

Via CLI:

```yaml
apiVersion: appstore.eda.nokia.com/v1
kind: AppInstaller
metadata:
  name: uninstall-useraudit
  namespace: eda-system
spec:
  operation: delete
  dryRun: false
  apps:
    - appId: useraudit.eda.edacommunity.com
      catalog: kkayhan-catalog
```

Or remove through the EDA App Store UI.

## Source-IP attribution

`IPADDR=` in transaction logs is whatever Keycloak stored in its login event for the user who made the change. For that to be the real client IP (not a CNI gateway like `10.244.0.1` or a node IP), the cluster must be deployed per Nokia EDA's documented production topology.

What the app already does, no operator action needed:

- Filters Keycloak events to `clientId="auth"` (the browser code-flow) so the controller's own admin token-refresh logins (`clientId="eda"`, source IP = useraudit Pod IP) don't pollute attribution.

What the cluster operator must do, once per cluster:

1. Run an Ingress controller in front of `eda-api` and let it terminate TLS so it can add `X-Forwarded-For` with the real client IP. EDA's docs cover this in *Software Install → Exposing the Nokia EDA UI/API*. A ready-to-apply Ingress manifest ships with the playground at `eda-kpt/eda-external-packages/eda-api-ingress-https/`. The Ingress controller's own Service should have `externalTrafficPolicy: Local` so kube-proxy doesn't SNAT external clients before they reach it.
2. Set `EngineConfig.spec.cluster.external.proxyMode` to `XForward` (or `Forward`). Default is `None`, which causes the api-server to drop incoming `X-Forwarded-*` headers. EDA configures Keycloak with the matching `--proxy-headers` flag automatically. See *User Guide → Security → Platform Security → Proxy forward headers*.
3. Make sure `eda-api` does not also claim the LoadBalancer IP that the Ingress now owns. On EDA 26.4.1 the cleanest way is to set `autoAssign: false` on the MetalLB `IPAddressPool` — `eda-api` Service stays type `LoadBalancer` (its `EngineConfig.spec.api.serviceType` field, although exposed by the CRD enum, is not safe to set to `ClusterIP` on this release: the api-server reconciler trips a K8s validation error on `allocateLoadBalancerNodePorts`). With `autoAssign: false`, the Ingress controller claims the VIP via an explicit `loadBalancerIP` request, and `eda-api` sits with a Pending external IP but is still fully reachable in-cluster via its ClusterIP — which is all the Ingress backend needs.

Without those three pieces, Keycloak records the IP that kube-proxy or the cluster's network plumbing rewrote the request to — typically the CNI gateway or a node IP — and useraudit faithfully attributes that.

Verification: after a UI login from your client, fetch
`https://<eda-vip>/core/httpproxy/v1/useraudit/logs/EDA-user-events-<YYYY-MM>.log`
and the latest `Transaction-<n>` line should show the client's real IP.

## Troubleshooting

**Pod in ImagePullBackOff**: Verify the cluster can reach `ghcr.io`. The images are public -- no authentication needed.

**Health shows degraded/error**: Check `kubectl get userauditconfig default -o yaml` and pod logs.

**No logs**: The controller auto-discovers the latest transaction ID on first start and only logs new events going forward. Make a change in EDA and wait for the next poll cycle.

**HttpProxy 404**: Verify the HttpProxy CR exists: `kubectl get httpproxies.core.eda.nokia.com useraudit`
