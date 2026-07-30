# secure-agent-workspace

Isolated, per-user AI agent sandboxes on OpenShift Virtualization. Each user gets a dedicated KubeVirt VM running NVIDIA OpenShell with OIDC authentication (RHBK) and Vault-managed inference provider keys.

## Prerequisites

Install these on your cluster before deploying:


## Install

**Option A — values.yaml secrets** (CI / automated):

```bash
helm install secure-agent-workspace ./chart \
  --set secrets.vllmApiKey=<your-key> \
  --set secrets.minioAccessKey=admin \
  --set secrets.minioSecretKey=adminpassword \
  -n secure-agent-workspace --create-namespace
```

**Option B — create secrets out-of-band** (interactive / production):

```bash
oc new-project secure-agent-workspace
./scripts/create-secrets.sh
helm install secure-agent-workspace ./chart -n secure-agent-workspace
```

## Regenerate

This chart is generated from `spec.yaml` by quickpat:

```bash
quickpat compose spec.yaml --format qs
```
