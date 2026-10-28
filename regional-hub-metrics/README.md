# regional-hub-metrics: the regional hub's own metrics to the Global Hub (ACM-47259)

The regional hub (RH) keeps all its metrics, its own and its leaf clusters', in its own Thanos.
Its **own** platform metrics (its `local-cluster`) are also remote-written to the Global Hub (GH).
Leaf clusters (e.g. vm-cluster) stay regional and never get GH's URL or cert.

```
RH MCOA, placement rh-self   = [local-cluster] ─► RH Thanos + GH Thanos   {cluster=local-cluster, managed_hub=regional-hub}
RH MCOA, placement rh-leaves = [vm-cluster]    ─► RH Thanos only          {cluster=vm-cluster,    managed_hub=regional-hub}
GH MCOA, placement global-mcoa = [local-cluster] (excludes regional-hub)  {cluster=local-cluster, managed_hub=global-hub}
```

## Bootstrap (the only manual apply besides the prerequisites)
    KUBECONFIG=$HOME/.kube/acm-31440-gh oc apply -f regional-hub-metrics/apps/

| Application | Runs against | Folder | What it does |
|---|---|---|---|
| `sync-secret-to-regional-hub` | GH (in-cluster) | `sync-secret/` | Copies GH's metrics write cert to RH: Policy `acm47259-gh-write-cert` (on GH's local-cluster) owns ManifestWork `regional-hub/acm47259-gh-write-cert`, which creates Secret `open-cluster-management-observability/gh-observatorium-client` (GH client cert + GH server CA) on RH. Plus the Argo RBAC it needs |
| `sync-policy-to-regional-hub` | regional-hub (via GitOpsCluster) | `sync-policy/` | Policy `open-cluster-management-global-set/rh-metrics-to-global-hub` (ACM's namespace, already bound to clusterset `global`), enforced by RH's own governance on its local-cluster: adds the GH remoteWrite to RH's own collector and labels RH's series |
| `label-global-hub-metrics` | GH (in-cluster) | `label-global-hub/` | Policy `open-cluster-management-observability/label-global-hub-metrics`, enforced by GH's own governance on its local-cluster: adds `managed_hub=global-hub` to GH's own MCOA templates, so GH's local-cluster never mixes with RH's. Uses the `default` ClusterSetBinding from `sync-secret-to-regional-hub`. Plus the Argo RBAC it needs |

### How the RH Policy works
`musthave` appends array items and patches items with a matching `name`
(rhacm-docs `governance/config_policy_parameters.adoc`). The Policy adds only:
- on `mcoa-default-platform-metrics-collector-rh-self`: remoteWrite `global-hub`, `spec.secrets: [gh-observatorium-client]`, `externalLabels.managed_hub=regional-hub`;
- on the other three RH templates (rh-self user workloads, rh-leaves platform and user workloads): `externalLabels.managed_hub=regional-hub`.

MCOA's own `acm-observability` entry is not copied into Git. MCOA keeps other remoteWrite entries and listed secrets
(MCOA `agentssa.go`) and replicates `spec.secrets` into the agent namespace of the clusters that template targets
(`handler.go:276`), here RH itself only. If anything drops the `global-hub` entry, the Policy re-adds it.

### Secrets
Nothing secret is in Git. The ManifestWork's Secret data is rendered on GH from the MCO-managed secrets
`observability-controller-open-cluster-management.io-observability-signer-client-cert` (tls.crt, tls.key) and
`observability-server-ca-certs` (ca.crt) via `{{hub fromSecret}}`, so it follows GH's own cert renewal.
`{{hub fromSecret}}` only reads the Policy's own namespace, which is why that Policy lives in `open-cluster-management-observability`.

## Prerequisites (manual, not synced by Argo): one MCOA parent per cluster
The ClusterManagementAddOn placements are cluster setup. Re-apply them after any MCOA disable/enable, which recreates the CMA.

1. **GH: exclude regional-hub from GH's MCOA.** Otherwise both hubs write same-named PrometheusAgents onto RH and overwrite each other.
       KUBECONFIG=$HOME/.kube/acm-31440-gh oc apply -f regional-hub-metrics/prereq/placement-global-mcoa.yaml
       # decision must be exactly [local-cluster]:
       KUBECONFIG=$HOME/.kube/acm-31440-gh oc -n open-cluster-management-global-set get placementdecision \
         -l cluster.open-cluster-management.io/placement=global-mcoa -o jsonpath='{.items[*].status.decisions}'
       KUBECONFIG=$HOME/.kube/acm-31440-gh oc patch clustermanagementaddon multicluster-observability-addon --type=json \
         -p='[{"op":"test","path":"/spec/installStrategy/placements/0/name","value":"global"},{"op":"replace","path":"/spec/installStrategy/placements/0/name","value":"global-mcoa"}]'
   MCO then creates `mcoa-default-*-global-mcoa` templates; `label-global-hub/policy.yaml` targets those names.
2. **RH: split RH's MCOA into rh-self and rh-leaves** so only RH's own collector gets the extra remote write.
       KUBECONFIG=$HOME/.kube/acm-31440-rh oc apply -f regional-hub-metrics/prereq/rh-placements.yaml
       # decisions must be [local-cluster] and [vm-cluster]; back up the CMA (oc get ... -o json) first, then:
       KUBECONFIG=$HOME/.kube/acm-31440-rh oc patch clustermanagementaddon multicluster-observability-addon --type=json \
         --patch-file regional-hub-metrics/prereq/rh-cma-switch.json
   One patch replaces placement `global` with `rh-self` and `rh-leaves` (same configs), so neither ManagedClusterAddOn is deleted.
   MCO then generates the `-rh-self` and `-rh-leaves` templates.

## Verify
- GH Thanos: `count by (cluster, managed_hub) (up)` shows `{local-cluster, regional-hub}` and `{local-cluster, global-hub}`, and no `vm-cluster`.
- RH Thanos: `local-cluster` and `vm-cluster`, both `managed_hub=regional-hub`.
- RH: `oc get prometheusagent.monitoring.rhobs -n open-cluster-management-agent-addon platform-metrics-collector` has remoteWrite `[acm-observability, global-hub]`; vm-cluster's has only `acm-observability`.
- RH: `oc get policy -n local-cluster` shows `open-cluster-management-global-set.rh-metrics-to-global-hub` Compliant; on GH, `oc get policy -n local-cluster` shows `open-cluster-management-observability.label-global-hub-metrics` and `.acm47259-gh-write-cert` Compliant.

## Rollback
1. Delete the three Applications. Prune is off, so the objects stay.
2. On RH: delete Policy `open-cluster-management-global-set/rh-metrics-to-global-hub`, then remove the `global-hub` entry and `gh-observatorium-client` from the rh-self template.
3. On GH: delete Policy `open-cluster-management-observability/acm47259-gh-write-cert` and ManifestWork `regional-hub/acm47259-gh-write-cert`.
4. Prerequisites: patch each CMA back to a single `global` placement. On GH this brings the overwrite clash back.
