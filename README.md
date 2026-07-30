# Secure Agent Workspace

Deploy isolated, per-user AI agent sandboxes on OpenShift Virtualization with OIDC authentication.

This is the **spec-driven** source repo for the Secure Agent Workspace pattern. `spec.yaml` is the source of truth. The generated outputs are committed here:

| Directory | Contents | How to use |
|---|---|---|
| `vp-out/` | Validated Pattern (ArgoCD, GitOps) | `./vp-out/pattern.sh make install` |
| `qs-out/` | Quickstart (Helm, manual) | `helm install saw qs-out/chart` |
| `charts/` | Custom application charts | Input to the compiler |

## What it deploys

- **OpenShift Virtualization** — one KubeVirt VM per user (not containers — real VM isolation)
- **RHBK (Keycloak)** — OIDC authentication; the OpenShell gateway validates user JWTs before granting sandbox access
- **NVIDIA OpenShell + NemoClaw/OpenClaw** — AI coding and knowledge agents inside each VM
- **Vault + ESO** — inference provider API keys managed through Vault, never in Git
- **8 inference providers** — Anthropic, Gemini, OpenAI, NVIDIA Build, OpenRouter, Vertex, Tavily, Brave Search

## Quick start

```bash
# Generate or regenerate VP and QS outputs from spec.yaml
pip install quickpat
quickpat compose spec.yaml                   # → vp-out/
quickpat compose spec.yaml --format qs       # → qs-out/

# Option A: Validated Pattern (automated GitOps)
cp values-secret.yaml.template ~/values-secret.yaml
# Edit ~/values-secret.yaml — set at least one inference provider API key
./vp-out/pattern.sh make install

# Option B: Quickstart (step-by-step)
# See vp-out/README.md for Makefile targets
```

## Minimum requirements

| Resource | Per sandbox VM | Cluster overhead |
|---|---|---|
| CPU | 4 cores | 8 cores (operators, Keycloak, Vault) |
| Memory | 8 GiB | 16 GiB |
| Storage | 40 GiB (VM disk) | 50 GiB (golden image, registry) |

**Software:** OpenShift 4.16+, OpenShift Virtualization operator, Red Hat Build of Keycloak operator.

## Architecture

```
┌──────────────────────────────────────────────────────┐
│                  OpenShift Cluster                   │
│                                                      │
│  Operators (from spec.yaml blocks):                  │
│  ┌──────────────────┐  ┌──────────────────┐          │
│  │ OpenShift        │  │ Red Hat Build    │          │
│  │ Virtualization   │  │ of Keycloak      │          │
│  └──────────────────┘  └──────────────────┘          │
│                                                      │
│  Infrastructure (ArgoCD-managed):                    │
│  ┌──────────┐ ┌──────────┐ ┌──────────────────────┐  │
│  │ Vault    │ │ ESO      │ │ Keycloak (OIDC)      │  │
│  └──────────┘ └──────────┘ └──────────────────────┘  │
│       │                              │               │
│       │ secrets sync                 │ JWKS          │
│       ▼                              ▼               │
│  ┌────────────────────────────────────────────────┐  │
│  │ Golden Image (bootc, Fedora 44)                │  │
│  │ OpenShell + podman + nodejs (CDI DataSource)   │  │
│  └─────────────────────┬──────────────────────────┘  │
│                        │ clone per user              │
│       ┌────────────────┼────────────┐                │
│       ▼                ▼            ▼                │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐              │
│  │  alice  │  │   bob   │  │  carol  │  VMs         │
│  │ sandbox │  │ sandbox │  │ sandbox │  + gateway   │
│  └─────────┘  └─────────┘  └─────────┘  + agent    │
└──────────────────────────────────────────────────────┘
```

## References

- [NVIDIA Secure Agent Workspace Reference Design](https://docs.nvidia.com/enterprise-reference-architectures/secure-agent-workspace-reference-design/latest/)
- [Red Hat Validated Patterns](https://validatedpatterns.io/)
- [QuickPat compose docs](https://github.com/atyronesmith/quickpat/blob/main/docs/compose-tutorial.md)
