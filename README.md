# vertical-pod-autoscaler-extras

Extras for running the Kubernetes [Vertical Pod Autoscaler](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler).

## kube-state-metrics

kube-state-metrics removed the built-in `verticalpodautoscalers` collector in
[v2.9.0](https://github.com/kubernetes/kube-state-metrics/releases/tag/v2.9.0).
[kube-state-metrics/custom-resource-state.yaml](kube-state-metrics/custom-resource-state.yaml)
is a [Custom Resource State](https://github.com/kubernetes/kube-state-metrics/blob/main/docs/metrics/extend/customresourcestate-metrics.md)
config that re-creates those metrics under the same `kube_verticalpodautoscaler_*`
prefix and adds a few more.

### Metrics

| Metric | Type | Notes |
| --- | --- | --- |
| `kube_verticalpodautoscaler_annotations` | Info | `annotation_*` labels |
| `kube_verticalpodautoscaler_labels` | Info | `label_*` labels |
| `kube_verticalpodautoscaler_spec_updatepolicy_updatemode` | StateSet | `update_mode` label |
| `kube_verticalpodautoscaler_spec_updatepolicy_minreplicas` | Gauge | |
| `kube_verticalpodautoscaler_spec_updatepolicy_evictafteroomseconds` | Gauge | |
| `kube_verticalpodautoscaler_spec_resourcepolicy_container_policies_mode` | StateSet | `container`, `mode` labels |
| `kube_verticalpodautoscaler_spec_resourcepolicy_container_policies_controlledvalues` | StateSet | `container`, `controlled_values` labels |
| `kube_verticalpodautoscaler_spec_resourcepolicy_container_policies_{minallowed,maxallowed}_{cpu,memory}` | Gauge | `container`, `resource`, `unit` labels |
| `kube_verticalpodautoscaler_status_recommendation_containerrecommendations_{lowerbound,target,uncappedtarget,upperbound}_{cpu,memory}` | Gauge | `container`, `resource`, `unit` labels |
| `kube_verticalpodautoscaler_status_condition` | StateSet | `condition`, `status` labels |
| `kube_verticalpodautoscaler_status_observedgeneration` | Gauge | |

All metrics carry `namespace`, `verticalpodautoscaler`, `target_api_version`,
`target_kind` and `target_name` labels, plus the `customresource_group`,
`customresource_version` and `customresource_kind` labels that kube-state-metrics
always adds.

### Deploying

#### Plain manifests

Create a ConfigMap from the file and point kube-state-metrics at it:

```sh
kubectl -n kube-system create configmap kube-state-metrics-custom-resource-state \
  --from-file=config.yaml=kube-state-metrics/custom-resource-state.yaml
```

```yaml
args:
  - --custom-resource-state-config-file=/etc/customresourcestate/config.yaml
volumeMounts:
  - name: custom-resource-state
    mountPath: /etc/customresourcestate
    readOnly: true
volumes:
  - name: custom-resource-state
    configMap:
      name: kube-state-metrics-custom-resource-state
```

#### Helm chart

With the [kube-state-metrics chart](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-state-metrics),
put the contents of the file under `customResourceState.config`:

```yaml
customResourceState:
  enabled: true
  config:
    kind: CustomResourceStateMetrics
    spec:
      # ...contents of kube-state-metrics/custom-resource-state.yaml
```

### RBAC

kube-state-metrics needs `list` and `watch` on CustomResourceDefinitions (to
discover the served VPA versions) and on VerticalPodAutoscalers themselves. Add
the following to the kube-state-metrics `ClusterRole`:

```yaml
- apiGroups: ["apiextensions.k8s.io"]
  resources: ["customresourcedefinitions"]
  verbs: ["list", "watch"]
- apiGroups: ["autoscaling.k8s.io"]
  resources: ["verticalpodautoscalers"]
  verbs: ["list", "watch"]
```

The Helm chart adds the CustomResourceDefinitions rule itself when
`customResourceState.enabled` is `true`, so only the VPA rule is needed:

```yaml
rbac:
  extraRules:
    - apiGroups: ["autoscaling.k8s.io"]
      resources: ["verticalpodautoscalers"]
      verbs: ["list", "watch"]
```
