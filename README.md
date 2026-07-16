# kkayhan EDA catalog

A [Nokia EDA](https://docs.eda.dev/) **app catalog** — a small collection of community
apps you can install into an EDA cluster straight from the EDA Store.

This repository is the **single home for everything needed to install these apps**: the
catalog entries (Store metadata), the container images (under
`ghcr.io/kkayhan/eda-catalog/…`), and the offline **air-gap bundles** (attached to this
repo's GitHub Releases). Each app's **source code** lives in its own repository, linked
under each app below.

## Apps

### EDA Grafana — `grafana.eda.edacommunity.com`

Turns any EDA-managed data-center fabric into a **live Grafana topology map**,
automatically. It discovers every namespace that contains a fabric, draws the switch
topology by role (border-leaf on top, spines in the middle, leaves at the bottom), and
wires up live per-interface throughput and up/down state read **straight from EDA over
EQL** — no Prometheus, no external dependencies. Dashboards regenerate on their own as the
topology changes; users open a read-only, login-free Grafana, pick a fabric, and see it.

**Source code:** <https://github.com/kkayhan/edaapp_grafana>

### EDA Image Manager — `imagemanager.eda.edacommunity.com`

A lab-friendly way to get **network-OS images into EDA**. Upload an SR Linux, SR OS (7750
TiMOS), or SR-SIM simulator `.zip` through a web page and the app stores it in-cluster and
creates the matching EDA `Artifact` resources for you — no external file server, no
hand-written YAML — ready for Zero-Touch Provisioning, software upgrades, and the **Digital
Twin** simulator. The YANG schema profile and an optional simulator license key are handled
automatically.

**Source code:** <https://github.com/kkayhan/edaapp_ImageManager>

### EDA User Audit — `useraudit.eda.edacommunity.com`

Turns the EDA cluster into a **system-of-record for who did what**. Once installed it
continuously records every configuration change (with the user, timestamp, source IP, and a
human-readable per-device diff), every EDA sign-in/out, and every Keycloak admin change — to
monthly log files on a persistent volume, served read-only over HTTP. Built for compliance
archives, SIEM feeds, and change-management audits.

**Source code:** <https://github.com/kkayhan/edaapp_UserAudit>

## How to use this catalog

Register it on your EDA cluster with a **single** `Catalog` CR, then install any app from
the Store (in the EDA UI: **System Administration → APP Management → Catalogs → Create**,
paste the YAML, **Commit**):

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
# all three apps then appear in the EDA UI under Store
```

One catalog covers every app above — you don't add a separate entry per app.

## Layout

- `apps/<group>/` — each app's **manifest** (Store metadata), CRD(s), docs, and any
  install-time settings schema. Published by `edabuilder publish` from each app's source repo.
- Git tag `apps/<group>/<version>` marks each published version.
- **Container images** live under `ghcr.io/kkayhan/eda-catalog/<app>-{app,controller,…}`
  (the bundle OCI image is referenced by `image:` in each manifest).
- **Air-gap bundles** for offline install are attached to the GitHub **Release** on the
  matching `apps/<group>/<version>` tag (image archives + catalog git bundle + an
  `INSTALL-GUIDE.md`).

## License

MIT — see [LICENSE](LICENSE).
