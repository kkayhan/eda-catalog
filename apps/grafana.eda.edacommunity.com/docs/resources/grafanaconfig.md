---
resource_name: GrafanaConfig
resource_name_plural: grafanaconfigs
resource_name_plural_title: Grafana Configs
resource_name_acronym: TV
crd_path: docs/grafana.eda.edacommunity.com/crds/grafana.eda.edacommunity.com_grafanaconfigs.yaml
icon: auto-crd
---

# Grafana Config

-{{% import 'icons.html' as icons %}}-

-{{ category(resource_name_plural) }}- → -{{ icons.circle(letter=resource_name_acronym, text=resource_name_plural_title) }}-

Cluster-scoped singleton (`default`) holding global tuning for Grafana and its live
status. All spec fields are optional; auto-discovery works without any configuration.

## Examples

/// tab | YAML

```yaml
-{{ include_snippet(resource_name) }}-
```

///

/// tab | `kubectl`

```bash
cat << 'EOF2' | kubectl apply -f -
-{{ include_snippet(resource_name) }}-
EOF2
```

///

## Custom Resource Definition

To browse the Custom Resource Definition go to [crd.eda.dev](https://crd.eda.dev/-{{ resource_name_plural }}-.-{{ app_group }}-/-{{ app_api_version }}-).

-{{ crd_viewer(crd_path, collapsed=False) }}-
