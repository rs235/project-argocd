# Kubernetes GitOps Platform

[![Validate](https://github.com/rs235/gitops/actions/workflows/validate.yml/badge.svg)](https://github.com/rs235/gitops/actions/workflows/validate.yml)

GitOps configuration for deploying Kubernetes workloads and platform services with **Argo CD**. The repository separates cluster-specific application selection from reusable workload manifests and shared Helm values. It defines independent entry points for the `home` and `aws` environments, with each Argo CD instance reconciling its own cluster.

This repository is part of a broader infrastructure project using Terraform, Ansible, and Kubespray. See the [infrastructure repository](https://github.com/rs235/infrastructure) for provisioning and cluster configuration.

## Architecture

The repository uses the Argo CD **App of Apps** pattern. A root Application watches a cluster directory; the YAML files in that directory define the child Applications for that environment.

```text
bootstrap/root-home.yml                 bootstrap/root-aws.yml
          │                                        │
          ▼                                        ▼
  root Application                         root Application
  path: clusters/home                      path: clusters/aws
          │                                        │
          ├── argocd-projects                       ├── argocd-projects
          │     └── projects/                       │     └── projects/
          ├── website                              ├── website
          │     └── applications/workloads/website  │     └── applications/workloads/website
          └── kube-prometheus-stack                └── kube-prometheus-stack
                └── Helm chart + Git values              └── Helm chart + Git values
```

The cluster directories contain Argo CD `Application` manifests directly; they are not Kustomize overlays. `home` and `aws` currently declare the same set of child Applications, while keeping each cluster's entry point and reconciliation independent.

## Repository layout

```text
.
├── .github/workflows/validate.yml
├── applications/
│   ├── platform/
│   │   └── kube-prometheus-stack/values.yml
│   └── workloads/
│       └── website/
│           ├── deployment.yml
│           ├── service.yml
│           └── kustomization.yml
├── bootstrap/
│   ├── argocd.yml
│   ├── root-aws.yml
│   └── root-home.yml
├── clusters/
│   ├── aws/
│   │   ├── argocd-projects.yml
│   │   ├── kube-prometheus-stack.yml
│   │   └── website.yml
│   └── home/
│       ├── argocd-projects.yml
│       ├── kube-prometheus-stack.yml
│       └── website.yml
├── projects/
│   ├── apps.yml
│   ├── argocd.yml
│   ├── platform.yml
│   └── root.yml
├── schemas/
│   ├── application.json
│   ├── applicationset.json
│   └── appproject.json
├── .kube-linter.yml
└── .yamllint.yml
```

| Path | Responsibility |
|---|---|
| `bootstrap/` | Root Application entry points and the separate Argo CD self-management Application. |
| `clusters/<name>/` | Applications enabled for a particular cluster. |
| `applications/workloads/` | Kubernetes manifests for application workloads. |
| `applications/platform/` | Values and configuration for platform services. |
| `projects/` | Argo CD `AppProject` definitions and their source, destination, and resource boundaries. |
| `schemas/` | Schemas used to validate Argo CD custom resources in CI. |
| `.github/workflows/validate.yml` | Pull-request validation; deployment is left to Argo CD. |

## Managed applications

### Website

`applications/workloads/website/` contains a Deployment and a `LoadBalancer` Service for an unprivileged NGINX container. Kustomize renders these manifests and sets the desired replica count. The Deployment also specifies non-root execution, the runtime-default seccomp profile, no privilege escalation, a read-only root filesystem, dropped Linux capabilities, and CPU/memory requests and limits.

The workload manifest is referenced by each cluster's `website` Application. Cluster-specific differences should be added only when the environments genuinely require them.

### Monitoring

Each cluster's `kube-prometheus-stack` Application deploys the `kube-prometheus-stack` Helm chart from the Prometheus Community chart repository, pinned to chart version `87.16.1`. Helm values live in `applications/platform/kube-prometheus-stack/values.yml` and are loaded from this Git repository using Argo CD's multi-source Application support.

The values enable Grafana, Prometheus, and Alertmanager, configure 15-day Prometheus retention, and add a scrape job for Proxmox metrics. The chart itself is sourced from its upstream Helm repository rather than copied into this repository.

### Argo CD

Argo CD is installed as part of the cluster bootstrap process. `bootstrap/argocd.yml` is a separate Application that tracks the upstream Argo CD `v3.5.2` cluster-install manifests. It is not a child of either cluster root and must be applied separately if Argo CD self-management is desired.

## Bootstrap

These commands assume that Argo CD is already installed, the `kubectl` context points to the intended cluster, and the commands are run from the repository root.

The root Application references the `root` AppProject. On a fresh Argo CD installation, create that project before applying the root Application; the root then installs the remaining project definitions through its `argocd-projects` child Application.

For the home environment:

```bash
kubectl apply -f projects/root.yml
kubectl apply -f bootstrap/root-home.yml
```

For the AWS environment, use the AWS cluster's context and apply its matching entry point instead:

```bash
kubectl apply -f projects/root.yml
kubectl apply -f bootstrap/root-aws.yml
```

Use **only the matching root manifest for the target cluster**. The two files create an Application named `root`, but point it at different cluster paths.

After the root has reconciled and the `argocd` AppProject exists, Argo CD self-management can be enabled separately with:

```bash
kubectl apply -f bootstrap/argocd.yml
```

Check the applications and projects with:

```bash
kubectl -n argocd get applications
kubectl -n argocd get appprojects
```

## GitOps workflow

1. Make infrastructure or workload changes in a branch and open a pull request.
2. GitHub Actions runs validation checks against the proposed changes.
3. Merge the pull request to `main` after the checks pass.
4. Argo CD detects the repository change and reconciles the desired state in the relevant cluster.

The workflow does not deploy workloads with `kubectl apply`. Argo CD is responsible for normal continuous deployment. The Applications use automated sync with pruning and self-healing: resources removed from Git may be pruned, and out-of-band changes to managed resources may be reverted.

## CI validation

The `Validate` workflow runs for pull requests targeting `main` and can also be started manually. It currently includes:

- **yamllint** for YAML style and syntax checks.
- **Gitleaks** for secret detection, including Git history.
- **Trivy config** scanning, failing on HIGH and CRITICAL findings.
- **Kustomize rendering** of the website workload, with the rendered output uploaded as a short-lived artifact.
- **kubeconform** against Kubernetes schemas for the rendered workload.
- **KubeLinter** checks on the rendered workload manifests.
- **kubeconform** validation of Argo CD `Application`, `ApplicationSet`, and `AppProject` manifests using the schemas under `schemas/`.

The current Kustomize render and workload schema/lint checks target the website manifests; the commented-out Helm rendering job is not active. The monitoring chart is reconciled by Argo CD from its pinned upstream chart version.

## Argo CD projects and security boundaries

The repository defines four AppProjects:

| Project | Intended use | Scope |
|---|---|---|
| `root` | Root Application and project-management Application | Restricted to the GitOps repository and the `argocd` namespace; permits Argo CD `Application` and `AppProject` resources. |
| `apps` | Workload Applications | Limited to the GitOps repository and the `apps` / `website` namespaces; cluster-scoped resources are restricted to `Namespace`. |
| `platform` | Platform services such as monitoring | Allows the GitOps repository and the Prometheus Community Helm repository; includes broad cluster-resource permissions for chart-managed cluster-scoped resources. |
| `argocd` | Argo CD self-management | Allows the upstream Argo CD repository and broad cluster-resource permissions within the Argo CD installation project. |

These projects create useful separation between root management, application workloads, platform components, and Argo CD itself. The `platform` and `argocd` projects intentionally have broad cluster-resource allow-lists; they should not be described as least-privilege production policies without further tightening and review.

Do not commit credentials, tokens, private keys, or other plaintext secrets. Use a dedicated secret-management approach if workloads need secrets sourced from Git.

## Adding an application

For a namespace-scoped workload:

1. Add its manifests and a Kustomization, if needed, under `applications/workloads/<name>/`.
2. Add an Argo CD `Application` manifest under each relevant `clusters/<cluster>/` directory, pointing to the workload path and using the appropriate AppProject.
3. Confirm that the AppProject permits the source repository, destination namespace, and resource kinds the workload needs.
4. Open a pull request and review the rendered manifests and CI results before merging.

For a third-party Helm chart, add an Argo CD `Application` that pins the chart version and references values stored in this repository. Keep the chart itself upstream unless there is a concrete reason to vendor it.

## Design choices

- **One Argo CD instance per cluster:** each cluster has its own root Application and reconciliation state; no central remote-cluster registration is required.
- **App of Apps:** cluster directories select the child Applications, while the application definitions and values live elsewhere in the repository.
- **Kustomize where useful:** use it for manifest composition and transformations, but do not add overlays when there are no cluster-specific differences.
- **Argo CD owns deployment:** GitHub Actions validates changes, while Argo CD continuously reconciles the merged desired state.
- **Pinned external inputs:** chart versions and CI actions are pinned in configuration to make changes reviewable and reduce ambiguity.
