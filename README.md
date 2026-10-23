# acm-31440-gh-policies

VM metrics rollup from a hosted-mode regional hub to the Global Hub (ACM-47259). The Global Hub's
Argo CD (push model, GitOpsCluster) syncs this repo into the regional hub's own ACM; a Policy on the
regional hub's local-cluster adds a second remoteWrite to the MCOA PrometheusAgent, so VM metrics
from its managed clusters reach the Global Hub's Thanos.

- `policies/`: plain manifests synced by Argo app `regional-hub-policies` (the metrics rollup policy).
- `charts/rollup-inputs/`: Helm chart synced by Argo app `regional-hub-rollup-inputs`; the TLS client cert and URL are
  passed in the Application's `helm.valuesObject` on the Global Hub, never committed here.
