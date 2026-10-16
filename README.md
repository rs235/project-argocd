# Overview

GitOps repository for managing Kubernetes workloads with Argo CD. This repository is built around App of Apps concepts and support for multiple clusters while still keeping things simple and manageable. Security was a major concern with separation of platform components and application workloads and minimal permissions based on actual application needs.

This repository is a part of a bigger infrastructure as code project that uses:

- **Terraform** to provision and manage infrastructure.
- **Ansible** to configure VMs and execute Kubespray.
- **Kubespray** to install Kubernetes and base cluster components.
- **Argo CD** which is initially installed by Kubespray to deploy and manage cluster workloads.
- **GitHub Actions** for CI/CD.

[IaC repository link](https://github.com/rs235/project-infrastructure) 

## Repository structure

```text
.
├── apps/
│   └── nginx/
│       ├── base/
│       │   ├── deployment.yaml
│       │   ├── service.yaml
│       │   └── kustomization.yaml
│       └── overlays/
│           ├── home/
│           │   └── kustomization.yaml
│           └── aws/
│               └── kustomization.yaml
│
├── argocd/
│   └── projects/
│       ├── apps.yaml
│       └── platform.yaml
│
├── bootstrap/
│   └── root-app.yaml
│
├── clusters/
│   ├── home/
│   │   ├── kustomization.yaml
│   │   ├── platform.yaml
│   │   └── apps.yaml
│   └── aws/
│       ├── kustomization.yaml
│       ├── platform.yaml
│       └── apps.yaml
│
└── platform/
    └── monitoring/
        ├── application.yaml
        └── values.yaml
```

## GitOps architecture

The repository uses the **App of Apps** pattern.

```text
                    Git repository
                         │
                         │
                  bootstrap/root-app
                         │
                         ▼
                  clusters/<cluster>
                         │
              ┌──────────┴──────────┐
              │                     │
              ▼                     ▼
        platform Application   apps Application
              │                     │
              ▼                     ▼
       platform/* resources    apps/* resources
```

For the `home` cluster:

```text
root-app
  └── clusters/home
        ├── platform
        │     └── kube-prometheus-stack
        └── apps
              └── nginx
```

The important distinction is:

- `clusters/` describes **what is enabled on a particular cluster**.
- `platform/` contains shared platform components.
- `apps/` contains workload definitions.
- `argocd/projects/` contains Argo CD authorization boundaries.
- `bootstrap/` contains the one-time bootstrap manifest.

## Multi-cluster model

Each Kubernetes cluster has its own Argo CD instance and its own root Application.

This is deliberately simpler than operating one central Argo CD instance that manages many remote clusters.

For example:

```text
Cluster: home
  Argo CD
    └── root-app -> clusters/home

Cluster: aws
  Argo CD
    └── root-app -> clusters/aws
```

Both clusters can use the same application base while selecting different overlays.

This gives us:

- clear cluster ownership
- no cross-cluster Argo CD credentials
- no manually maintained remote-cluster registration
- simple failure isolation
- an easy path to add more clusters later

If the project grows into a centralized multi-cluster platform, **ApplicationSet** and Argo CD cluster registration can be introduced later. They are intentionally not required here.

## Bootstrap

Argo CD itself is installed by the Kubernetes bootstrap process (Kubespray in this project).

The only manual GitOps bootstrap step is applying the root Application:

```bash
kubectl apply -f bootstrap/root-app.yaml
```

After that, Argo CD takes ownership of the repository's desired state.

Verify:

```bash
kubectl -n argocd get applications
kubectl -n argocd get appprojects
```

The root Application should create the cluster-level child Applications.

For another cluster, apply that cluster's root Application after changing its `path`:

```text
clusters/home
```

to:

```text
clusters/aws
```

A cleaner operational approach is to keep one small bootstrap file per cluster if different clusters are bootstrapped independently.

## Adding an application

1. Create the application's manifests under `apps/<name>/`.
2. Add a base if the application has reusable Kubernetes resources.
3. Add cluster overlays only when cluster-specific differences are needed.
4. Add an Argo CD `Application` under the appropriate `clusters/<cluster>/` directory.
5. Add the Application to that cluster's `kustomization.yaml`.
6. Commit and push.

Example:

```text
apps/
└── whoami/
    ├── base/
    └── overlays/
        └── home/
```

Then create:

```text
clusters/home/apps.yaml
```

with an Application pointing to:

```text
apps/whoami/overlays/home
```

Argo CD detects the Git change and synchronizes it.

## Platform applications

Platform applications are things the cluster needs to operate, rather than user workloads.

Examples:

- monitoring
- ingress controller
- cert-manager
- external-dns
- storage components

Third-party Helm charts should normally be deployed through an Argo CD `Application`.

For example, kube-prometheus-stack is represented by:

```text
platform/monitoring/application.yaml
platform/monitoring/values.yaml
```

The Helm chart itself is not copied into this repository.

## Application Projects

Two projects are intentionally used:

### `platform`

Used for cluster/platform components.

### `apps`

Used for workloads deployed into application namespaces.

The projects provide a basic security boundary and make the repository structure clearer.

The permissions are intentionally broad enough for a learning/home-lab Kubernetes platform. In a production environment, these should be tightened to the minimum required resources and namespaces.

## Kustomize

Kustomize is used only where it provides a real benefit.

The nginx application has:

```text
base/
```

for common manifests and:

```text
overlays/home/
overlays/aws/
```

for cluster-specific differences.

Do not create an overlay for every application automatically. If an application is identical on every cluster, point the Application directly at its base.

## Helm

Helm is preferred for third-party software that is already distributed as a Helm chart.

The repository stores:

- the Argo CD Application
- Helm values

It does not store rendered Helm manifests.

This keeps upgrades straightforward: change the chart version or values, commit, and let Argo CD reconcile.

## Secrets

Do **not** commit plaintext Kubernetes Secrets containing passwords, API keys, tokens, or private keys.

For a future production-oriented version, use one of:

- SOPS + age
- External Secrets Operator
- a cloud secret manager
- another dedicated secret-management solution

For the current project, keeping secrets out of Git is sufficient.

## What belongs where?

| Directory | Purpose |
|---|---|
| `bootstrap/` | One-time Argo CD bootstrap |
| `clusters/` | Cluster-specific desired state and enabled applications |
| `argocd/projects/` | Argo CD AppProjects |
| `apps/` | Application workload manifests |
| `platform/` | Cluster/platform services |

A useful rule is:

> `clusters/` decides **what runs where**.  
> `apps/` and `platform/` define **how it runs**.

## CI/CD responsibility

GitHub Actions should validate changes before they reach the cluster.

A sensible next step is a workflow that runs on pull requests and performs:

```text
yamllint
kustomize build
kubectl apply --dry-run=client
```

Argo CD remains responsible for deployment.

GitHub Actions should **not** run `kubectl apply` for normal application deployments. That would duplicate the deployment mechanism and weaken the GitOps model.

The desired flow is:

```text
Developer
   │
   ▼
Git commit / Pull Request
   │
   ▼
GitHub Actions
   │
   ├── YAML validation
   ├── Kustomize validation
   └── manifest checks
   │
   ▼
Merge to main
   │
   ▼
Argo CD
   │
   ▼
Kubernetes
```

## Bootstrap vs continuous deployment

There is deliberately a small bootstrap exception to GitOps.

### Bootstrap

Performed once:

```bash
kubectl apply -f bootstrap/root-app.yaml
```

### Continuous operation

After bootstrap:

```text
Git
 ↓
Argo CD
 ↓
Kubernetes
```

This is normal for an App of Apps setup. Something must initially tell Argo CD which Git path to watch.

## Design goals

This repository intentionally avoids:

- unnecessary ApplicationSets
- deeply nested App of Apps
- generated YAML
- excessive Kustomize overlays
- a central cluster-management abstraction
- copying third-party Helm charts into Git
- GitHub Actions performing deployments

Those features can be introduced when the project actually needs them.

The goal is a repository that is:

- easy to understand
- reproducible
- reviewable
- extensible
- suitable for a junior DevOps/GitOps portfolio
- close enough to real operational patterns to discuss in an interview

## Interview / CV talking points

This project demonstrates:

- Infrastructure as Code with Terraform
- configuration management with Ansible
- Kubernetes provisioning with Kubespray
- GitOps with Argo CD
- App of Apps
- Kustomize bases and overlays
- Helm-based third-party application deployment
- Kubernetes namespaces and resource ownership
- declarative reconciliation
- CI validation with GitHub Actions
- multi-cluster repository organization
