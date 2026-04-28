# eks-gitops-platform
WIP - Architecture overview and documentation for a GitOps-driven CI/CD platform on AWS EKS

## Project Repositories

| Repo | Description |
|------|-------------|
| [infra-aws](https://github.com/dana951/infra-aws) | Infrastructure provisioning for a production-grade EKS cluster hosting a GitOps-based CI/CD platform |
| [app-source](https://github.com/dana951/app-source) | Application source code with a full CI pipeline driving a GitOps deployment workflow |
| [gitops-manifests](https://github.com/dana951/gitops-manifests) | GitOps repository: the source of truth for Kubernetes workload state, owned and synced by ArgoCD |
| [argocd-apps](https://github.com/dana951/argocd-apps) | ArgoCD bootstrap repository managing all application deployments via the App of Apps pattern |
| [jenkins-shared-lib](https://github.com/dana951/jenkins-shared-lib) | Shared CI library providing reusable pipeline steps across all Jenkins-based pipelines |
| [tests-repo](https://github.com/dana951/tests-repo.git) | Test automation repository with smoke and API E2E checks used as CI/CD validation gates |
