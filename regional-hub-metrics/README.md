# regional-hub-metrics: the regional hub's own metrics to the Global Hub (ACM-47259)

The regional hub (RH) keeps all its metrics, its own and its leaf clusters', in its own Thanos.
Its **own** platform metrics (its `local-cluster`) are also remote-written to the Global Hub (GH).
Leaf clusters (e.g. vm-cluster) stay regional and never get GH's URL or cert.

```
RH MCOA, placement rh-self   = [local-cluster] ─► RH Thanos + GH Thanos   {cluster=local-cluster, managed_hub=regional-hub}
RH MCOA, placement rh-leaves = [vm-cluster]    ─► RH Thanos only
GH MCOA, placement global-mcoa = [local-cluster] (excludes regional-hub)  {cluster=local-cluster}   (no managed_hub)
```
In GH's Thanos, RH's own series carry `managed_hub="regional-hub"`; GH's own series have no `managed_hub`.

## Bootstrap (the only manual apply besides the prerequisites)
    KUBECONFIG=$HOME/.kube/acm-31440-gh oc apply -f regional-hub-metrics/apps/

| Application | Runs against | Folder | What it does |
|---|---|---|---|
| `sync-secret-to-regional-hub` | GH (in-cluster) | `sync-secret/` | Policy `open-cluster-management-global-set/gh-cert-to-regional-hub`, enforced by GH's governance on its local-cluster. It owns ManifestWork `regional-hub/acm47259-gh-write-cert`, which creates Secret `open-cluster-management-observability/gh-observatorium-client` (GH client cert + GH server CA) on RH. Plus one Role so Argo can create the Policy objects |
| `sync-policy-to-regional-hub` | regional-hub (via GitOpsCluster) | `sync-policy/` | Policy `open-cluster-management-global-set/rh-metrics-to-global-hub`, enforced by RH's governance on its local-cluster. It adds the GH remoteWrite, the cert secret and `managed_hub=regional-hub` to RH's own platform collector template |

Both Policies live in ACM's `open-cluster-management-global-set` namespace, which ACM creates on every hub already
bound to clusterset `global`. So no Namespace or ManagedClusterSetBinding is needed.

### sync-secret-to-regional-hub
- Managed-cluster templates (`{{ fromSecret ... }}`, no `hub`) are resolved on GH's local-cluster at enforcement time. They read
  `observability-controller-open-cluster-management.io-observability-signer-client-cert` (tls.crt, tls.key) and
  `observability-server-ca-certs` (ca.crt) in `open-cluster-management-observability`, and are re-evaluated, so the
  ManifestWork follows GH's own cert renewal. No key material is in Git or in the Policy objects.
- ManifestWork is the transport because GH's policy controller is off for the hosted-mode regional hub.

### sync-policy-to-regional-hub
- `musthave` appends array items and patches items with a matching `name`
  (rhacm-docs `governance/config_policy_parameters.adoc`). So the Policy adds only remoteWrite `global-hub`,
  `spec.secrets: [gh-observatorium-client]` and `externalLabels.managed_hub=regional-hub` to
  `mcoa-default-platform-metrics-collector-rh-self`. MCOA's own `acm-observability` entry is not in Git.
- MCOA keeps other remoteWrite entries and listed secrets (MCOA `agentssa.go`) and replicates `spec.secrets` only
  to the clusters that template targets (`handler.go:276`), here RH itself. If the entry is removed, the Policy re-adds it.

## Prerequisites (manual, not synced by Argo): one MCOA parent per cluster
The ClusterManagementAddOn placements are cluster setup. Re-apply them after any MCOA disable/enable, which recreates the CMA.

1. **GH: exclude regional-hub from GH's MCOA.** Otherwise both hubs write same-named PrometheusAgents onto RH and overwrite each other.
       KUBECONFIG=$HOME/.kube/acm-31440-gh oc apply -f regional-hub-metrics/prereq/placement-global-mcoa.yaml
       # decision must be exactly [local-cluster]:
       KUBECONFIG=$HOME/.kube/acm-31440-gh oc -n open-cluster-management-global-set get placementdecision \
         -l cluster.open-cluster-management.io/placement=global-mcoa -o jsonpath='{.items[*].status.decisions}'
       KUBECONFIG=$HOME/.kube/acm-31440-gh oc patch clustermanagementaddon multicluster-observability-addon --type=json \
         -p='[{"op":"test","path":"/spec/installStrategy/placements/0/name","value":"global"},{"op":"replace","path":"/spec/installStrategy/placements/0/name","value":"global-mcoa"}]'
2. **RH: split RH's MCOA into rh-self and rh-leaves** so only RH's own collector gets the extra remote write.
       KUBECONFIG=$HOME/.kube/acm-31440-rh oc apply -f regional-hub-metrics/prereq/rh-placements.yaml
       # decisions must be [local-cluster] and [vm-cluster]; back up the CMA (oc get ... -o json) first, then:
       KUBECONFIG=$HOME/.kube/acm-31440-rh oc patch clustermanagementaddon multicluster-observability-addon --type=json \
         --patch-file regional-hub-metrics/prereq/rh-cma-switch.json
   One patch replaces placement `global` with `rh-self` and `rh-leaves` (same configs), so neither ManagedClusterAddOn is deleted.
   MCO then generates the `-rh-self` and `-rh-leaves` templates.

## Verify
- GH Thanos: `count by (cluster, managed_hub) (up)` shows `{local-cluster, regional-hub}` and GH's own `{local-cluster}`, and no `vm-cluster`.
- RH: `oc get prometheusagent.monitoring.rhobs -n open-cluster-management-agent-addon platform-metrics-collector` has remoteWrite `[acm-observability, global-hub]`; vm-cluster's has only `acm-observability`.
- Policies: on GH, `oc get policy -n local-cluster` shows `open-cluster-management-global-set.gh-cert-to-regional-hub` Compliant;
  on RH, it shows `open-cluster-management-global-set.rh-metrics-to-global-hub` Compliant.

## Rollback
1. Delete both Applications. Prune is off, so the objects stay.
2. On RH: delete Policy `open-cluster-management-global-set/rh-metrics-to-global-hub`, then remove the `global-hub` entry, `gh-observatorium-client` and `managed_hub` from the rh-self template.
3. On GH: delete Policy `open-cluster-management-global-set/gh-cert-to-regional-hub` and ManifestWork `regional-hub/acm47259-gh-write-cert`.
4. Prerequisites: patch each CMA back to a single `global` placement. On GH this brings the overwrite clash back.
