# secure-agent-workspace

Isolated, per-user AI agent sandboxes on OpenShift Virtualization. Each user gets a dedicated KubeVirt VM running NVIDIA OpenShell with OIDC authentication (RHBK) and Vault-managed inference provider keys.

- **Version:** 0.1.0
- **Source:** ``

## Architecture

## Required OpenShift Operators

The following operators are automatically installed by the Validated Pattern:

| Operator | Subscription | Channel | Source |
|----------|-------------|---------|--------|
| Red Hat Build of Keycloak | rhbk-operator | stable-v26 | redhat-operators |
| OpenShift Virtualization | kubevirt-hyperconverged | stable | redhat-operators |

## Framework Architecture

This pattern uses the **multisource configuration** approach. Infrastructure Helm charts (clustergroup, vault, external-secrets) are pulled dynamically from the upstream Validated Patterns registry rather than stored locally. This means:

- No fork of multicloud-gitops required
- Upstream bug fixes are received by bumping `clusterGroupChartVersion`
- No `common/` git subtree needed (modern patterns use Ansible collections in the utility container)

The `pattern.sh` script runs all make targets inside a podman-based utility container (`quay.io/validatedpatterns/utility-container`) which includes the `rhvp.cluster_utils` Ansible collection and all required tooling.

> **Note:** The multisource feature is not yet documented on validatedpatterns.io but is used by all current production patterns (multicloud-gitops, rag-llm-gitops) and documented in the [common repo README](https://github.com/validatedpatterns/common).

## Pattern Configuration

- **Pattern name:** secure-agent-workspace
- **Application name:** secure-agent-workspace
- **Namespace:** secure-agent-workspace
- **Chart strategy:** remote
- **Vault enabled:** True

## Deployment

```bash
git init && git add -A && git commit -m "Initial pattern"
oc login <cluster>
./pattern.sh make install
```
