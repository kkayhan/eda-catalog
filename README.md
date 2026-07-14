# kkayhan EDA catalog

A [Nokia EDA](https://docs.eda.dev/) **app catalog** — a small collection of community
apps you can install into an EDA cluster straight from the EDA Store.

## Apps

| App | Group | What it does |
|---|---|---|
| **EDA TopoView** | `topoview.eda.edacommunity.com` | Auto-generates a live Grafana topology dashboard per fabric (border-leaf / spine / leaf) with per-interface telemetry. [repo](https://github.com/kkayhan/edaapp_grafana) |
| **EDA Image Manager** | `imagemanager.eda.edacommunity.com` | Web UI to upload network-OS images (SR Linux / SR OS) and auto-create the EDA Artifact CRs. [repo](https://github.com/kkayhan/edaapp_ImageManager) |
| **EDA User Audit** | `useraudit.eda.edacommunity.com` | Records and reports EDA user login / activity events. [repo](https://github.com/kkayhan/edaapp_UserAudit) |

## How to use this catalog

Register it on your EDA cluster with a `Catalog` CR, then install any app from the Store:

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
# the apps then appear in the EDA UI under Store
```

## Layout

Each app is a directory under `apps/<group>/` holding the app **manifest** (Store metadata),
its CRD(s), docs, and any install-time settings schema. A git tag `apps/<group>/<version>`
marks each published version. The actual installable content lives in the app's **bundle OCI
image** (referenced by `image:` in the manifest), published to GHCR.

## License

MIT — see [LICENSE](LICENSE).
