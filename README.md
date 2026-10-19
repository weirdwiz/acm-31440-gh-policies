# acm-31440-gh-policies

Test repo for ACM-47259: Global Hub's Argo CD (push model, GitOpsCluster) syncs
`policies/` into a hosted-mode regional hub's own ACM, which propagates the Policy
to its managed clusters. Demo content only.

- `policies/`: plain manifests synced by Argo app `regional-hub-policies` (test policy + metrics rollup policy).
- `charts/rollup-inputs/`: Helm chart synced by Argo app `regional-hub-rollup-inputs`; the TLS client cert and URL are
  passed in the Application's `helm.valuesObject` on the Global Hub, never committed here.
