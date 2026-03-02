# ride-hail-gitops

GitOps source of truth for the Ride-Hailing platform. ArgoCD watches this
repository and reconciles the cluster state to match every commit.

---

## Role in the 3-repo Architecture

| Repo | Purpose |
|---|---|
| `ride-hail-platform` | VM provisioning — Vagrant + Ansible, Kubernetes, ArgoCD bootstrap |
| `ride-hail-services` | Application source code + Jenkins CI pipelines |
| **`ride-hail-gitops`** | **Cluster desired state — Kustomize manifests + Helm values** |

No `kubectl apply` or `helm install` commands are ever run manually after Day 0.
All changes flow through git commits to this repository.

---

## Directory Structure

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

```bash
kubectl apply -f root-app.yaml -n argocd
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

## Adding a New Environment

1. Create `apps/<service>/overlays/<env>/kustomization.yaml` with the target namespace.
2. Create `platform/argocd/<env>/` and add an `Application` manifest pointing at the new overlay.
3. Commit and push — ArgoCD reconciles automatically (`recurse: true` in `root-app.yaml` picks up the new folder).

No branching required. Environments are isolated by directory, not by git branch
(Global Principle #4 — Folders over Branches).
