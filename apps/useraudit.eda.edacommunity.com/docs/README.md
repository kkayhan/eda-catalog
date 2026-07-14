# EDA User Audit

Automatically logs all EDA transactions and Keycloak authentication events into monthly audit log files. Provides a read-only HTTP API for viewing and downloading logs.

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
  name: community-apps
  namespace: eda-system
spec:
  remoteURL: https://github.com/kkayhan/edaapp_UserAudit.git
  skipTLSVerify: false
  title: Community EDA Apps
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
      catalog: community-apps
      version:
        type: semver
        value: "v0.7.0"
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

# Check health
curl -sk https://<eda-address>/core/httpproxy/v1/useraudit/healthz
```

## Usage

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

# Health check
curl -sk $BASE/healthz

# List log files
curl -sk $BASE/logs/

# Download current month's log
curl -sk $BASE/logs/EDA-user-events-2026-05.log
```

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
      catalog: community-apps
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
