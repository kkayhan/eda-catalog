# Imagemanager Application

-{{% import 'icons.html' as icons %}}-

| <nbsp> {: .hide-th } |                                         |
| -------------------- |-----------------------------------------|
| **Group/Version**    | -{{ app_group }}-/-{{ app_api_version }}-   |
| **Supported OS**     | -{{ supported_os_versions() }}-  |
| **Catalog**          | [edacommunity/catalog/imagemanager ][manifest] |
| **Source Code**      | [kkayhan/edaapp_ImageManager][src]      |

[manifest]: https://github.com/kkayhan/edaapp_ImageManager/tree/main/apps/imagemanager.eda.edacommunity.com
[src]: https://github.com/kkayhan/edaapp_ImageManager

Upload Nokia NOS images as vendor **`.zip`** files through a web page — the type is detected
automatically. Three kinds are supported:

* **SR Linux** and **SR OS 7750 (TiMOS) hardware** images become EDA **Artifacts**: each upload
  is stored in‑cluster and the built‑in artifact server (`eda-asvr`) downloads, md5‑validates,
  and re‑hosts the file(s) for Zero‑Touch Provisioning and software upgrades. The matching md5
  and YANG schema profile are handled for you. No external file server, no hand‑written
  `Artifact` resources.
* **SR‑SIM** (the SR OS *simulator* for EDA's Digital Twin) is a **container image**: the app
  unpacks it and serves it from a built‑in OCI registry endpoint, and gives you a ready‑to‑paste
  sim **NodeProfile** (`containerImage`). The app makes the node able to pull it
  automatically — a bundled node‑agent DaemonSet configures the node's containerd
  (Talos immutable nodes need a one‑time machine‑config registry mirror instead).
* **Licenses (optional)** — **paste a Nokia simulator/node license key** into the upload dialog
  (extra spaces/labels are parsed out for you). The app stores it as a `license.key` **ConfigMap**
  in `eda-system` and wires `spec.license: <image>-license` into the generated NodeProfile. A sim
  boots without one; add a key only to unlock licensed scale/features.

After install, open the app UI (while signed into the EDA UI) at
`https://<your-eda-address>/core/httpproxy/v1/imagemanager/`, pick a vendor `.zip`, choose the
namespace, and upload. A status table shows each image move to `Available` (file images) or
`Ready` (SR‑SIM), with a per‑row **Details** popup giving the NodeProfile to use.

The application provides the following components:

/// tab | Resources

<div class="grid" markdown>
<div markdown>

* **ImageManagerConfig** — cluster‑scoped settings (default namespace/repo, max upload size) and live status.

</div>
</div>
///
