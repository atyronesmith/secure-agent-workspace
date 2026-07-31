# Secure Agent Workspace

Deploy isolated, per-user AI agent sandboxes on OpenShift Virtualization with OIDC authentication and policy-controlled access.

This is the **spec-driven** source repo for the Secure Agent Workspace pattern. `spec.yaml` is the source of truth. Edit it and run `quickpat compose` to regenerate the outputs:

| Directory | Contents | How to deploy |
|---|---|---|
| `vp-out/` | Validated Pattern (ArgoCD, GitOps) | `./vp-out/pattern.sh make install` |
| `qs-out/` | Quickstart (Helm, step-by-step) | See [Option B](#option-b-quickstart-manual-step-by-step) below |
| `charts/` | Hand-written Helm charts | Input to the compiler — edit here |
| `docs/` | Source documentation | Compiled into vp-out/ and qs-out/ READMEs |

Regenerate after editing `spec.yaml` or `charts/`:

```bash
pip install git+https://github.com/atyronesmith/quickpat.git
quickpat compose spec.yaml              # → vp-out/
quickpat compose spec.yaml --format qs  # → qs-out/
```

---

## Overview

Secure Agent Workspace provisions dedicated KubeVirt virtual machines for each user, running NVIDIA OpenShell with NemoClaw/OpenClaw AI agents. Each sandbox is isolated at the VM level, authenticated via OIDC, and connected to the user's chosen inference provider. The platform supports both a GitOps-driven Validated Pattern deployment and a manual quickstart flow.

## Detailed description

Organizations adopting AI coding and knowledge agents need strong isolation guarantees: each user's agent must run in its own boundary, with auditable access to enterprise systems, controlled network egress, and centralized identity management. Traditional container-based isolation is insufficient when agents can execute arbitrary code and tool calls.

This pattern implements NVIDIA's [Secure Agent Workspace reference architecture](https://docs.nvidia.com/enterprise-reference-architectures/secure-agent-workspace-reference-design/latest/openshift-virtualization-reference-implementation.html) on Red Hat OpenShift. Each user gets a dedicated Fedora 44 VM running the OpenShell gateway and an AI agent (OpenClaw, Hermes, or Deep Agents Code). The VM provides process-level and network-level isolation. OIDC authentication (via Red Hat Build of Keycloak) ensures only the sandbox owner can access their workspace. Secrets for inference providers flow through HashiCorp Vault and the External Secrets Operator, keeping API keys out of Git and Helm values.

The system supports multiple inference providers (Gemini, Anthropic, OpenAI, NVIDIA Build, OpenRouter, Ollama, or custom endpoints) and optional web search integration (Tavily, Brave). A bootc-based golden image pipeline pre-bakes all packages into a container image that CDI imports directly, enabling fast VM provisioning without cloud-init package installation.

### Architecture diagrams

The following diagrams are from the [NVIDIA Secure Agent Workspace OpenShift Virtualization Reference Implementation](https://docs.nvidia.com/enterprise-reference-architectures/secure-agent-workspace-reference-design/latest/openshift-virtualization-reference-implementation.html).

#### Reference Architecture

![OpenShift Virtualization Reference Implementation](https://raw.githubusercontent.com/validatedpatterns-sandbox/secure-agent-workspace/main/docs/images/openshift-reference-shape.png)

#### GitOps Policy Model

![GitOps Policy Model — End-to-End Policy Flow](https://raw.githubusercontent.com/validatedpatterns-sandbox/secure-agent-workspace/main/docs/images/gitops-policy-model.png)

#### Storage Layout

![NFS storage layout for policy bundles and workspace persistence](https://raw.githubusercontent.com/validatedpatterns-sandbox/secure-agent-workspace/main/docs/images/nfs-storage-layout.png)

#### Implementation Overview

```
                     OpenShift Cluster
┌──────────────────────────────────────────────────────────┐
│                                                          │
│  Operators (deployed by Validated Pattern or manually):  │
│  ┌──────────────────┐  ┌──────────────────┐              │
│  │ OpenShift        │  │ Red Hat Build    │              │
│  │ Virtualization   │  │ of Keycloak      │              │
│  └──────────────────┘  └──────────────────┘              │
│                                                          │
│  Infrastructure (ArgoCD-managed):                        │
│  ┌──────────┐ ┌──────────┐ ┌──────────────────────────┐  │
│  │ Vault    │ │ ESO      │ │ Keycloak (OIDC provider) │  │
│  └──────────┘ └──────────┘ └──────────────────────────┘  │
│       │                              │                   │
│       │ secrets sync                 │ JWKS validation   │
│       ▼                              ▼                   │
│  ┌──────────────────────────────────────────┐            │
│  │ Golden Image (bootc)                     │            │
│  │ Fedora 44 + OpenShell + podman + nodejs  │            │
│  │ Built via BuildConfig → CDI DataSource   │            │
│  └─────────────────┬────────────────────────┘            │
│                    │ clone per user                      │
│       ┌────────────┼────────────┐                        │
│       ▼            ▼            ▼                        │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐                     │
│  │ alice   │ │ bob     │ │ carol   │  Per-user VMs       │
│  │ sandbox │ │ sandbox │ │ sandbox │  with gateway +     │
│  │  VM     │ │  VM     │ │  VM     │  agent + routes     │
│  └─────────┘ └─────────┘ └─────────┘                     │
│       │            │            │                        │
│       └────────────┼────────────┘                        │
│                    │                                     │
│  Routes:  TLS passthrough (gRPC) + edge (dashboard)      │
└──────────────────────────────────────────────────────────┘
        │
        ▼
   User (openshell CLI / browser)
```

| Component | Technology | Purpose |
|---|---|---|
| VM isolation | OpenShift Virtualization (KubeVirt) | One VM per user with process and network isolation |
| Identity | Red Hat Build of Keycloak (RHBK) | OIDC authentication, user management, SSO |
| Agent runtime | NVIDIA OpenShell + OpenClaw/NemoClaw | AI coding and knowledge agents inside sandbox |
| Gateway | OpenShell Gateway (gRPC over TLS) | Sandbox lifecycle, SSH proxy, inference routing |
| Golden image | Bootc (Fedora 44) + CDI | Pre-baked VM image for fast provisioning |
| Secrets | HashiCorp Vault + External Secrets Operator | API keys for inference providers, SSH keys |
| GitOps | ArgoCD (Validated Patterns) | Declarative cluster configuration |
| Access control | Dashboard token + OIDC gateway validation | Per-user access via application-level tokens |

## Requirements

### Minimum hardware requirements

| Resource | Per sandbox VM | Cluster overhead |
|---|---|---|
| CPU | 4 cores | 8 cores (operators, Keycloak, Vault) |
| Memory | 8 GiB | 16 GiB |
| Storage | 40 GiB (VM disk) | 50 GiB (golden image, registry) |

### Minimum software requirements

| Software | Version |
|---|---|
| Red Hat OpenShift | 4.16+ |
| OpenShift Virtualization operator | stable channel |
| Red Hat Build of Keycloak operator | stable-v26 channel |
| Helm CLI | 3.x |
| oc CLI | matching cluster version |
| openshell CLI | [latest release](https://github.com/NVIDIA/OpenShell/releases) |

### Required user permissions

**Cluster admin** is required for the initial deployment (operator installation, namespace creation). After setup, end users interact only via the `openshell` CLI and their OIDC credentials — no OpenShift access needed.

## Deploy

### Prerequisites

1. An OpenShift 4.16+ cluster with the required operators installed (see Requirements)
2. `oc` CLI logged in with cluster-admin
3. `helm` 3.x installed
4. An API key for at least one inference provider (Gemini, Anthropic, OpenAI, NVIDIA, OpenRouter)
5. The `openshell` CLI installed ([releases](https://github.com/NVIDIA/OpenShell/releases))

Verify prerequisites:

```bash
make check-prereqs
```

### Build the golden VM image (one-time)

Before deploying sandboxes, build the bootc-based Fedora 44 golden image and register it as a CDI DataSource. This runs once per cluster and takes ~15 minutes.

```bash
# Generate SSH keys for sandbox provisioning
make generate-keys

# Build images (runs BuildConfigs in the cluster)
make build                    # NemoClaw sandbox container image
make build-cli                # NemoClaw CLI image
make build-gateway-image      # Bootc gateway VM image → CDI DataSource

# Verify the golden image DataSource is ready
oc get dv openshell-gateway-golden -n build-saw-images
# Wait for PHASE=Succeeded before deploying sandboxes
```

> **Note:** This step will be replaced by pre-built upstream golden images in a future release.

### Installation

Two deployment paths are available:

#### Option A: Validated Pattern (automated, GitOps)

Deploys everything — operators, Vault, ESO, Keycloak, secrets, and the default sandbox configuration — via ArgoCD. All resources are continuously reconciled.

```bash
# 1. Configure secrets (do NOT commit this file)
cp vp-out/values-secret.yaml.template ~/values-secret.yaml
# Edit ~/values-secret.yaml:
#   - Paste SSH keys from make generate-keys output
#   - Set at least one inference provider API key

# 2. Deploy the pattern (runs inside the VP utility container)
cd vp-out
./pattern.sh make install
```

After install, ArgoCD manages all resources. Monitor sync status:

```bash
oc get applications -n openshift-gitops
```

#### Option B: Quickstart (manual, step-by-step)

Install operators from OperatorHub first, then deploy components using Makefile targets.

```bash
# 1. Verify prerequisites
make check-prereqs

# 2. Deploy Keycloak (RHBK operator must be installed from OperatorHub first)
make keycloak

# 3. Verify Keycloak is ready
make keycloak-issuer
curl -sk "$(make keycloak-issuer)/.well-known/openid-configuration" | python3 -m json.tool | head -5

# 4. Authenticate
make login                    # Opens browser → login with alice / alice
make whoami                   # Verify identity

# 5. Create a sandbox
export OPENSHELL_SAW_NAME=alice-openshell-saw
make openshell-saw-create \
  PROVIDER=gemini \
  MODEL=gemini-2.5-flash \
  API_KEY=<your-api-key>

# 6. Follow setup logs (in another terminal)
make openshell-saw-logs

# 7. Check status
make openshell-saw-list
make status

# 8. Wait for VM to be ready
oc get vmi -n openshell-agents
# Wait for PHASE=Running, READY=True

# 9. Configure the openshell CLI
make openshell-saw-configure-gateway

# 10. Login to the gateway
openshell gateway login

# 11. Verify
openshell --gateway-insecure sandbox list
```

> **Note:** The gateway VM uses a self-signed TLS certificate. Pass `--gateway-insecure` to `openshell` commands, or set `export OPENSHELL_GATEWAY_INSECURE=true`.

You can set `OPENSHELL_SAW_NAME` once via `export` and all `openshell-saw-*` targets will use it automatically.

### Supported inference providers

| Provider | Key | Example model |
|---|---|---|
| Google Gemini | `gemini` | `gemini-2.5-flash` |
| Anthropic | `anthropic` | `claude-sonnet-4-6` |
| OpenAI | `openai` | `gpt-4o` |
| NVIDIA Build | `build` | `meta/llama-3.3-70b-instruct` |
| OpenRouter | `openrouter` | `anthropic/claude-sonnet-4-6` |
| Ollama (local) | `ollama` | `llama3` |
| Custom endpoint | `custom` | any (set `ENDPOINT_URL`) |

### Validating the deployment

```bash
# SSH into the sandbox
make openshell-saw-ssh

# Launch the OpenClaw TUI
make openshell-saw-tui

# Open the web UI (port-forward via openshell ssh-proxy)
make openshell-saw-gui
# Opens: http://localhost:18789/#token=<token>

# Access the dashboard directly via the route
oc get route ${OPENSHELL_SAW_NAME}-dashboard -n openshell-agents -o jsonpath='https://{.spec.host}'

# Run the automated E2E test (headless, creates its own sandbox)
make test

# Run offline template validation (43 checks)
./tests/test-oidc-templates.sh
```

### Delete

```bash
# Delete a single sandbox
make openshell-saw-delete

# Delete Keycloak + PostgreSQL
make delete-keycloak

# Delete all quickstart resources (keycloak, images, gateway image)
make delete-all

# Uninstall the validated pattern
cd vp-out && ./pattern.sh make uninstall
```

## Repository structure

```
.
├── spec.yaml                         # Source of truth — edit this, then regenerate
├── docs/
│   └── README.md                     # Source documentation (compiled into vp-out/ and qs-out/)
├── charts/                           # Hand-written Helm charts
│   ├── openshell-keycloak/           # Keycloak CR + KeycloakRealmImport (RHBK operator)
│   ├── openshell-saw/                # Per-user sandbox VM + gateway + agent + routes
│   ├── openshift-cnv/                # HyperConverged CR (activates KubeVirt + CDI)
│   ├── openshift-registry/           # Integrated registry config (for BuildConfig push)
│   └── pattern-secrets/              # ExternalSecrets for provider API keys + SSH
├── image-builder-charts/             # Build-time charts (applied manually, not via ArgoCD)
│   └── helm/
│       ├── nemoclaw-imagestream/     # NemoClaw sandbox image BuildConfig
│       ├── nemoclaw-cli-imagestream/ # NemoClaw CLI image BuildConfig
│       └── openshell-gateway-image/  # Bootc gateway VM image + CDI DataSource
├── overrides/
│   └── openshell-saw.yaml            # Default sandbox values (provider, model, namespace_mode)
├── scripts/                          # Runtime utilities
│   ├── generate-keys.sh              # SSH keypair generation
│   ├── openshell-saw-create.sh       # Sandbox provisioning logic
│   ├── openshell-saw-gui.sh          # Web UI port-forward
│   ├── openshell-saw-logout.sh       # Clear OIDC tokens from VMs
│   └── oidc-login.sh                 # Browser-based OIDC login
├── tests/                            # Test scripts
│   ├── test-bootc-e2e.sh             # Bootc pipeline E2E (28 checks)
│   ├── test-oidc-templates.sh        # Helm template validation (43 checks)
│   ├── test-access-control.sh        # Per-user access control E2E
│   └── test-multiuser-isolation.sh   # Namespace isolation E2E
├── vp-out/                           # Generated Validated Pattern (ArgoCD)
│   ├── values-global.yaml
│   ├── values-prod.yaml
│   ├── values-secret.yaml.template
│   ├── Makefile / Makefile-common
│   ├── pattern.sh
│   └── charts/
└── qs-out/                           # Generated Quickstart Helm chart
    ├── chart/
    └── scripts/create-secrets.sh
```

## References

- [NVIDIA Secure Agent Workspace Reference Design](https://docs.nvidia.com/enterprise-reference-architectures/secure-agent-workspace-reference-design/latest/)
- [OpenShift Virtualization Reference Implementation](https://docs.nvidia.com/enterprise-reference-architectures/secure-agent-workspace-reference-design/latest/openshift-virtualization-reference-implementation.html)
- [NVIDIA OpenShell](https://github.com/NVIDIA/OpenShell)
- [NVIDIA NemoClaw](https://github.com/NVIDIA/NemoClaw)
- [Red Hat Validated Patterns](https://validatedpatterns.io/)
- [Red Hat Build of Keycloak](https://docs.redhat.com/en/documentation/red_hat_build_of_keycloak/)
- [QuickPat compose docs](https://github.com/atyronesmith/quickpat/blob/main/docs/compose-tutorial.md)

## Technical details

### Security model

The system implements layered isolation:

1. **VM-level isolation** — Each user gets a dedicated KubeVirt VM (one VM per user, no shared agent process space)
2. **OIDC authentication** — Keycloak provides SSO with PKCE and device code flow support
3. **Per-user access control** — Auth proxy validates the OIDC token's `preferred_username` matches the sandbox owner
4. **TLS passthrough** — Gateway route preserves gRPC/HTTP2 end-to-end; the gateway validates OIDC tokens directly
5. **Secret management** — API keys flow through Vault + ESO; the user's SSH private key never touches the cluster in plaintext

### Keycloak test users

| Username | Password | Roles |
|---|---|---|
| `developer` | `developer` | `openshell-user` |
| `admin` | `admin` | `openshell-user`, `openshell-admin` |
| `alice` | `alice` | `openshell-user`, `openshell-admin` |
| `bob` | `bob` | `openshell-user`, `openshell-admin` |

### Namespace modes

| Mode | Description |
|---|---|
| `shared` (default) | All sandboxes in one namespace. Scales to thousands of users. |
| `perUser` | Each user gets `saw-<username>` namespace. Kubernetes-level resource isolation. |

Change the mode in `overrides/openshell-saw.yaml` and re-deploy.

### OIDC issuer resolution

The sandbox chart resolves the OIDC issuer URL automatically:
- **Validated Pattern flow:** Computed from `global.clusterDomain` injected by ArgoCD at deploy time
- **Quickstart flow:** Detected from the Keycloak route at `make openshell-saw-create` time
- **Manual override:** Set `oidc.issuerUrl` explicitly in `overrides/openshell-saw.yaml`

## Tags

| Field | Value |
|---|---|
| **Title** | Secure Agent Workspace |
| **Description** | Deploy isolated, per-user AI agent sandboxes on OpenShift Virtualization |
| **Industry** | Cross-industry |
| **Product** | Red Hat OpenShift |
| **Use case** | AI agent sandboxing, secure coding environments |
| **Partner** | NVIDIA |
