# ride-hail-gitops (Repo 3 of 3)

> GitOps source of truth for the Ride-Hailing platform.
> Part of a 3-repo GitOps architecture.

---

## 🔗 Related Repositories
- [ride-hail-platform](https://github.com/ama2352/ride-hail-platform) (Repo 1 - Infrastructure)
- [ride-hail-services](https://github.com/ama2352/ride-hail-services) (Repo 2 - Application Source)
- **ride-hail-gitops** (Repo 3 - You are here)

---

## 🏛️ Architecture Overview

```text
┌─────────────────────┐     ┌──────────────────────┐     ┌─────────────────────┐
│  ride-hail-platform │     │  ride-hail-services  │     │  ride-hail-gitops   │
│      (Repo 1)       │     │      (Repo 2)        │     │  >>> THIS REPO <<<  │
│                     │     │                      │     │                     │
│  Vagrant, Ansible,  │     │  Go source code,     │     │  K8s manifests,     │
│  K8s bootstrap,     │     │  Dockerfiles,        │     │  Helm values,       │
│  ArgoCD install     │     │  Jenkins & GitLab CI │     │  ArgoCD App defs    │
└─────────────────────┘     └──────────┬───────────┘     └──────────▲──────────┘
                                       │  git commit image tag      │
                                       └───────────────────────────►┘
                                              ArgoCD reconciles
```

All cluster state (infrastructure tools and deployed APIs) flows exclusively through commits to this repository. No `kubectl apply` or `helm install` is ever run manually after Day 0.

---

## 🚀 Dual-CI Workflows

This system supports two independent Continuous Integration (CI) methods. Both pipelines end by publishing a container image and automatically triggering a deployment via GitOps.

**Jenkins Workflow (Legacy/Alternative):**
1. Code pushed to `ride-hail-services`.
2. Jenkins builds the Docker images and pushes to Docker Hub.
3. Jenkins clones `ride-hail-gitops`, patches the `dev` Kustomize overlays with the new tag, and pushes back to `main`.
4. ArgoCD detects the new tag and rolls out the deployment.

**GitLab CI Workflow (Primary):**
1. Code pushed to `ride-hail-services`.
2. GitLab CI natively builds, scans (Trivy), and pushes images using Docker-in-Docker.
3. The `gitops_update_dev` job clones `ride-hail-gitops`, natively updates the `newTag` in Kustomization files, and pushes the commit.
4. ArgoCD handles the sync.

---

## ⚙️ Setup Guide (Fresh Environment)

1. **Infrastructure:** Go to `ride-hail-platform` and run `vagrant up` to provision the VMs and install Kubernetes/ArgoCD.
2. **Bootstrap:** The Vagrant provisioning natively executes `kubectl apply -n argocd -f root-app.yaml`. 
3. **Reconciliation:** ArgoCD uses the parameterized Helm "App of Apps" pattern (`platform/argocd/`) to resolve and deploy all dependent shared tools and custom apps.

*(To migrate origin URLs from GitHub to GitLab, simply update `repoURL` in `root-app.yaml` and `platform/argocd/values.yaml`.)*
