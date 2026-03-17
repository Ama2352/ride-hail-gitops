# ride-hail-gitops (Repo 3 of 3)

> GitOps source of truth for the Ride-Hailing platform.
> Part of a 3-repo GitOps architecture governed by `Global_Principles.md`.

---

## Architecture Position

```
┌─────────────────────┐     ┌──────────────────────┐     ┌─────────────────────┐
│  ride-hail-platform │     │  ride-hail-services  │     │  ride-hail-gitops   │
│      (Repo 1)       │     │      (Repo 2)        │     │  >>> THIS REPO <<<  │
│                     │     │                      │     │                     │
│  Vagrant, Ansible,  │     │  Go source code,     │     │  K8s manifests,     │
│  K8s bootstrap,     │     │  Dockerfiles,        │     │  Helm values,       │
│  ArgoCD install     │     │  Jenkinsfile (CI)    │     │  ArgoCD App defs    │
└─────────────────────┘     └──────────┬───────────┘     └──────────▲──────────┘
                                       │  git commit image tag      │
                                       └───────────────────────────►┘
                                              ArgoCD reconciles
```

No `kubectl apply` or `helm install` is ever run manually after Day 0.
All cluster state flows through git commits to this repository.

---

## Repository Structure

```
ride-hail-gitops/
├── root-app.yaml                     # Day 0 bootstrap — apply once, ArgoCD does the rest
│
├── platform/
│   ├── argocd/                       # ArgoCD Application manifests (App of Apps)
│   │   ├── shared/                   # Platform tools — deployed once, shared by all envs
│   │   │   ├── grafana-app.yaml
│   │   │   ├── prometheus-app.yaml
│   │   │   ├── sonarqube-app.yaml
│   │   │   ├── istio-base-app.yaml
│   │   │   ├── istio-istiod-app.yaml
│   │   │   └── istio-gateway-app.yaml
│   │   ├── dev/                      # Service apps — dev environment
│   │   │   ├── dispatch-app.yaml
│   │   │   └── notification-app.yaml
│   │   ├── test/                     # Service apps — test environment
│   │   │   ├── dispatch-app.yaml
│   │   │   └── notification-app.yaml
│   │   └── prod/                     # Service apps — prod environment
│   │       ├── dispatch-app.yaml
│   │       └── notification-app.yaml
│   │
│   ├── grafana/values.yaml           # Helm values — Grafana
│   ├── prometheus/values.yaml        # Helm values — Prometheus + Node Exporter
│   ├── sonarqube/values.yaml         # Helm values — SonarQube + PostgreSQL
│   └── istio/
│       ├── istiod-values.yaml        # Helm values — Istio control plane
│       ├── gateway-values.yaml       # Helm values — Istio ingress gateway
│       └── gateway.yaml             # Gateway CRD — entry point for external traffic
│
└── apps/
    ├── dispatch/
    │   ├── base/                     # Deployment, Service, VirtualService, DestinationRule
    │   └── overlays/
    │       ├── dev/                  # namespace: ride-hailing-dev
    │       ├── test/                 # namespace: ride-hailing-test
    │       └── prod/                 # namespace: ride-hailing-prod
    └── notification/
        ├── base/
        └── overlays/
            ├── dev/
            ├── test/
            └── prod/
```

---

## Platform Tools

| Tool | Namespace | NodePort | Helm Chart |
|---|---|---|---|
| Grafana | monitoring | 30300 | `grafana/grafana` 8.x |
| Prometheus | monitoring | 30909 | `prometheus-community/prometheus` 25.x |
| SonarQube | sonarqube | 30090 | `sonarqube/sonarqube` 10.x |
| Istio (base + istiod + gateway) | istio-system | 30080 (HTTP) / 30443 (HTTPS) | `istio-release/…` 1.27.x |

---

## Day 0 — Bootstrap

Prerequisites: ArgoCD is running in the `argocd` namespace (installed by `ride-hail-platform`).

> **With `ride-hail-platform`:** Bootstrap is fully automatic — the Ansible
> `playbook_argocd.yml` applies `root-app.yaml` directly from GitHub at the
> end of provisioning. `vagrant up` is all you need.

If you need to apply it manually (e.g. re-bootstrapping without re-provisioning):

```bash
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/ama2352/ride-hail-gitops/main/root-app.yaml
```

ArgoCD discovers every `Application` manifest under `platform/argocd/` (recursively)
and begins reconciling. Sync waves ensure correct ordering:

```
istio-base (wave -2) → istiod (wave -1) → gateway + platform tools (wave 0) → services
```

---

## CI → GitOps Flow

1. Developer pushes code to `ride-hail-services`.
2. Jenkins builds the Docker image and pushes it to Docker Hub with a unique tag.
3. Jenkins clones this repository and updates the image tag in the overlay:
   ```
   apps/<service>/overlays/dev/kustomization.yaml  →  newTag: "<build-tag>"
   ```
4. Jenkins commits and pushes the change to `main`.
5. ArgoCD detects the diff and rolls out the new image — no human action required.

---

## Namespaces

| Workload | Namespace | Created by |
|---|---|---|
| dispatch-service (dev) | `ride-hailing-dev` | ArgoCD `CreateNamespace=true` |
| dispatch-service (test) | `ride-hailing-test` | ArgoCD `CreateNamespace=true` |
| dispatch-service (prod) | `ride-hailing-prod` | ArgoCD `CreateNamespace=true` |
| notification-service (dev) | `ride-hailing-dev` | ArgoCD `CreateNamespace=true` |
| notification-service (test) | `ride-hailing-test` | ArgoCD `CreateNamespace=true` |
| notification-service (prod) | `ride-hailing-prod` | ArgoCD `CreateNamespace=true` |
| Monitoring | `monitoring` | ArgoCD `CreateNamespace=true` |
| SonarQube | `sonarqube` | ArgoCD `CreateNamespace=true` |
| Istio | `istio-system` | ArgoCD `CreateNamespace=true` |

The `ride-hailing-dev` namespace is automatically labelled `istio-injection: enabled`
via `managedNamespaceMetadata` in both service apps.

---
# ride-hail-gitops

> GitOps source of truth for the Ride-Hailing platform.
> Repo 3 of 3 in a GitOps architecture: **platform** → **services** → **gitops**.

No `kubectl apply` or `helm install` is ever run manually after Day 0.
All cluster state flows exclusively through commits to this repository.

---

## Architecture Overview

```
┌─────────────────────┐     ┌──────────────────────┐     ┌─────────────────────┐
│  ride-hail-platform │     │  ride-hail-services  │     │  ride-hail-gitops   │
│      (Repo 1)       │     │      (Repo 2)        │     │   >>>THIS REPO<<<   │
│                     │     │                      │     │                     │
│  Vagrant · Ansible  │     │  Go source code      │     │  K8s manifests      │
│  K8s bootstrap      │     │  Dockerfiles         │     │  Helm values        │
│  ArgoCD install     │     │  GitLab CI pipeline  │     │  ArgoCD App defs    │
└─────────────────────┘     └──────────┬───────────┘     └──────────▲──────────┘
                                       │  git commit (image tag)    │
                                       └───────────────────────────►┘
                                              ArgoCD reconciles
```

---

## Repository Structure

```
ride-hail-gitops/
├── root-app.yaml                     # Day 0 bootstrap — apply once; ArgoCD manages everything after
│
├── platform/
│   ├── argocd/                       # ArgoCD Application manifests (App of Apps pattern)
│   │   ├── shared/                   # Platform tools — deployed once, shared across all environments
│   │   │   ├── istio-base-app.yaml   # sync-wave: -2 (must apply before istiod)
│   │   │   ├── istio-istiod-app.yaml # sync-wave: -1
│   │   │   ├── istio-gateway-app.yaml
│   │   │   ├── local-path-provisioner-app.yaml
│   │   │   ├── prometheus-app.yaml
│   │   │   ├── grafana-app.yaml
│   │   │   └── sonarqube-app.yaml
│   │   ├── dev/                      # Service apps — dev environment
│   │   │   ├── dispatch-app.yaml
│   │   │   └── notification-app.yaml
│   │   ├── test/                     # Service apps — test environment
│   │   │   ├── dispatch-app.yaml
│   │   │   └── notification-app.yaml
│   │   └── prod/                     # Service apps — prod environment
│   │       ├── dispatch-app.yaml
│   │       └── notification-app.yaml
│   │
│   ├── grafana/values.yaml           # Helm values — Grafana
│   ├── prometheus/values.yaml        # Helm values — Prometheus + Node Exporter
│   ├── sonarqube/values.yaml         # Helm values — SonarQube + PostgreSQL
│   └── istio/
│       ├── istiod-values.yaml        # Helm values — Istio control plane
│       ├── gateway-values.yaml       # Helm values — Istio ingress gateway
│       └── gateway.yaml             # Gateway CRD — cluster entry point for external traffic
│
└── apps/
    ├── dispatch/
    │   ├── base/                     # Deployment, Service, VirtualService, DestinationRule
    │   └── overlays/
    │       ├── dev/                  # namespace: ride-hailing-dev  | image tag updated by CI
    │       ├── test/                 # namespace: ride-hailing-test
    │       └── prod/                 # namespace: ride-hailing-prod
    └── notification/
        ├── base/
        └── overlays/
            ├── dev/
            ├── test/
            └── prod/
```

---

## Platform Tools

| Tool | Namespace | NodePort | Helm Chart |
|---|---|---|---|
| Grafana | `monitoring` | `30300` | `grafana/grafana` 8.x |
| Prometheus | `monitoring` | `30909` | `prometheus-community/prometheus` 25.x |
| SonarQube | `sonarqube` | `30090` | `sonarqube/sonarqube` 10.x |
| Istio base + istiod + gateway | `istio-system` | `30080` (HTTP) · `30443` (HTTPS) | `istio-release/*` 1.27.x |
| local-path-provisioner | `local-path-storage` | — | `rancher/local-path-provisioner` |

---

## Day 0 — Bootstrap

**Prerequisites:** ArgoCD v2.14.9 is installed in the `argocd` namespace (performed automatically by `ride-hail-platform`).

> **With `ride-hail-platform`:** Bootstrap is fully automatic. The Ansible playbook `playbook_argocd.yml` applies `root-app.yaml` at the end of provisioning — `vagrant up` is all you need.

If you need to re-bootstrap manually (e.g. after a failed run without re-provisioning the cluster):

```bash
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/ama2352/ride-hail-gitops/main/root-app.yaml
```

ArgoCD discovers every `Application` manifest under `platform/argocd/` recursively and begins reconciling. Sync waves guarantee a safe rollout order:

```
istio-base (wave -2) ──► istiod (wave -1) ──► gateway + platform tools (wave 0) ──► services
```

---

## CI → GitOps Flow

1. A developer merges a pull request into `main` on `ride-hail-services`.
2. **GitLab CI** builds, scans, and pushes the Docker images to `docker.io/ama2352` with a unique tag (`<pipeline_iid>-<short_sha>`).
3. The `gitops_update_dev` job in GitLab CI clones this repository and patches the image tag in the dev overlays:
   ```
   apps/dispatch/overlays/dev/kustomization.yaml     →  newTag: "<image-tag>"
   apps/notification/overlays/dev/kustomization.yaml →  newTag: "<image-tag>"
   ```
4. GitLab CI commits and pushes the change to `main`.
5. **ArgoCD** detects the diff in the overlay and rolls out the new image automatically — no human intervention required.

---

## Namespaces

| Workload | Namespace | Istio Injection | Created by |
|---|---|---|---|
| dispatch-service (dev) | `ride-hailing-dev` | ✅ enabled | ArgoCD `CreateNamespace=true` |
| dispatch-service (test) | `ride-hailing-test` | — | ArgoCD `CreateNamespace=true` |
| dispatch-service (prod) | `ride-hailing-prod` | — | ArgoCD `CreateNamespace=true` |
| notification-service (dev) | `ride-hailing-dev` | ✅ enabled | ArgoCD `CreateNamespace=true` |
| notification-service (test) | `ride-hailing-test` | — | ArgoCD `CreateNamespace=true` |
| notification-service (prod) | `ride-hailing-prod` | — | ArgoCD `CreateNamespace=true` |
| Monitoring (Prometheus + Grafana) | `monitoring` | — | ArgoCD `CreateNamespace=true` |
| SonarQube | `sonarqube` | — | ArgoCD `CreateNamespace=true` |
| Istio control plane | `istio-system` | — | ArgoCD `CreateNamespace=true` |

The `ride-hailing-dev` namespace is automatically labelled `istio-injection: enabled` via `managedNamespaceMetadata` in the ArgoCD Application manifests for both services.

---

## Adding a New Environment

1. Create `apps/<service>/overlays/<env>/kustomization.yaml` pointing at the base with the target namespace.
2. Create `platform/argocd/<env>/` and add `Application` manifests pointing at the new overlay.
3. Commit and push — `root-app.yaml` uses `recurse: true`, so ArgoCD automatically discovers the new folder and begins reconciling.

No git branching is required. Environments are isolated by directory structure (Global Principle #4 — Folders over Branches).

---

## Key Design Decisions

| Decision | Rationale |
|---|---|
| App of Apps pattern | `root-app.yaml` bootstraps everything in one `kubectl apply`; ArgoCD discovers all `Application` manifests recursively |
| Kustomize overlays for environments | Base manifests are DRY; overlays patch only namespace and image tag — no duplication |
| Helm values in git | Platform tools use upstream Helm charts; the only customization is the values file committed here |
| Sync waves for Istio ordering | CRD installation (`istio-base`) must precede the control plane (`istiod`) which must precede the gateway; sync waves enforce this |
| `CreateNamespace=true` | ArgoCD creates all namespaces declaratively — no manual pre-provisioning required |
| `selfHeal: true` · `prune: true` | Drift is corrected automatically; resources deleted from git are removed from the cluster |
| Folders over branches | Environment differences are directory overlays, not git branches — simpler history and clearer ownership |

---

## Global Principles

1. **Declarative** — Every cluster state is described in Git. No manual `kubectl` or ad-hoc shell for final state.
2. **Repo Separation** — Each repository owns exactly one concern: infrastructure, application code, or desired state.
3. **Pull-Based CD** — GitLab CI pushes images; ArgoCD pulls manifests from this repository.
4. **Folders over Branches** — Environment differences live in directory overlays, not git branches.
