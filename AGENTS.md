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
- **Secrets diverge by deploy path, on purpose.** VP path: `ExternalSecret` → ESO → Vault, no API keys in Git. QS path: plain `kind: Secret`, populated by `qs-out/scripts/create-secrets.sh` — no Vault, no ESO required. Both are driven by the same top-level `secrets:` block in `spec.yaml`; do not add Vault-specific fields to `charts/openshell-saw` — it must stay backend-agnostic (reads `.Values.sshSecret` etc., not a hardcoded ExternalSecret).
- At least one inference provider secret must be set before creating a sandbox.
- The bootc golden image build is one-time and takes ~15 min. Do not re-trigger unnecessarily.
- **`charts/openshell-saw` conventions, learned the hard way (2026-08-03 live-cluster testing):**
  - Any resource a **pre-install hook** Job depends on (its ServiceAccount, Role, RoleBinding, ClusterRole/ClusterRoleBinding) must itself be a pre-install hook at an *earlier* `helm.sh/hook-weight` — plain (non-hook) resources are only created in Helm's main apply phase, which runs *after* all pre-install hooks. Getting this wrong fails every fresh `helm install` outright.
  - Secret field names must match exactly what `create-secrets.sh` generates (`private_key`/`public_key` for `ssh`, not `key`) — a mismatch here fails silently (`set -euo pipefail` aborts the script, or a soft warning if the lookup itself is wrapped in `|| true`), not loudly.
  - Any job image that shells out to `kubectl` must actually have it — `registry.fedoraproject.org/fedora:44` does not. `run-setup.sh`'s pattern (curl the binary from `dl.k8s.io` before using it) is the one to copy, not `hook-inject-ssh.yaml`'s (which assumes it's already present).

## Deploying the Validated Pattern (`vp-out/`)

The Validated Patterns Operator's `Pattern` CRD has no subdirectory-path field — it always
clones the whole repo and expects `values-global.yaml`/`values-prod.yaml`/`Makefile`/
`pattern.sh`/`charts/` at the **repo root**, not nested under `vp-out/`. A live `Pattern`
CR's GitOps reconciliation will not find those files until `vp-out/`'s content is
published to a dedicated tag:

```bash
git add vp-out && git commit -m "Regenerate vp-out"
quickpat publish-vp                                        # → prints the new tag, e.g. vp-v3
cd vp-out && ./pattern.sh make install                      # creates/updates the Pattern CR
oc patch pattern secure-agent-workspace -n patterns-operator --type merge \
  -p '{"spec":{"gitSpec":{"targetRevision":"vp-v3"}}}'      # repoint at the published tag
oc get applications -n vp-gitops                             # monitor sync
```

`./pattern.sh make install` alone will report success even without publishing first — it
operates on the local files directly. The `Pattern` CR it creates will not reconcile until
it's repointed at a published tag.

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
