# Overview

Argo CD GitOps repository built using App of Apps pattern with support for multi cluster setups and strong focus on simplicity, manageablility and security. Principle of least priviledge was a leading factor in a design of Argo CD Projects and Kubernetes deployments.

This repository is a part of a bigger infrastructure as code project built with:

- **Terraform** to provision and manage infrastructure.
- **Ansible** to configure VMs and execute Kubespray.
- **Kubespray** to install Kubernetes and base cluster components.
- **Argo CD** which is initially installed by Kubespray to deploy and manage cluster workloads.
- **GitHub Actions** for CI/CD.

[IaC repository link](https://github.com/rs235/project-infrastructure)

## Repository structure

```text
.
├── applications  # Contains Kubernetes manifests and additional configuration files if needed.
│   ├── platform
│   │   └── kube-prometheus-stack
│   │       └── values.yml
│   └── workloads
│       └── website
│           ├── deployment.yml
│           ├── kustomization.yml
│           └── service.yml
├── bootstrap  # Argo CD bootstrap applications. Applied manually on fresh cluster installation.
│   ├── argocd.yml
│   ├── root-aws.yml
│   └── root-home.yml
├── clusters  # Defines cluster specific configuration.
│   ├── aws
│   │   ├── argocd-projects.yml
│   │   ├── kube-prometheus-stack.yml
│   │   └── website.yml
│   └── home
│       ├── argocd-projects.yml
│       ├── kube-prometheus-stack.yml
│       └── website.yml
├── projects  # Argo CD Project manifests. 
│   ├── apps.yml
│   ├── argocd.yml
│   ├── platform.yml
│   └── root.yml
├── README.md
└── schemas  # Kubernetes Custom Resource Definitions (CRD) for Argo CD manifests. Used in Actions by kubeconform.
    ├── application.json
    ├── applicationset.json
    └── appproject.json
```

## Architecture

This repository utilises **App of Apps** pattern and offers support for multiple clusters.

```mermaid
---
config:
  theme: redux
---
flowchart LR
    n1["Git repository"] --> n2["Root application"]
    n2 --> n3["Cluster applications"]
    n3 --> n4["Platform services"] & n5["Application workloads"]

     n1:::Sky
     n2:::Sky
     n3:::Sky
     n4:::Sky
     n5:::Sky
     classDef Sky stroke-width:1px, stroke-dasharray:none, stroke:#374D7C, fill:#E2EBFF, color:#374D7C
```

#### App of Apps pattern 

App of Apps allows for automated deployment and configuration of cluster resources with a ***Git repository as a single source of truth***. Continuous repository scans and cluster state comparisons detect configuration drift and trigger reconciliation steps for ensuring parity between repository held configuration and actual state of deployment.

#### Multi-cluster model

This repository assumes that each Kubernetes cluster has its own Argo CD instance and its own root Application. This design choice was made to avoid one central Argo CD instance that manages many remote clusters. This architectural choice aligns with project goal of creating a simple, manageable and secure GitOps setup. Decentralised cluster management minimises potential blast radius of Argo CD or cluster downtime and increases security by eliminating a need for shared cluster credentials while only marginally increasing maintenance cost.

<!-- #### Bootstrap

For this project Argo CD installation is done by Kubespray as part of the infrastructure provisioning managed by the IaC project: **LINK**

The only manual bootstrap step is applying the cluster specific root Application:

```bash
kubectl apply -f bootstrap/root-<cluster>.yml
```

After that, Argo CD takes ownership of the repository's desired state.

Verify:

```bash
kubectl -n argocd get applications
kubectl -n argocd get appprojects
```

The root Application then creates cluster-level child Applications. -->

## CI/CD

This project has a fairly simple but pretty powerful CI pipeline built on top of GitHub Actions that is triggered automatically on new pull request.

The responsibility of this pipeline is restricted to code validation and security checks while deployment is handled by Argo CD.

#### CI workflow

``` mermaid
flowchart LR
    trigger["Pull Request"]

    trigger --> yamllint
    trigger --> gitleaks
    trigger --> trivy
    trigger --> kustomize
    trigger --> argocd

    subgraph yamllint["YAML Linter"]
        direction LR
        y1["Checkout"] --> y2["Install yamllint"] --> y3["Run yamllint"]
    end

    subgraph gitleaks["Gitleaks"]
        direction LR
        g1["Checkout<br/>fetch-depth: 0"] --> g2["Run Gitleaks"]
    end

    subgraph trivy["Trivy"]
        direction LR
        t1["Checkout"] --> t2["Run Trivy<br/>config scan<br/>HIGH, CRITICAL"]
    end

    subgraph kustomize["Kustomize Render"]
        direction LR
        k1["Checkout"] --> k2["Set up kubectl<br/>v1.35.4"] --> k3["Create directory<br/>rendered/kustomize"] --> k4["Render manifests<br/>kubectl kustomize"] --> k5["Upload rendered manifests<br/>kustomize-rendered"]
    end

    subgraph kubeconform["Kubernetes Schema Validation"]
        direction LR
        kc1["Download Kustomize manifests"] --> kc2["Debug rendered manifests"] --> kc3["Validate manifests<br/>kubeconform v0.8.0"]
    end

    subgraph kubelinter["KubeLinter"]
        direction LR
        kl1["Download Kustomize manifests"] --> kl2["Run KubeLinter<br/>directory: rendered"]
    end

    subgraph argocd["Argo CD Schema Validation"]
        direction LR
        a1["Checkout"] --> a2["Validate manifests<br/>kubeconform v0.8.0"]
    end

    kustomize --> kubeconform
    kustomize --> kubelinter
```

**YAML Linter** - This step uses **yamllint** version 1.38.0 to recursively scan all YAML files in the repository and report any discovered YAML formatting errors.

**Secrets Scan** - **Gitleaks** is used to scan all files in the repository for potential unsecured credentials.

**Security Scan** - Repo security is assessed by Trivy in misconfioguration scanning mode. This scan is restricted to repo files only with no access to Kubernetes cluster.

**Kustomize Render** - Kustomize file file validity is confirmed by **kubectl kustomize** render. Rendered manifests are then uploaded as a GitHub Artifact for further validation by Kubeconform and KubeLinter.

**Manifest Validation** - Kubernetes and Argo CD manifests are validated by **Kubeconform**. Argo CD validation is done with official Argo CD CRDs converted to Kubeconform supported JSON format. 

**Best Practices Scan** - Rendered kustomize files are scanned by KubeLinter to detect misconfiguation and best practice violations.

## Future improvements

1. ApplicationSets for multi cluster configurations.
2. Expanded applications list making better use of the cluster.
3. Inclusion of Helm charts.
4. Documentation improvements.