# writestorage: regional-hub writeStorage to Global Hub (ACM-47259 e2e)

## Bootstrap (one command, the only manual apply)
    KUBECONFIG=$HOME/.kube/acm-31440-gh oc apply -f writestorage/apps/

This creates two Argo Applications in `openshift-gitops` on the Global Hub. Both track branch `writestorage-e2e`.
Every other object comes from Git through them. No RBAC is applied by hand.

| Application | Destination | Path (recursive) | What it syncs |
|---|---|---|---|
| `apps/gh.yaml` `writestorage-gh` | GH (in-cluster) | `gh/` | see waves below |
| `apps/rh.yaml` `writestorage-rh` | regional-hub (Argo cluster secret `regional-hub`) | `rh/` | MCO writeStorage, endpoint secret, RH label templates. Retries with backoff until the cert exists |

## Waves inside `writestorage-gh`
| Wave | Path | Objects |
|---|---|---|
| 0 | `gh/cert/` | ClusterSetBinding and Placement `gh-local-only` (GH local-cluster only); Policy `acm47259-gh-write-cert` that owns ManifestWork `regional-hub/acm47259-gh-write-cert` (Secret `gh-observatorium-client` on RH, data rendered from GH's MCO-managed secrets at runtime, nothing in Git); GH MCOA templates with `externalLabels.managed_hub=global-hub` |
| 1 | `gh/mcoa/` (gate, added in a later commit) | Sync hook Job that waits until ManifestWork `regional-hub/acm47259-gh-write-cert` has condition Applied |
| 2 | `gh/mcoa/` (later commit) | Placement `global-mcoa` (adopted unchanged); ClusterManagementAddOn `multicluster-observability-addon` with placement `global` changed to `global-mcoa`. This is the clash fix |

The CMA switch is held back until the cert and the RH writes are verified, so the gate runs first.
Operator-owned objects (MCO, PrometheusAgent templates, CMA) are applied with ServerSideApply and carry
`Prune=false,Delete=false`. Argo owns only the fields listed here.

## Secrets
Nothing secret is committed. The ManifestWork Secret data is rendered on GH from the MCO-managed secrets
`open-cluster-management-observability/observability-controller-open-cluster-management.io-observability-signer-client-cert`
(tls.crt, tls.key) and `observability-server-ca-certs` (ca.crt). The RH endpoint Secret holds only the URL
and the names of the cert keys.

## Rollback
1. Set the CMA placement back to `global` (backup: `scratchpad/mcoa/backup-A/gh-cma-mcoa.yaml`) or revert the
   wave-2 commit. GH's MCOA then targets regional-hub again, and the clash returns.
2. Remove `spec.storageConfig.writeStorage` from MCO `observability` on RH and delete Secret
   `gh-observatorium-endpoint` there.
3. Delete ManifestWork `regional-hub/acm47259-gh-write-cert` and Policy `acm47259-gh-write-cert`
   (namespace `open-cluster-management-observability`).
4. Delete the two Applications (`writestorage-gh`, `writestorage-rh`). Do not delete the CMA.
5. Remove the `managed_hub` externalLabels from the MCOA templates on both hubs if the label should go.
