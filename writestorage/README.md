# writestorage: regional-hub writeStorage to Global Hub (ACM-47259 e2e)

## Bootstrap (one command, the only manual apply)
    KUBECONFIG=$HOME/.kube/acm-31440-gh oc apply -f writestorage/apps/root.yaml

The root Application (openshift-gitops on the Global Hub) syncs `apps/children/` from branch
`writestorage-e2e`. Every other object comes from Git through it.

## Layout and sync order
| Wave | Child Application | Destination | Path | Contents |
|---|---|---|---|---|
| 0 | `writestorage-gh-cert` | GH (in-cluster) | `gh/cert/` | Namespace `acm47259-policies`; ClusterSetBinding and Placement `gh-local-only` (GH local-cluster only); Policy `acm47259-gh-write-cert` that owns ManifestWork `regional-hub/acm47259-gh-write-cert` (Secret `gh-observatorium-client`, data rendered from GH's MCO-managed source secrets, not stored in Git); GH MCOA templates with `externalLabels.managed_hub=global-hub` |
| 1 | `writestorage-rh` | RH (Argo cluster secret `regional-hub`) | `rh/` | MCO `observability` `storageConfig.writeStorage` (partial); Secret `gh-observatorium-endpoint` (`ep.yaml`: URL and key names only); RH MCOA templates with `externalLabels.managed_hub=regional-hub` (partial) |
| 2 | `writestorage-gh-mcoa` | GH (in-cluster) | `gh/mcoa/` | Placement `global-mcoa` (adopted unchanged); ClusterManagementAddOn `multicluster-observability-addon` with placement `global` changed to `global-mcoa` (the clash fix) |

Argo syncs the waves in order. Policy and ManifestWork have no health check, so wave 1 can start before the
cert exists; MCO retries until the secret appears. Wave 2 removes RH from GH's MCOA. Do not sync wave 2
until the cert and the writeStorage writes are verified.

No Helm, no kustomize. All Applications use `targetRevision: writestorage-e2e`.
Operator-owned objects (MCO, PrometheusAgent templates, CMA) are applied with ServerSideApply and carry
`Prune=false,Delete=false`. Argo owns only the fields listed here.

## Secrets
Nothing secret is committed. The ManifestWork Secret data is rendered on GH from the MCO-managed secrets
`open-cluster-management-observability/observability-controller-open-cluster-management.io-observability-signer-client-cert`
(tls.crt, tls.key) and `observability-server-ca-certs` (ca.crt). The RH endpoint Secret holds only the URL
and the names of the cert keys.

## Rollback
1. Set the CMA placement back to `global` (backup: `scratchpad/mcoa/backup-A/gh-cma-mcoa.yaml`), or revert the
   wave-2 commit. This brings back GH's MCOA work for regional-hub, and the clash returns.
2. Remove `spec.storageConfig.writeStorage` from MCO `observability` on RH and delete Secret
   `gh-observatorium-endpoint` there.
3. Delete ManifestWork `regional-hub/acm47259-gh-write-cert` and Policy `acm47259-gh-write-cert`
   (namespace `acm47259-policies`).
4. Delete the child Applications. Do not delete the CMA.
5. Remove the `managed_hub` externalLabels from the MCOA templates on both hubs if the label should go.
