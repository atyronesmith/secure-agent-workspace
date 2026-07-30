# Secure Agent Workspace

Deploy isolated, per-user AI agent sandboxes on OpenShift Virtualization with OIDC authentication and policy-controlled access.

## Overview

Organizations adopting AI coding and knowledge agents need strong isolation guarantees: each user's agent must run in its own boundary, with auditable access to enterprise systems, controlled network egress, and centralized identity management. Traditional container-based isolation is insufficient when agents can execute arbitrary code and tool calls.

This pattern implements NVIDIA's [Secure Agent Workspace reference architecture](https://docs.nvidia.com/enterprise-reference-architectures/secure-agent-workspace-reference-design/latest/openshift-virtualization-reference-implementation.html) on Red Hat OpenShift. Each user gets a dedicated Fedora 44 VM running the OpenShell gateway and an AI agent (OpenClaw, Hermes, or Deep Agents Code). The VM provides process-level and network-level isolation. OIDC authentication via Red Hat Build of Keycloak ensures only the sandbox owner can access their workspace. Secrets for inference providers flow through HashiCorp Vault and the External Secrets Operator, keeping API keys out of Git and Helm values.

The system supports multiple inference providers (Gemini, Anthropic, OpenAI, NVIDIA Build, OpenRouter, Ollama, or custom endpoints) and optional web search integration (Tavily, Brave). A bootc-based golden image pipeline pre-bakes all packages into a container image that CDI imports directly, enabling fast VM provisioning without cloud-init package installation.

## Architecture

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
│  │ sandbox │ │ sandbox │ │ sandbox │  gateway + agent    │
│  └─────────┘ └─────────┘ └─────────┘                     │
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
| GitOps | ArgoCD (Validated Patterns framework) | Declarative cluster configuration |
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

**Cluster admin** is required for initial deployment. After setup, end users interact only via the `openshell` CLI and their OIDC credentials — no OpenShift access needed.

## Prerequisites

Before deploying, ensure you have:

1. An OpenShift 4.16+ cluster with cluster-admin access
2. `oc` CLI authenticated against the cluster
3. `helm` 3.x installed locally
4. An API key for at least one inference provider
5. The `openshell` CLI installed ([releases](https://github.com/NVIDIA/OpenShell/releases))

## Build the golden VM image (one-time)

This step builds three container images in-cluster (~15 minutes total) and registers a CDI DataSource that sandbox VMs are cloned from. It runs once before the first sandbox is created.

```bash
# 1. Clone this repository
git clone https://github.com/atyronesmith/secure-agent-workspace.git
cd secure-agent-workspace

# 2. Generate SSH keys for sandbox provisioning
make generate-keys

# 3. Build images (runs BuildConfigs in the cluster)
make build                    # NemoClaw sandbox container image
make build-cli                # NemoClaw CLI image
make build-gateway-image      # Bootc gateway VM image → CDI DataSource

# 4. Verify the golden image DataSource is ready
oc get dv openshell-gateway-golden -n build-saw-images
# Wait for PHASE=Succeeded before deploying sandboxes
```

## Deploy

<!-- vp-only -->
### Option A: Validated Pattern (automated, GitOps)

Deploys everything — operators, Vault, ESO, Keycloak, secrets, and the default sandbox configuration — via ArgoCD. All resources are continuously reconciled.

```bash
# 1. Configure secrets (do NOT commit this file)
cp vp-out/values-secret.yaml.template ~/values-secret.yaml
# Edit ~/values-secret.yaml — paste SSH keys from make generate-keys output
# and set at least one inference provider API key

# 2. Deploy the pattern
cd vp-out
./pattern.sh make install
```

After install, ArgoCD manages all resources. Monitor progress in the ArgoCD dashboard or with:

```bash
oc get applications -n openshift-gitops
```

**Delete the VP:**
```bash
cd vp-out
./pattern.sh make uninstall
```
<!-- end -->

<!-- qs-only -->
### Option B: Quickstart (manual, step-by-step)

Deploys components using Helm and Makefile targets. Useful when you want to inspect or modify each step before applying.

```bash
# 1. Verify prerequisites
make check-prereqs

# 2. Deploy Keycloak (RHBK operator must be installed from OperatorHub first)
make keycloak

# 3. Verify Keycloak is ready
make keycloak-issuer
curl -sk "$(make keycloak-issuer)/.well-known/openid-configuration" | python3 -m json.tool | head -5

# 4. Authenticate
make login     # opens browser → log in as alice / alice
make whoami    # verify identity

# 5. Create your first sandbox
export OPENSHELL_SAW_NAME=alice-openshell-saw
make openshell-saw-create \
  PROVIDER=gemini \
  MODEL=gemini-2.5-flash \
  API_KEY=<your-api-key>

# 6. Check status
make openshell-saw-list
oc get vmi -n openshell-agents
# Wait for PHASE=Running, READY=True

# 7. Configure the CLI
make openshell-saw-configure-gateway

# 8. Login to the gateway
openshell gateway login

# 9. Verify
openshell --gateway-insecure sandbox list
```

> **Note:** The gateway VM uses a self-signed TLS certificate. Pass `--gateway-insecure` to `openshell` commands until you configure a real certificate.

**Delete resources:**
```bash
make openshell-saw-delete     # delete a single sandbox
make delete-keycloak          # delete Keycloak + PostgreSQL
make delete-all               # delete all quickstart resources
```
<!-- end -->

## Supported inference providers

| Provider | Key | Example model |
|---|---|---|
| Google Gemini | `gemini` | `gemini-2.5-flash` |
| Anthropic | `anthropic` | `claude-sonnet-4-6` |
| OpenAI | `openai` | `gpt-4o` |
| NVIDIA Build | `build` | `meta/llama-3.3-70b-instruct` |
| OpenRouter | `openrouter` | `anthropic/claude-sonnet-4-6` |
| Ollama (local) | `ollama` | `llama3` |
| Custom endpoint | `custom` | any (set `ENDPOINT_URL`) |

## Validate the deployment

```bash
# SSH into your sandbox
make openshell-saw-ssh

# Launch the OpenClaw TUI
make openshell-saw-tui

# Open the web UI
make openshell-saw-gui
# Opens: http://localhost:18789/#token=<token>

# Run the automated E2E test
make test

# Run offline template validation
./tests/test-oidc-templates.sh
```

## Security model

The system implements layered isolation:

1. **VM-level isolation** — Each user gets a dedicated KubeVirt VM (one VM per user, no shared agent process space)
2. **OIDC authentication** — Keycloak provides SSO with PKCE and device code flow support
3. **Per-user access control** — Auth proxy validates the OIDC token's `preferred_username` matches the sandbox owner
4. **TLS passthrough** — Gateway route preserves gRPC/HTTP2 end-to-end; the gateway validates OIDC tokens directly
5. **Secret management** — API keys flow through Vault + ESO; the user's SSH private key never touches the cluster in plaintext

## Keycloak test users

| Username | Password | Roles |
|---|---|---|
| `developer` | `developer` | `openshell-user` |
| `admin` | `admin` | `openshell-user`, `openshell-admin` |
| `alice` | `alice` | `openshell-user`, `openshell-admin` |
| `bob` | `bob` | `openshell-user`, `openshell-admin` |

## Namespace modes

| Mode | Description |
|---|---|
| `shared` (default) | All sandboxes in one namespace. Scales to thousands of users. |
| `perUser` | Each user gets `saw-<username>` namespace. Kubernetes-level resource isolation. |

Change the mode in `overrides/openshell-saw.yaml` and re-deploy.

## References

- [NVIDIA Secure Agent Workspace Reference Design](https://docs.nvidia.com/enterprise-reference-architectures/secure-agent-workspace-reference-design/latest/)
- [OpenShift Virtualization Reference Implementation](https://docs.nvidia.com/enterprise-reference-architectures/secure-agent-workspace-reference-design/latest/openshift-virtualization-reference-implementation.html)
- [NVIDIA OpenShell](https://github.com/NVIDIA/OpenShell)
- [NVIDIA NemoClaw](https://github.com/NVIDIA/NemoClaw)
- [Red Hat Validated Patterns](https://validatedpatterns.io/)
- [Red Hat Build of Keycloak](https://docs.redhat.com/en/documentation/red_hat_build_of_keycloak/)
- [QuickPat compose docs](https://github.com/atyronesmith/quickpat/blob/main/docs/compose-tutorial.md)
