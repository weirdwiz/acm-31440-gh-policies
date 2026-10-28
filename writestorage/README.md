# writestorage: regional hub's own metrics to Global Hub (ACM-47259 e2e)

## Bootstrap (one command, the only manual apply)
    KUBECONFIG=$HOME/.kube/acm-31440-gh oc apply -f writestorage/apps/

This creates two Argo Applications in `openshift-gitops` on the Global Hub. Both track branch `writestorage-e2e`.
Every other object comes from Git through them. No RBAC is applied by hand.

| Application | Destination | Path (recursive) | What it syncs |
|---|---|---|---|
| `apps/gh.yaml` `writestorage-gh` | GH (in-cluster) | `gh/` | see waves below |
| `apps/rh.yaml` `writestorage-rh` | regional-hub (Argo cluster secret `regional-hub`) | `rh/` | RH label templates (rh-self, rh-leaves) and the rh-self platform agent's extra remoteWrite to GH. Retries with backoff until the cert exists |

## Waves inside `writestorage-gh`
| Wave | Path | Objects |
|---|---|---|
| 0 | `gh/cert/` | ClusterSetBinding and Placement `gh-local-only` (GH local-cluster only); Policy `acm47259-gh-write-cert` that owns ManifestWork `regional-hub/acm47259-gh-write-cert` (Secret `gh-observatorium-client` on RH, data rendered from GH's MCO-managed secrets at runtime, nothing in Git); GH MCOA templates with `externalLabels.managed_hub=global-hub` |
| 1 | `gh/mcoa/` (gate, added in a later commit) | Sync hook Job that waits until ManifestWork `regional-hub/acm47259-gh-write-cert` has condition Applied |
| 2 | `gh/mcoa/` (later commit) | Placement `global-mcoa` (adopted unchanged); ClusterManagementAddOn `multicluster-observability-addon` with placement `global` changed to `global-mcoa`. This is the clash fix |

The CMA switch is held back until the cert and the RH writes are verified, so the gate runs first.
Operator-owned objects (MCO, PrometheusAgent templates, CMA) are applied with ServerSideApply and carry
`Prune=false,Delete=false`. Argo owns only the fields listed here.

## Regional hub (`rh/`): only RH's own metrics go to GH
- `rh/rh-self-remotewrite.yaml`: PrometheusAgent `mcoa-default-platform-metrics-collector-rh-self` gets `externalLabels.managed_hub=regional-hub`, secret `gh-observatorium-client` in `spec.secrets`, and a second remoteWrite `global-hub` (GH receive URL, certs from that secret). It lists the acm-observability entry too, copied from MCOA's rendering, because remoteWrite is an atomic list.
- `rh/hub-templates-labels.yaml`: `managed_hub=regional-hub` on the other three RH templates (rh-self UWL, rh-leaves platform and UWL). rh-leaves (vm-cluster) never gets the GH entry.
- Argo syncs with `ServerSideApply=true` and no prune. One object per file: two files for one object under one field manager drop fields.
- MCOA keeps the extra remoteWrite and secret (agentssa.go keeps non-acm remoteWrite entries and user secrets) and replicates `gh-observatorium-client` into the agent namespace on RH's local-cluster only (handler.go:276).
- Removed in ACM-47259 cc4c9c9: the MCO writeStorage and `gh-observatorium-endpoint`. writeStorage forwarded leaves too and failed on the tenant header.

## Prerequisite on regional-hub (manual, NOT synced by Argo)
Argo does not own the CMA or these Placements. The RH CMA `multicluster-observability-addon` had one placement `global`, which covered both clusters. Two placements now split it:
1. Placements `rh-self` (name In [local-cluster]) and `rh-leaves` (name NotIn [local-cluster]) in `open-cluster-management-global-set`: `KUBECONFIG=$HOME/.kube/acm-31440-rh oc apply -f writestorage/prereq/rh-placements.yaml`. Verify the decisions are [local-cluster] and [vm-cluster].
2. Back up the CMA first (`oc get clustermanagementaddon multicluster-observability-addon -o json`). Then apply the single JSON patch (test on placements/0/name=global, then replace the list with rh-self and rh-leaves, same configs):
       KUBECONFIG=$HOME/.kube/acm-31440-rh oc patch clustermanagementaddon multicluster-observability-addon --type=json --patch-file writestorage/prereq/rh-cma-switch.json
   Both MCAs (local-cluster, vm-cluster) stay. MCO then generates the `-rh-self` and `-rh-leaves` templates.
- Rollback: back up is `scratchpad/mcoa/backup-rh/rh-cma-mcoa.json`. Patch the CMA back to a single `global` placement with the same configs; delete placements `rh-self` and `rh-leaves`. MCO then regenerates the `-global` templates, so drop the `-rh-self`/`-rh-leaves` labels and revert the RH commits (ae50e0e, 716e461, cb44c06) too.
- MCO tracking note: the RH MCO `observability` is not in Git, so its Argo tracking annotation was removed by hand so the app is Synced.

## Secrets
Nothing secret is committed. The ManifestWork Secret data is rendered on GH from the MCO-managed secrets
`open-cluster-management-observability/observability-controller-open-cluster-management.io-observability-signer-client-cert`
(tls.crt, tls.key) and `observability-server-ca-certs` (ca.crt). The RH agent object holds only the GH URL
and the names of the cert keys.

## Rollback (GH side)
1. Set the CMA placement back to `global` (backup: `scratchpad/mcoa/backup-A/gh-cma-mcoa.yaml`) or revert the
   wave-2 commit. GH's MCOA then targets regional-hub again, and the clash returns.
2. Delete ManifestWork `regional-hub/acm47259-gh-write-cert` and Policy `acm47259-gh-write-cert`
   (namespace `open-cluster-management-observability`).
3. Delete the two Applications (`writestorage-gh`, `writestorage-rh`). Do not delete the CMA.
4. Remove the `managed_hub` externalLabels from the MCOA templates on both hubs if the label should go.

## Prerequisite: one MCOA parent per cluster (manual, NOT synced by Argo)
Why: regional-hub is a GH managed cluster and also runs its own MCOA. Both hubs' `multicluster-observability-addon`
deployed the same-named PrometheusAgents to regional-hub, and the last writer won. The remote-write URL flipped
between GH and RH. The fix is to make GH's MCOA target everything except regional-hub.

- YAML: `prereq/placement-global-mcoa.yaml` (Placement `global-mcoa`, predicate `name NotIn [regional-hub]`).
  It is referenced by no Application. Adopt it by hand:
      KUBECONFIG=$HOME/.kube/acm-31440-gh oc apply -f writestorage/prereq/placement-global-mcoa.yaml
- Verify first. The decision must be exactly `local-cluster`:
      KUBECONFIG=$HOME/.kube/acm-31440-gh oc -n open-cluster-management-global-set get placementdecision \
        -l cluster.open-cluster-management.io/placement=global-mcoa -o jsonpath='{.items[*].status.decisions}'
- Switch the CMA (JSON patch with a test op, so it fails if the CMA is not on `global`):
      KUBECONFIG=$HOME/.kube/acm-31440-gh oc patch clustermanagementaddon multicluster-observability-addon --type=json \
        -p='[{"op":"test","path":"/spec/installStrategy/placements/0/name","value":"global"},{"op":"replace","path":"/spec/installStrategy/placements/0/name","value":"global-mcoa"}]'
- Side effect seen in this lab: MCO then creates `mcoa-default-*-global-mcoa` PrometheusAgent templates, adds two
  `-global-mcoa` entries to the CMA configs, and drops the `-global` GH templates. The GH label templates in
  `gh/cert/mcoa-templates.yaml` therefore target the `-global-mcoa` names.
- Rollback: patch back with `test` on `global-mcoa` and `replace` to `global`. Then GH's MCOA targets regional-hub again.
  Expect the clash to return. Do not delete the Placement or the CMA.
