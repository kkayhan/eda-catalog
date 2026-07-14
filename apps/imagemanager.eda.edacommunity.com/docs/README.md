# EDA Image Manager

Upload Nokia NOS images as vendor **`.zip`** files through a web page — the type is detected
automatically. **SR Linux** and **SR OS 7750 (TiMOS) hardware** images become EDA **Artifacts**
that the built‑in artifact server (`eda-asvr`) downloads, md5‑validates, and re‑hosts for ZTP
and software upgrades. **SR‑SIM** (the SR OS *simulator* for EDA's Digital Twin) is a container
image: the app serves it from a built‑in OCI registry and gives you a sim **NodeProfile**
(`containerImage`). The md5 and YANG schema profile are handled for you. You can also **paste a
Nokia license key** with any image (extra spaces/labels are parsed out) — it's stored as a
`license.key` ConfigMap in `eda-system` and referenced from the NodeProfile's `spec.license`
(a sim boots without one).

EDA's artifact server uses a pull model that integrates with an organization's central
image store or data lake — the right fit for production. This app is the lab‑friendly
alternative: it stages uploads in‑cluster and creates the `Artifact`(s) for you, so you
don't need an external file server or hand‑written resources just to try a NOS image.

## How to use

1. Install from the EDA App Store.
2. Open the app UI (logged into the EDA UI):
   `https://<your-eda-address>/core/httpproxy/v1/imagemanager/`
3. Pick a vendor `.zip` (SR Linux, SR OS 7750 TiMOS, or SR‑SIM) — the type and the packaged
   md5 are detected automatically. The image name is auto‑filled (lowercase, e.g.
   `srlinux-<version>` / `sros-<version>` / `srsim-<version>`); choose the namespace and upload.
4. Watch the status table: file images move `InProgress → Available`; SR‑SIM shows `Ready`.
   Each row's **Details** popup gives the copy‑paste **NodeProfile** (a `spec.images` snippet
   for file images, or a `containerImage` sim NodeProfile for SR‑SIM) and a
   **delete** action.

## Configuration

Edit the cluster‑scoped `ImageManagerConfig` named `default` (GUI: **Image Manager → Config**):
default artifact namespace (`eda`), default repo (`images`), and max upload size (`4096` MiB).
The app keeps each uploaded file as the durable origin `eda-asvr` pulls from (it re‑pulls on
restart), so files are never auto‑purged — they are removed only when you delete the Artifact.

See the repository README for full details.
