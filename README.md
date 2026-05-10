# eks-gitops-platform

This repository is the **entry point** and **metadata** for a `GitOps-driven` CI/CD platform on AWS EKS.

The implementation is spread across **six** companion repositories, each summarized in the `Project Repositories` table below.

> Built with: **GitHub Actions**, **Jenkins (JCasC)**, **AWS EKS**, **Helm**, **Docker**, **K8s**, **Python**, **Pytest**, **Groovy** (Jenkinsfiles and shared library)

## Project Repositories

| Repo | Description |
|------|-------------|
| [infra-aws](https://github.com/dana951/infra-aws) | Terraform provisioning for VPC, EKS, add-ons, and platform tooling |
| [app-source](https://github.com/dana951/app-source) | Application source code with a full CI/CD pipeline driving a GitOps deployment workflow |
| [gitops-manifests](https://github.com/dana951/gitops-manifests) | Kubernetes desired state (source of truth consumed by Argo CD) |
| [argocd-apps](https://github.com/dana951/argocd-apps) | Argo CD bootstrap repository using App-of-Apps pattern and ApplicationSets |
| [jenkins-shared-lib](https://github.com/dana951/jenkins-shared-lib) | Shared Jenkins pipeline library providing reusable pipeline steps|
| [tests-repo](https://github.com/dana951/tests-repo.git) | Smoke and E2E test suite used as release quality gates |

> Note: For spcific repo design and implementation details refer to the repo's **README** file.

## Overview

This project demonstrates a delivery model with clear separation of responsibilities:

- **Terraform (IaC)** provisions AWS network, EKS, add-ons, and tooling.
- **GitHub Actions + Jenkins** run CI/CD, promotion, and quality gates.
- **GitOps repository** hold desired Kubernetes state.
- **Argo CD** continuously reconciles cluster state from Git.

## CI/CD and GitOps Flow

> This platform uses a **GitHub branch strategy** (short-lived branches and **pull requests into `main`**).

- This platform implements **continuous integration** and **continuous delivery/deployment**.
- The app is packaged as a Helm chart
- **Image tags** are what move through the pipeline - **not** a rebuild of the image after PR validation.

### On pull request (`feature/*` → `main`)

1. In GitHub Actions workflow - **Lint / static analysis** → **unit tests** → build **Docker image** with tag `{short-sha}-dev` → push to Docker Registry → triggers Jenkins → wait for Jenkins build status. 
2. **Jenkins** runs **smoke** and **E2E** tests against `preview environment` (ephemeral, isolated testing env), it does that by → creating a side branch in [gitops-manifests](https://github.com/dana951/gitops-manifests) repository (with a specific naming format) and an environment values file containing the `{short-sha}-dev` image tag → open PR → wait for **Argo CD** sync → run tests once the environment is ready.

### On Merge to `main`

1. In GitHub Actions workflow - the **same image** is **promoted by retagging only** (no rebuild) as `{short-sha}` and `latest` in the registry → triggers Jenkins → wait for Jenkins build status.  
2. **Jenkins** updates **staging** values file in the [gitops-manifests](https://github.com/dana951/gitops-manifests) repository → wait for **Argo CD** sync → run **smoke** tests once the environment is ready → wait for **human approval** → upon approval updates **prod** values file in the [gitops-manifests](https://github.com/dana951/gitops-manifests) → wait for **Argo CD** sync → run **smoke** tests once the environment is ready.

### Rollback behavior

The platform uses a **safety-first rollback model** to protect production stability:

- **Production rollback is automatic** if a pipeline `failure` or `abort` happens after the prod update is pushed to the GitOps repository.
- The pipeline restores the **last known stable production version** by creating a **new GitOps commit** that sets production back to the previously live image tag (instead of reverting an older commit), then Argo CD reconciles the cluster to that desired state.
- A new rollback commit (forward commit) is used instead of `git revert` to avoid accidentally undoing unrelated changes that may have landed on `main` between the deploy and the rollback.
- If deployment is stopped at the **manual approval gate**, production remains unchanged.
- **Staging is not auto-rolled back** on failure; it is intentionally left available for troubleshooting, and a later run replaces it.

### Required GitHub settings

These settings apply to the [**application**](https://github.com/dana951/app-source) repository where PR and merge workflows live.

**Branch protection (`main`)** - *Settings → Branches → Add rule for `main`*

- **Require a pull request before merging** - all changes land via PR.  
- **Require branches to be up to date before merging** - PRs must include the latest `main`; add the workflow jobs as **required status checks** so nothing merges untested.

**Merge strategy** - *Settings → General → Pull Requests*

- Allow **merge commits** only; **disable squash and rebase** merges.  
- Promotion uses the merge commit’s **second parent (`HEAD^2`)** to find the commit that was built and tested in PR validation. Squash/rebase rewrites history and breaks that link, so promotion would target an image that was never built in this pipeline.

For workflow file names, required status checks, secrets, and concurrency behavior, see the **[app-source](https://github.com/dana951/app-source)** repository README (application source, GitHub Actions workflows, and Jenkinsfiles).

## Platform Architecture (High Level)

Core components and responsibilities:

- **AWS + EKS** host application workloads and platform tooling.
- **GitHub Actions** automates source workflows and builds the image artifact.
- **Jenkins (in-cluster)** orchestrates tests (smoke/E2E) and deployment pipelines
- **GitOps manifests repo** is the source of truth for Kubernetes desired state.
- **Argo CD (in-cluster)** syncs desired state to the cluster.
- **Automated tests** (smoke + E2E) act as promotion quality gates.



## Current AWS/EKS Architecture (PoC)

### Network and cluster foundations

- **Region:** `us-east-1`
- **VPC:** `10.0.0.0/16`
- **Subnets:** 3 public + 3 private subnets across AZs
- **Internet egress:** 1 Internet Gateway + **single NAT Gateway**
- **EKS endpoint:** private access **enabled** and public access **enabled**
- **Public endpoint restriction:** operationally limited to operator IP range (security posture for admin access, CIDR Whitelisting)
- **Worker placement:** all managed node groups are in **private subnets**

### Managed node groups (private)

All node groups are currently configured with `min=1`, `max=1`, `desired=1` (fixed-size for predictable PoC cost/behavior).

Scheduling is enforced with **node labels** on each managed node group and matching **`nodeSelector`** on Deployments so workloads land only on the intended pool.

| Node group | Purpose |
|------------|---------|
| `jenkins` | Jenkins controller and Argo CD |
| `jenkins-agents` | Ephemeral Jenkins Kubernetes agents (build/test isolation) |
| `github-runners` | Self-hosted GitHub Actions runners |
| `podinfo-app` | Application workloads and environment-specific runtime pods |

### Why 4 node groups?

The split is intentional and production-aligned:

- **Isolation of failure domains** - CI spikes or misbehaving runners do not impact core control-plane tooling.
- **Independent scaling policy** - each capacity pool can scale with different rules and priorities.
- **Workload-specific governance** - labels/taints and pod scheduling policies can be applied per class of workload.
- **Security and blast-radius reduction** - tighter IAM/network policies per worker class are easier to enforce.


### Access model (current)

To minimize cost, Jenkins and Argo CD UI are currently accessed via `kubectl port-forward` to ClusterIP services (no public ingress/LB exposed for UIs in this PoC). The **AWS Load Balancer Controller** is installed with **Helm** as part of the cluster add-ons stack, but it is **not used** here - there are no `Ingress` resources wired to it in this PoC.


## Cost-Optimized PoC Limitations

This `POC` intentionally prioritizes cost over full production hardening.

- Single AWS account and single EKS cluster host both tooling and app workloads.
- Fixed-size node groups (`1/1/1`) and no Cluster Autoscaler/Karpenter.
- Single NAT Gateway (cost-optimized, lower AZ-failure resilience).
- Jenkins/Argo CD UI access via port-forward (no private ingress stack yet).
- Docker Hub is used as image registry instead of ECR.
- No observability stack.
- EBS/EFS CSI drivers are installed, but persistent storage patterns are intentionally minimal in this phase.

## Production considerations

The sections below describe **production-grade** design considerations for this platform implementation.

**Topics**

- [Environment and account strategy](#prod-environment-and-account-strategy)
- [Private enterprise access to Jenkins and Argo CD](#prod-private-enterprise-access-to-jenkins-and-argo-cd)
- [Cluster autoscaling and node-group strategy](#prod-cluster-autoscaling-and-node-group-strategy)
- [Secrets management for Jenkins and the platform](#prod-secrets-management-for-jenkins-and-the-platform)
- [Persistent storage and Jenkins state](#prod-persistent-storage-and-jenkins-state)
- [Container registry](#prod-container-registry)
- [Monitoring and observability](#prod-monitoring-and-observability)

---

### Prod: Environment and account strategy

For production, use **separate AWS accounts** (at minimum: `dev/test`, `staging`, `prod`; and a dedicated `shared-services/devops` account).

- Run Argo CD in a management cluster/account.
- Register workload clusters from other accounts in Argo CD.
- Use Argo CD to deploy to multiple clusters from one control point.

This model is a **Centralized Multi-Cluster Argo CD (Hub-and-Spoke) architecture**:
- **Hub:** management/devops cluster (Argo CD control plane)
- **Spokes:** workload clusters per environment/account

This reduces blast radius, enforces stronger separation of duties, and aligns with enterprise governance.

---

### Prod: Private enterprise access to Jenkins and Argo CD

Recommended production pattern for office/private access:

- Deploy **Ingress** for Jenkins and Argo CD with **host-based routing** (`jenkins.<domain>`, `argocd.<domain>`).
- Use AWS Load Balancer Controller with an **internal ALB** (private subnets).
- Manage DNS with **Route 53 private hosted zone**.
- Terminate TLS with **ACM certificates**.
- Provide network path from office to VPC (e.g using **Site-to-Site VPN**)
- Enforce access controls (e.g IP allowlists).

---

### Prod: Cluster autoscaling and node-group strategy

In production, install **Cluster Autoscaler** (or Karpenter where appropriate):

- Keep platform-critical group (`jenkins`) with safe minimum capacity.
- Allow `jenkins-agents` and `github-runners` groups to **scale to zero** when idle.
- Scale `podinfo-app` independently based on workload demand.

This reduces cost while preserving platform reliability and performance isolation.

---

### Prod: Secrets management for Jenkins and the platform

Do not store secrets in Jenkins internal DB.

Recommended approach:

- Store secrets in **AWS Secrets Manager** (or SSM Parameter Store for specific use-cases).
- Use **IRSA** so Jenkins and controllers read only the secrets they need.
- Use **External Secrets Operator** (or Secrets Store CSI) to sync runtime Kubernetes secrets from AWS.
- Keep Jenkins Configuration as Code (JCasC) in Git, with only **secret references**, never secret values.
- Rotate secrets centrally and audit access via AWS APIs/CloudTrail.

---

### Prod: Persistent storage and Jenkins state

What should persist in a mature setup:

- Jenkins configuration baseline (Jenkins Configuration as Code (JCasC) in Git)
- Job history/queue metadata (if retained)
- Build logs/artifacts (prefer external object storage)

Recommended pattern:

- For shared/multi-AZ file persistence, prefer **EFS** for Jenkins home if persistence is required.
- Use **EBS** only when single-AZ constraints are acceptable and documented.
- Protect state with **AWS Backup/EBS snapshots**, tested restore runbooks, and periodic recovery drills.

Practical note on AZ failure:
- A single EBS volume is AZ-bound. If the AZ is unavailable, direct failover requires recovery from snapshot to a volume in another AZ.
- EFS is regional and multi-AZ, which generally simplifies recovery objectives for stateful controllers.

---

### Prod: Container registry

Move from Docker Hub to **Amazon ECR** in production:

- Private registry integrated with IAM and CloudTrail
- Better enterprise control (lifecycle policies, scanning, permissions)
- Use **IRSA** for Jenkins agents and GitHub runners to pull/push images

---

### Prod: Monitoring and observability

Add an end-to-end observability stack:

- **Metrics:** Prometheus + Alertmanager (or AWS Managed Prometheus)
- **Dashboards:** Grafana (or AWS Managed Grafana)
- **Logs:** CloudWatch / Loki
- **Kubernetes telemetry:** cluster, node, pod, API server, scheduler, etcd, CNI
- **Platform:** Jenkins metrics (e.g. queue size, running executors, job success/failure rates, build duration for slow-stage analysis). Argo CD metrics (e.g. sync/health/drift, reconcile time).
- **Alerting:** paging for availability and delivery failures, actionable notifications by team ownership

---

## Appendix: Setup of self-hosted GitHub runners on Kubernetes (ARC)

<details>
<summary><strong>Expand: ARC setup steps</strong></summary>

<br />

This appendix documents the **manual** setup path for **self-hosted GitHub runners** on Kubernetes using `Actions Runner Controller (ARC)`.

Actions Runner Controller (ARC) is a Kubernetes operator that orchestrates and scales self-hosted runners for GitHub Actions.

> Note: For production environments, install and manage ARC through **automation** (e.g Terraform/GitOps).

> Note: GitHub recommends using self-hosted runners only for private repos, since fork PRs can run untrusted code. Here, app-source is public (PoC), so only approved contributors can open fork-based PRs.

### Create a GitHub App for ARC

ARC must [authenticate](https://docs.github.com/en/actions/how-tos/manage-runners/use-actions-runner-controller/authenticate-to-the-api#authenticating-arc-with-a-github-app) to the GitHub API.

1. Go to Settings -> Developer settings -> GitHub Apps -> New GitHub App.
2. Configure:
   - **Homepage URL**: `https://github.com/actions/actions-runner-controller`
   - Under **Repository permissions**:
     - **Administration**: Read and write (required for repository-scope runner registration)
   - Under **Organization permissions**:
     - **Self-hosted runners**: Read and write
   - **Webhook**: disable by unchecking **Active**
3. Create the app, then copy the **App ID**.
4. Under **Private keys**, click **Generate a private key** and download the `.pem` file.
5. Install the app to the target repository (here: [app-source](https://github.com/dana951/app-source)) from **Install app**.
6. Copy the **Installation ID** from the browser:
   `https://github.com/settings/installations/<installation_id>`

### ARC installation

Need to install two helm charts:

1. `gha-runner-scale-set-controller`
2. `gha-runner-scale-set`

Reference: [actions/actions-runner-controller](https://github.com/actions/actions-runner-controller)

Before installation, ensure the runner namespace already exists (e.g: `github-runners`).  
This is where self-hosted runner pods are created.

#### 1) Create the GitHub App secret in Kubernetes

Register the App ID, Installation ID, and private key as a Kubernetes secret:

```bash
kubectl create secret generic gha-runner-scale-set-secret \
  --namespace=github-runners \
  --from-literal=github_app_id=<your-github-app-id> \
  --from-literal=github_app_installation_id=<your-github-app-installation-id> \
  --from-file=github_app_private_key=<path-to-github-app-private-key.pem>
```

Set this in `gha-runner-scale-set` values:

```yaml
githubConfigSecret: gha-runner-scale-set-secret
```

#### 2) Install `gha-runner-scale-set-controller`

Example (controller installed in `jenkins` namespace, chart version `0.14.0`):

```bash
helm install github-arc \
  --namespace jenkins \
  --create-namespace \
  --version 0.14.0 \
  oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set-controller
```

#### 3) Install `gha-runner-scale-set`

Install into the runner namespace (`github-runners` in this example):

```bash
helm install eks-gh-runner \
  --namespace github-runners \
  --create-namespace \
  --version 0.14.0 \
  --set githubConfigUrl=https://github.com/dana951/app-source \
  --set githubConfigSecret=gha-runner-scale-set-secret \
  oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set
```

> Note: the release name you use here (e.g eks-gh-runner) is the "runs_on" values you will set in the github actions workflow

### Runner image update warning ([official GitHub guidance](https://docs.github.com/en/actions/reference/runners/self-hosted-runners#runner-software-updates-on-self-hosted-runners))

GitHub warns that self-hosted runners must stay current:

> Any updates released for the software, including major, minor, or patch releases, are considered as an available update. If you do not perform a software update within 30 days, the GitHub Actions service will not queue jobs to your runner. In addition, if a critical security update is required, the GitHub Actions service will not queue jobs to your runner until it has been updated.

</details>

---

## License

Each repository contains its own license file. See `LICENSE` in the relevant repository.
