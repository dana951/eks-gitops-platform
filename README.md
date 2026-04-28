# eks-gitops-platform

This is the metadata and entry-point repository for a GitOps-driven CI/CD platform on AWS EKS, with references to all repositories that make up the project in [Project Repositories](#project-repositories).

## Executive Overview

This project demonstrates a delivery model with clear separation of responsibilities:

- **Infrastructure as Code** provisions AWS and EKS foundations.
- **CI pipelines** build, test, and validate application artifacts.
- **GitOps workflows** promote deployments through Git commits.
- **Argo CD** continuously reconciles cluster state from Git.


## Platform Architecture (High Level)

Core components and responsibilities:

- **AWS + EKS** host application workloads and platform tooling.
- **GitHub Actions** automates source workflows and builds the image artifact.
- **Jenkins (in-cluster)** orchestrates tests (smoke/E2E) and deployment pipelines
- **GitOps manifests repo** is the source of truth for Kubernetes desired state.
- **Argo CD (in-cluster)** syncs desired state to the cluster.
- **Automated tests** (smoke + E2E) act as promotion quality gates.



## Project Repositories

| Repo | Description |
|------|-------------|
| [infra-aws](https://github.com/dana951/infra-aws) | Infrastructure provisioning for a production-grade EKS cluster hosting a GitOps-based CI/CD platform |
| [app-source](https://github.com/dana951/app-source) | Application source code with a full CI pipeline driving a GitOps deployment workflow |
| [gitops-manifests](https://github.com/dana951/gitops-manifests) | GitOps repository: the source of truth for Kubernetes workload state, owned and synced by ArgoCD |
| [argocd-apps](https://github.com/dana951/argocd-apps) | ArgoCD bootstrap repository managing all application deployments via the App of Apps pattern |
| [jenkins-shared-lib](https://github.com/dana951/jenkins-shared-lib) | Shared CI library providing reusable pipeline steps across all Jenkins-based pipelines |
| [tests-repo](https://github.com/dana951/tests-repo.git) | Test automation repository with smoke and API E2E checks used as CI/CD validation gates |

## License

Each repository contains its own license file. See the `LICENSE` file in the specific repository.
