# AGENTS.md — Secure Agent Workspace

## What this repo is

Spec-driven source repo for the Secure Agent Workspace Validated Pattern and Quickstart.

- `spec.yaml` — QuickPat ApplicationSpec (source of truth)
- `charts/` — hand-written Helm charts for custom components
- `image-builder-charts/` — in-cluster bootc image build pipeline charts
- `vp-out/` — generated Validated Pattern (ArgoCD) — **do not edit directly**
- `qs-out/` — generated Quickstart Helm chart — **do not edit directly**

## Key constraints

- **Never edit `vp-out/` or `qs-out/` directly.** They are compiler output. Edit `spec.yaml` or `charts/` and regenerate.
- **All secrets go through Vault.** No API keys in Git. Use the ExternalSecret pattern in `charts/pattern-secrets/`.
- At least one inference provider secret must be set before creating a sandbox.
- The bootc golden image build is one-time and takes ~15 min. Do not re-trigger unnecessarily.

## How to regenerate outputs

```bash
pip install quickpat
quickpat compose spec.yaml              # regenerates vp-out/
quickpat compose spec.yaml --format qs  # regenerates qs-out/
```

## Block types used

This pattern uses three block types added to QuickPat specifically for this pattern:

| Block type | Operator | Purpose |
|---|---|---|
| `openshift-virtualization` | `kubevirt-hyperconverged` | KubeVirt + CDI runtime |
| `keycloak-oidc` | `rhbk-operator` | OIDC identity provider |
| `vm-workspace` | (none — uses above) | Per-user VM sandboxes |

See [QuickPat docs/adding-block-types.md](https://github.com/atyronesmith/quickpat/blob/main/docs/adding-block-types.md).

## Namespace layout

| Namespace | Contents |
|---|---|
| `openshift-cnv` | CNV/KubeVirt operator |
| `rhbk-operator` | RHBK operator |
| `openshell-agents` | Keycloak, Vault, ESO, sandbox VMs |
| `build-saw-images` | Bootc image BuildConfigs, CDI DataSource |
| `vault` | HashiCorp Vault |
| `external-secrets` | ESO operator |

## Test users (dev only)

| Username | Password | Role |
|---|---|---|
| `alice` | `alice` | admin |
| `bob` | `bob` | user |
| `developer` | `developer` | user |
| `admin` | `admin` | admin |
