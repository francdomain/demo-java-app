# DevOps Platform — Complete Implementation Guide

A complete CI/CD platform built on AWS EKS, ArgoCD GitOps, and GitHub Actions reusable workflows. This workspace contains four independent Git repositories that together provision infrastructure, build容器 images, and deploy a Spring Boot application across three environments.

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Repository Map](#repository-map)
3. [Prerequisites](#prerequisites)
4. [Phase 1 — Infrastructure (Terraform)](#phase-1--infrastructure-terraform)
5. [Phase 2 — GitOps Repository](#phase-2--gitops-repository)
6. [Phase 3 — CI/CD Pipeline Templates](#phase-3--cicd-pipeline-templates)
7. [Phase 4 — Application Repository](#phase-4--application-repository)
8. [Deployment Flow by Environment](#deployment-flow-by-environment)
9. [Secrets & Variables Reference](#secrets--variables-reference)
10. [Accessing ArgoCD & Applications](#accessing-argocd--applications)
11. [Operational Runbook](#operational-runbook)
12. [Troubleshooting Guide](#troubleshooting-guide)
13. [Architecture Decisions](#architecture-decisions)

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                        GitHub Actions                                │
│  demo-java-app  ──►  reusable-pipeline-templates  ──►  Docker Hub  │
│      │                        │                                      │
│      │   GitOps commit        │                                      │
│      ▼                        │                                      │
│  gitops-manifests  ◄──────────┘                                      │
│      │                                                               │
│      │   ArgoCD auto-sync                                           │
│      ▼                                                               │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                    AWS EKS Cluster                            │   │
│  │  ┌──────────────────────────────────────────────────────┐    │   │
│  │  │  demo-java-app-development  │  demo-java-app-uat    │    │   │
│  │  │  demo-java-app-production   │                       │    │   │
│  │  └──────────────────────────────────────────────────────┘    │   │
│  │                        ^                                      │   │
│  │                        │                                      │   │
│  │                   ArgoCD Server                               │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                        ▲                                              │
│                        │                                              │
│              terraform-aws-eks-argocd (EKS + ArgoCD)                 │
└─────────────────────────────────────────────────────────────────────┘
```

### Deployment Flow

1. Developer pushes code to `demo-java-app` (feature branch, main, or release branch)
2. GitHub Actions triggers the appropriate reusable workflow
3. Pipeline builds JAR, runs tests, builds Docker image, pushes to Docker Hub
4. Pipeline clones `gitops-manifests`, updates the image tag in the target environment's values file
5. ArgoCD detects the GitOps change and auto-syncs the application
6. Pipeline optionally triggers explicit sync and polls for rollout health (dev and UAT only)

---

## Repository Map

| Directory | Role | Stack | Consumable Branch |
|-----------|------|-------|------------------|
| [`demo-java-app/`](./demo-java-app/) | Spring Boot web app + Docker image source | Java 17, Maven, Spring Boot 3.2.5, Thymeleaf | — |
| [`reusable-pipeline-templates/`](./reusable-pipeline-templates/) | Centralized GitHub Actions reusable workflows | GitHub Actions | `shared/java` |
| [`gitops-manifests/`](./gitops-manifests/) | Helm charts per environment consumed by ArgoCD | Helm, Kubernetes | `main` |
| [`terraform-aws-eks-argocd/`](./terraform-aws-eks-argocd/) | Infrastructure: AWS VPC → EKS → ArgoCD | Terraform >= 1.5 | — |

**Important:** The root directory is **not a Git repository**. Each subdirectory is an independent repository.

---

## Prerequisites

Before starting, ensure you have:

### Tools

| Tool | Version | Purpose |
|------|---------|---------|
| Terraform | >= 1.5 | Infrastructure provisioning |
| AWS CLI | latest | AWS authentication |
| kubectl | >= 1.28 | Kubernetes cluster access |
| Helm | >= 3.12 | Chart validation |
| ArgoCD CLI | >= 2.11 | Token generation and server access |
| Java | 17 | Local app development |
| Maven | 3.9+ | Build tool |
| Docker | latest | Local container builds |

### Accounts & Access

- **AWS Account** with credentials configured (`aws configure`)
- **GitHub Account** with access to create organizations/repos
- **Docker Hub Account** (or alternative container registry)
- **GitOps Personal Access Token (PAT)** with write access to `gitops-manifests`

### GitHub Organization Setup

Create or ensure these repositories exist:
- `your-org/demo-java-app`
- `your-org/reusable-pipeline-templates`
- `your-org/gitops-manifests`

> Replace `your-org` with your actual GitHub organization or username throughout this guide.

---

## Phase 1 — Infrastructure (Terraform)

This phase provisions AWS infrastructure (VPC, EKS, ArgoCD) using the [`terraform-aws-eks-argocd`](./terraform-aws-eks-argocd/) repository.

### 1.1 Backend Setup — S3 + DynamoDB for Terraform State

The backend setup uses **local state** (checked into git). This must be applied first.

```bash
cd terraform-aws-eks-argocd/backend-setup
terraform init
terraform apply
```

**What it creates:**
- S3 bucket `terraform-state-francdomain` (name may vary — check `main.tf`)
- DynamoDB table `terraform-locks` for state locking
- Both in `us-east-1`

> **Do not delete** `terraform.tfstate` in this directory.

### 1.2 VPC + EKS Cluster

```bash
cd terraform-aws-eks-argocd/environments/dev-infra
terraform init
terraform apply
```

**What it creates:**
- VPC with 3 AZs, public + private subnets, NAT gateways
- EKS cluster v1.30 with managed node group (t3.medium, 2–4 nodes)
- Kubernetes-specific subnet tagging for ELB discovery

**Required variables** (`terraform.tfvars`):
```hcl
aws_region      = "us-east-1"
environment     = "dev"
cluster_name    = "dev-eks"
cluster_version = "1.30"
node_instance_types = ["t3.medium"]
desired_capacity = 2
min_size         = 2
max_size         = 4
vpc_cidr         = "10.0.0.0/16"
azs              = ["us-east-1a", "us-east-1b", "us-east-1c"]
private_subnets  = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
public_subnets   = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]
```

### 1.3 ArgoCD + Bootstrap Application

Create `terraform.tfvars` in `environments/dev-k8s/`:

```hcl
cluster_name           = "dev-eks"  # Must match dev-infra cluster name
gitops_repo_url        = "https://github.com/your-org/gitops-manifests.git"
gitops_pat             = "ghp_xxxxxxxxxxxxxxxxxxxx"  # Your GitHub PAT
gitops_target_revision = "main"
```

```bash
cd terraform-aws-eks-argocd/environments/dev-k8s
terraform init
terraform apply
```

**What it creates:**
- ArgoCD v7.3.11 exposed via AWS LoadBalancer (HTTP port 80)
- Repository secret for GitOps repo access
- Bootstrap Application pointing to `argocd/` path in GitOps repo

### 1.4 Configure kubectl Access

```bash
aws eks update-kubeconfig --region us-east-1 --name dev-eks
```

### 1.5 Retrieve ArgoCD Credentials

```bash
cd terraform-aws-eks-argocd/environments/dev-k8s

# ArgoCD server hostname
terraform output -raw argocd_server_url
# Example: afd645bd6a92b477b9537913bd3e87b9-1404881483.us-east-1.elb.amazonaws.com

# Initial admin password
terraform output -raw argocd_initial_admin_password
```

### 1.6 Generate ArgoCD API Token

The pipeline needs an API token (not the admin password) to trigger syncs.

**Step 1:** Enable `apiKey` capability for the admin account:
```bash
kubectl patch configmap argocd-cm -n argocd --type merge \
  -p '{"data":{"accounts.admin":"apiKey, login"}}'

kubectl rollout restart deployment argocd-server -n argocd
kubectl rollout status deployment argocd-server -n argocd
```

**Step 2:** Log in via CLI:
```bash
# Get the hostname
ARGOCD_HOST=$(terraform output -raw argocd_server_url)
ADMIN_PWD=$(terraform output -raw argocd_initial_admin_password)

argocd login "$ARGOCD_HOST" \
  --username admin \
  --password "$ADMIN_PWD" \
  --grpc-web \
  --insecure
```

**Step 3:** Generate the token:
```bash
argocd account generate-token --account admin --grpc-web --insecure
```

**Step 4:** Copy the token and save it as `ARGOCD_AUTH_TOKEN` in GitHub secrets (see [Secrets & Variables Reference](#secrets--variables-reference)).

### 1.7 Verify ArgoCD Bootstrap

```bash
kubectl get applications -n argocd
```

You should see:
```
NAME        SYNC STATUS   HEALTH STATUS
bootstrap   Synced        Healthy
```

If the `bootstrap` app does not exist, force sync manually:
```bash
argocd app sync bootstrap --server localhost:8080 --insecure
```

### 1.8 Enable Directory Recursion (Critical Fix)

The bootstrap Application must recurse into subdirectories to discover the `argocd/appsets/` folder.

```bash
kubectl patch application bootstrap -n argocd --type merge \
  -p '{"spec":{"source":{"directory":{"recurse":true}}}}'

argocd app sync bootstrap --server localhost:8080 --insecure
```

After sync, verify the ApplicationSet exists:
```bash
kubectl get applicationset -n argocd
# Output: microservices
```

And verify the environment applications:
```bash
kubectl get applications -n argocd
# Output:
# NAME                        SYNC STATUS   HEALTH STATUS
# bootstrap                   Synced        Healthy
# demo-java-app-development   Synced        Healthy
# demo-java-app-production    Synced        Progressing
# demo-java-app-uat           Synced        Healthy
```

> **Strict deployment order matters:** `backend-setup` → `dev-infra` → `dev-k8s`. `dev-k8s` uses a `data.aws_eks_cluster` lookup that fails if the cluster doesn't exist yet.

---

## Phase 2 — GitOps Repository

This phase sets up the [`gitops-manifests/`](./gitops-manifests/) repository that ArgoCD watches.

### 2.1 Repository Structure

```
gitops-manifests/
├── argocd/
│   └── appsets/
│       └── microservices.yaml      # ApplicationSet for all environments
├── development/                      # Helm chart for Development
│   ├── Chart.yaml
│   ├── values.yaml
│   ├── values/demo-java-app.yaml
│   └── templates/
│       ├── _helpers.tpl
│       ├── deployment.yaml
│       └── service.yaml
├── uat/                              # Helm chart for UAT
│   ├── Chart.yaml
│   ├── values.yaml
│   ├── values/demo-java-app.yaml
│   └── templates/
│       ├── _helpers.tpl
│       ├── deployment.yaml
│       └── service.yaml
└── production/                       # Helm chart for Production
    ├── Chart.yaml
    ├── values.yaml
    ├── values/demo-java-app.yaml
    └── templates/
        ├── _helpers.tpl
        ├── deployment.yaml
        └── service.yaml
```

### 2.2 ApplicationSet Configuration

The [`argocd/appsets/microservices.yaml`](./gitops-manifests/argocd/appsets/microservices.yaml) uses a `list` generator to create three ArgoCD Applications:

| Generated Name | Source Path | Namespace | Values File |
|---------------|-------------|-----------|-------------|
| `demo-java-app-development` | `development/` | `demo-java-app-development` | `values/demo-java-app.yaml` |
| `demo-java-app-uat` | `uat/` | `demo-java-app-uat` | `values/demo-java-app.yaml` |
| `demo-java-app-production` | `production/` | `demo-java-app-production` | `values/demo-java-app.yaml` |

### 2.3 Helm Chart Per Environment

Each environment is a self-contained Helm chart. All three share **identical templates**.

**Key files:**
- `Chart.yaml` — Chart metadata (name matches environment)
- `values.yaml` — Base defaults (replicaCount, service type, resources)
- `values/demo-java-app.yaml` — Deployment-specific image tag and overrides
- `templates/deployment.yaml` — Kubernetes Deployment
- `templates/service.yaml` — Kubernetes Service (type: LoadBalancer)

**Critical conventions:**
- **Template edits must be mirrored** across all three environments. Keep `development/templates/`, `uat/templates/`, and `production/templates/` identical.
- **Always use immutable image tags** — never `latest`. The deployment values file enforces this with `tag: 'placeholder'`.
- **Base `values.yaml` uses `tag: "latest"`** but this is overridden by `values/demo-java-app.yaml`.

### 2.4 Local Validation

Before committing changes, validate the Helm chart:

```bash
cd gitops-manifests
helm template demo-java-app ./development -f ./development/values/demo-java-app.yaml
```

### 2.5 Deployment via Pipeline

The CI/CD pipeline automatically updates `values/demo-java-app.yaml` for the target environment:

| Trigger Branch | Environment | Values Path Updated |
|---------------|-------------|-------------------|
| `feature/*` | Development | `development/values/demo-java-app.yaml` |
| `main` | UAT | `uat/values/demo-java-app.yaml` |
| `release/*` | Production | `production/values/demo-java-app.yaml` |

The pipeline commits the new `image.tag` to `gitops-manifests` on the `main` branch. ArgoCD auto-syncs within ~30 seconds.

---

## Phase 3 — CI/CD Pipeline Templates

This phase configures the [`reusable-pipeline-templates/`](./reusable-pipeline-templates/) repository.

### 3.1 Branch Layout

| Branch | Purpose | External Callers |
|--------|---------|-----------------|
| `shared/java` | Active consumable branch for Java/Maven | ✅ Yes |
| `shared/javascript` | Future: Node.js pipelines | 🔮 Planned |
| `main` | Placeholder only; **not for consumption** | ❌ No |

> **Critical:** External repositories **must** reference `@shared/java`, not `@main`.

### 3.2 Workflow Architecture

```
External Callers                               Core Pipeline
┌─────────────────────────┐                   ┌──────────────────────────┐
│ java-on-feature-branch  │────┐         ┌───►│ java-shared-pipeline.yml │
│ java-on-main-branch     │────┤         ┤    │  (boolean toggles)       │
│ java-on-release-branch  │────┼─────────┼────│                          │
│ java-on-pull-request    │────┤         ┤    │ Jobs:                    │
│ java-on-any-branch      │────┘         └────│  - changes-check         │
└─────────────────────────┘                   │  - gitleaks              │
                                              │  - dependency-check      │
Root CI (this repo only)                      │  - build-test            │
┌─────────────────────────┐                   │  - sonar                 │
│ default.yml             │                   │  - veracode-scan         │
│ feature.yml             │                   │  - package-app           │
│ main.yml                │                   │  - docker-build-push     │
│ pull-request.yml        │                   │  - deploy                │
└─────────────────────────┘                   └──────────────────────────┘
```

### 3.3 External Caller Entrypoints

| Entrypoint | Trigger | gitleaks | dep-check | build | sonar | veracode | package | docker | deploy | deploy target |
|-----------|---------|----------|-----------|-------|-------|----------|---------|--------|--------|---------------|
| `java-on-feature-branch.yml` | `feature/*` push | ❌ | ❌ | ✅ | ❌ | ❌ | ✅ | ✅ | ✅ | Development |
| `java-on-main-branch-update.yml` | `main` push | ✅ | ❌ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | UAT |
| `java-on-release-branch.yml` | `release/*` push | ✅ | ❌ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | Production |
| `java-on-pull-request.yml` | PRs | ✅ | ❌ | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ | none |
| `java-on-any-branch-update.yml` | Any other push | ❌ | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | none |

### 3.4 Changeset Filtering

A `changes-check` job at the start gates expensive jobs. If **every** changed file matches documentation patterns (`README*`, `*.md`, `docs/**`, `.github/**`, `LICENSE*`, etc.), then `dependency-check`, `build-test`, `package-app`, and `docker-build-push` are all skipped.

`gitleaks`, `sonar`, and `veracode` run regardless of changeset.

### 3.5 Scheduled Security Scan

[`java-scheduled-security-scan.yml`](./reusable-pipeline-templates/.github/workflows/java-scheduled-security-scan.yml) runs nightly at 02:00 UTC:

- OWASP Dependency Check (intentionally **disabled** in main/release pipelines due to NVD API timeouts)
- Gitleaks
- SonarCloud
- Veracode

### 3.6 Deployment Job Details

The `deploy` job in `java-shared-pipeline.yml`:
1. Clones the GitOps repo using `GITOPS_TOKEN`
2. Pulls latest changes (to avoid rebase conflicts)
3. Updates `image.repository` and `image.tag` in the target values file via `sed`
4. Commits and pushes changes to the GitOps repo
5. Optionally triggers ArgoCD sync via REST API (`ARGOCD_AUTH_TOKEN`)
6. Optionally polls ArgoCD app health for up to 600 seconds (`rollout-enabled: true`)

**Deployment constraints:**
- `rollout-enabled: true` only on `feature/*` and `main` (dev and UAT)
- `rollout-enabled: false` on `release/*` (production)
- ArgoCD sync is triggered with `POST http://<server>/api/v1/applications/<app-name>/sync`

### 3.7 Updating a Workflow

1. Edit the relevant `.yml` files on the `shared/java` branch
2. Commit and push to `shared/java`
3. Changes take effect immediately on the next pipeline run

> Do **not** edit workflows on `main` — that branch is not consumed by external repos.

---

## Phase 4 — Application Repository

This phase configures the [`demo-java-app/`](./demo-java-app/) repository that developers work in.

### 4.1 Project Structure

```
demo-java-app/
├── .github/workflows/
│   ├── feature.yml          # Build & deploy to dev
│   ├── main.yml             # Build & deploy to UAT
│   ├── pull-request.yml     # CI checks on PR
│   └── release.yml          # Build & deploy to prod
├── src/
│   ├── main/java/com/example/demo/
│   │   ├── DemoApplication.java   # Entry point
│   │   └── HomeController.java    # GET /
│   ├── main/resources/templates/
│   │   └── index.html             # Thymeleaf template
│   └── test/java/com/example/demo/
│       └── HomeControllerTest.java
├── Dockerfile
├── pom.xml
└── README.md
```

### 4.2 Technology Stack

- **Java 17**, **Maven 3.9+**
- **Spring Boot 3.2.5** with Thymeleaf
- **JaCoCo** for coverage (runs during `mvn test`)
- **SonarCloud** for code quality
- **Docker** multi-stage build (temurin-17 → jre-alpine)

### 4.3 Build Commands

```bash
# Run tests with coverage
mvn test

# Package JAR (skip tests)
mvn clean package -DskipTests

# Run locally
mvn spring-boot:run
# Access at http://localhost:8080
```

### 4.4 Branch Naming Convention

**Enforced by CI.** PR branches **must** use one of:
- `feature/*`
- `release/*`
- `hotfix/*`
- `bugfix/*`

Invalid prefixes cause the `check-branch-name` job to fail with exit 1.

### 4.5 Workflow Configuration

All four workflows delegate to the reusable pipeline. Key parameters:

[`feature.yml`](./demo-java-app/.github/workflows/feature.yml):
```yaml
gitops-values-path: "development/values/demo-java-app.yaml"
argocd-app-name: "demo-java-app-development"
rollout-enabled: true
```

[`main.yml`](./demo-java-app/.github/workflows/main.yml):
```yaml
gitops-values-path: "uat/values/demo-java-app.yaml"
argocd-app-name: "demo-java-app-uat"
rollout-enabled: true
```

[`release.yml`](./demo-java-app/.github/workflows/release.yml):
```yaml
gitops-values-path: "production/values/demo-java-app.yaml"
argocd-app-name: "demo-java-app-production"
rollout-enabled: false
```

> Production deployment **updates the GitOps values file** but does **not** trigger an explicit ArgoCD sync or rollout check. Auto-sync from ArgoCD handles the deployment.

### 4.6 Setting Up a New Application

To onboard a new Java application:

1. Create a new repo under `your-org/`
2. Copy `demo-java-app/.github/workflows/` to the new repo
3. Update these values in each workflow:
   - `docker-image-name`: `"your-org/your-app-name"`
   - `gitops-values-path`: `"<env>/values/your-app.yaml"`
   - `argocd-app-name`: `"your-app-<env>"`
4. Ensure the ApplicationSet in `gitops-manifests` generates the matching ArgoCD Application name

---

## Deployment Flow by Environment

### Development (`feature/*` branch)

```
Developer pushes feature/my-work
        │
        ▼
┌──────────────────┐
│ GitHub Actions   │
│ - Build & test   │
│ - Docker push    │
│ - Update GitOps  │
│ - ArgoCD sync    │
│ - Rollout check  │
└──────────────────┘
        │
        ▼
Commit to gitops-manifests/main:
  development/values/demo-java-app.yaml
  image.tag: e4b6b02
        │
        ▼
ArgoCD auto-syncs demo-java-app-development
Namespace: demo-java-app-development
```

**Pipeline jobs:** gitleaks ❌ | dep-check ❌ | build ✅ | sonar ❌ | veracode ❌ | package ✅ | docker ✅ | deploy ✅ | sync ✅ | rollout ✅

### UAT (`main` branch)

```
Developer pushes to main
        │
        ▼
┌──────────────────┐
│ GitHub Actions   │
│ - gitleaks       │
│ - Build & test   │
│ - Sonar + Vera   │
│ - Docker push    │
│ - Update GitOps  │
│ - ArgoCD sync    │
│ - Rollout check  │
└──────────────────┘
        │
        ▼
Commit to gitops-manifests/main:
  uat/values/demo-java-app.yaml
  image.tag: 9bd0952
        │
        ▼
ArgoCD auto-syncs demo-java-app-uat
Namespace: demo-java-app-uat
```

**Pipeline jobs:** gitleaks ✅ | dep-check ❌ | build ✅ | sonar ✅ | veracode ✅ | package ✅ | docker ✅ | deploy ✅ | sync ✅ | rollout ✅

### Production (`release/*` branch)

```
Developer pushes release/v1.2.0
        │
        ▼
┌──────────────────┐
│ GitHub Actions   │
│ - gitleaks       │
│ - Build & test   │
│ - Sonar + Vera   │
│ - Docker push    │
│ - Update GitOps  │
│ - No sync trigger│
│ - No rollout chk │
└──────────────────┘
        │
        ▼
Commit to gitops-manifests/main:
  production/values/demo-java-app.yaml
  image.tag: aec8cab
        │
        ▼
ArgoCD auto-syncs demo-java-app-production
Namespace: demo-java-app-production
```

**Pipeline jobs:** gitleaks ✅ | dep-check ❌ | build ✅ | sonar ✅ | veracode ✅ | package ✅ | docker ✅ | deploy ✅ | sync ❌ | rollout ❌

---

## Secrets & Variables Reference

### GitHub Repository Variables

Set these in **Settings → Secrets and variables → Actions → Variables** (can be organization-level):

| Variable | Value Example | Used In |
|----------|--------------|---------|
| `SONAR_PROJECT_KEY` | `francdomain` | `demo-java-app` workflows |
| `SONAR_ORGANIZATION` | `francdomain` | `demo-java-app` workflows |
| `ARGOCD_SERVER` | `afd645bd6a92b477b9537913bd3e87b9-1404881483.us-east-1.elb.amazonaws.com` | `demo-java-app` workflows |

### GitHub Secrets

Set these in **Settings → Secrets and variables → Actions → Secrets**:

| Secret | Purpose | Required By |
|--------|---------|-------------|
| `SONAR_TOKEN` | SonarCloud analysis | All pipelines with `run-sonar: true` |
| `SONAR_SCANNER_OPTS` | Sonar scanner options | SonarCloud step |
| `GITOPS_TOKEN` | PAT to commit to `gitops-manifests` | Deploy job |
| `ARGOCD_AUTH_TOKEN` | Token to trigger ArgoCD sync | Sync + rollout steps |
| `DOCKER_REGISTRY_USERNAME` | Docker Hub username | Docker build & push |
| `DOCKER_REGISTRY_PASSWORD` | Docker Hub password | Docker build & push |
| `VERACODE_API_ID` | Veracode API ID | Optional: veracode-scan |
| `VERACODE_API_KEY` | Veracode API key | Optional: veracode-scan |
| `NVD_API_KEY` | NVD API key for OWASP | Optional: scheduled scan |

### Terraform Variables

`environments/dev-k8s/terraform.tfvars`:

| Variable | Required | Notes |
|----------|----------|-------|
| `cluster_name` | ✅ | Must match `dev-infra` output |
| `gitops_repo_url` | ✅ | Example: `https://github.com/your-org/gitops-manifests.git` |
| `gitops_pat` | ✅ | GitHub PAT with repo scope |
| `gitops_target_revision` | ✅ | Default: `main` |

### ArgoCD RBAC

After initial Terraform apply, patch these ConfigMaps:

```bash
# Enable admin apiKey
echo '{"data":{"accounts.admin":"apiKey, login"}}' | kubectl patch configmap argocd-cm -n argocd --type merge -p @-

# Set default RBAC role
echo '{"data":{"policy.default":"role:admin"}}' | kubectl patch configmap argocd-rbac-cm -n argocd --type merge -p @-

# Restart server
kubectl rollout restart deployment argocd-server -n argocd
```

---

## Accessing ArgoCD & Applications

### ArgoCD UI

```bash
# Get server hostname
kubectl get svc argocd-server -n argocd -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'

# Open in browser (HTTP, not HTTPS):
# http://<hostname>
# Login: admin / <terraform output password>
```

### Application URLs

Each environment exposes the app via a LoadBalancer service:

```bash
# Development
kubectl get svc demo-java-app-development -n demo-java-app-development \
  -o jsonpath='http://{.status.loadBalancer.ingress[0].hostname}:8080'

# UAT
kubectl get svc demo-java-app-uat -n demo-java-app-uat \
  -o jsonpath='http://{.status.loadBalancer.ingress[0].hostname}:8080'

# Production
kubectl get svc demo-java-app-production -n demo-java-app-production \
  -o jsonpath='http://{.status.loadBalancer.ingress[0].hostname}:8080'
```

---

## Operational Runbook

### Adding a New Environment

1. Create a new directory in `gitops-manifests/` (e.g., `staging/`)
2. Copy an existing environment's complete structure:
   ```bash
   cp -r gitops-manifests/development/* gitops-manifests/staging/
   ```
3. Update `Chart.yaml` → change `name:` to `staging`
4. Update `values.yaml` → set `replicaCount`, `service.type`, etc.
5. Create `values/demo-java-app.yaml` with base image settings and `tag: 'placeholder'`
6. Update `gitops-manifests/argocd/appsets/microservices.yaml` → add `staging` to the `list` generator
7. Commit and push to `gitops-manifests/main`
8. ArgoCD will auto-create `demo-java-app-staging`
9. Update the reusable workflow entrypoint (if needed) to point to `staging/values/demo-java-app.yaml`

### Adding a New Application

1. Copy `demo-java-app/` structure to a new repo
2. Update `Dockerfile` and `pom.xml` as needed
3. Copy `.github/workflows/` from `demo-java-app`
4. Update workflow inputs:
   - `docker-image-name`
   - `gitops-values-path`
   - `argocd-app-name`
5. Create the corresponding `values/<app-name>.yaml` files in each environment directory in `gitops-manifests`
6. Update the ApplicationSet template to reference the new app (or create a new ApplicationSet)
7. Commit and push both repos

### Rolling Back a Deployment

**Via Git:**
```bash
cd gitops-manifests
git log --oneline
git revert <commit-hash-that-updated-tag>
git push origin main
```

ArgoCD will auto-sync to the previous tag.

**Via ArgoCD UI:**
1. Open the application in ArgoCD
2. Click "History and Rollback"
3. Select the previous revision and click "Sync"

### Viewing Application Logs

```bash
# Development
kubectl logs -n demo-java-app-development -l app.kubernetes.io/instance=demo-java-app-development --tail=100 -f

# UAT
kubectl logs -n demo-java-app-uat -l app.kubernetes.io/instance=demo-java-app-uat --tail=100 -f
```

### Manual ArgoCD Sync

```bash
argocd app sync demo-java-app-development --server <hostname> --insecure
argocd app sync demo-java-app-uat --server <hostname> --insecure
argocd app sync demo-java-app-production --server <hostname> --insecure
```

---

## Troubleshooting Guide

### GitOps pull rebase failed — "You have unstaged changes"

**Error:**
```
error: cannot pull with rebase: You have unstaged changes.
```

**Cause:** The deploy job runs `sed` to modify `values.yaml` before `git pull --rebase`.

**Fix:** Move `git pull --rebase` to execute **before** any local file modifications. This was fixed in the shared pipeline.

### OWASP Dependency Check is slow / times out

**Cause:** The NVD API download takes 8–15+ minutes per run.

**Fix:** OWASP Dependency Check is **disabled** in `java-on-main-branch-update.yml` and `java-on-release-branch.yml`. It runs only in the nightly scheduled scan (`java-scheduled-security-scan.yml` at 02:00 UTC). Do not re-enable it in deployment pipelines.

### ArgoCD sync returns 403 — "permission denied"

**Possible causes:**
1. **Application doesn't exist** — check `kubectl get applications -n argocd`
2. **Admin account lacks `apiKey` capability** — patch `argocd-cm` ConfigMap
3. **RBAC default policy is empty** — patch `argocd-rbac-cm` to set `policy.default: role:admin`
4. **Wrong application name** — must match the ApplicationSet-generated name (e.g., `demo-java-app-development`)

**Diagnosis commands:**
```bash
kubectl get applications -n argocd
kubectl get configmap argocd-cm -n argocd -o yaml
kubectl get configmap argocd-rbac-cm -n argocd -o yaml
```

### ArgoCD sync returns SSL error

**Error:** `curl: (35) OpenSSL SSL_connect: SSL_ERROR_SYSCALL`

**Cause:** Pipeline calls `https://` but ArgoCD server is configured with `insecure: true` (HTTP only on port 80).

**Fix:** Change the pipeline curl URL from `https://` to `http://`.

### ApplicationSet not creating applications

**Cause:** The `directories` generator in the ApplicationSet matches subdirectories, not the chart root.

**Fix:** Use a `list` generator with explicit elements instead of `directories` with `*` wildcards.

### Bootstrap app doesn't discover ApplicationSet

**Cause:** The `bootstrap` Application only syncs files directly in `argocd/`, not subdirectories.

**Fix:** Enable directory recursion:
```bash
kubectl patch application bootstrap -n argocd --type merge \
  -p '{"spec":{"source":{"directory":{"recurse":true}}}}'
```

### Branch name check fails on PR

**Error:** `Branch 'my-branch' does not follow naming convention. Allowed prefixes: feature/*, release/*, hotfix/*, bugfix/*`

**Fix:** Rename the branch to use one of the required prefixes.

### Docker build fails — "no Maven wrapper"

**Cause:** `demo-java-app` does not include `mvnw`. The build requires Maven to be installed on the runner.

**Fix:** This repo assumes the runner or container has Maven available. The Dockerfile uses `maven:3.9.6-eclipse-temurin-17` as the build stage.

---

## Architecture Decisions

| Decision | Rationale |
|----------|-----------|
| **Four independent repos** | Clean separation of concerns; allows independent versioning and access control |
| **Root dir is not a git repo** | Prevents accidental commits across repos; enforces repo boundaries |
| **Reusable workflows on `shared/java`** | Multiple stacks (Java, JavaScript, etc.) can coexist on separate branches without conflict |
| **Terraform split: `dev-infra` + `dev-k8s`** | Avoids `cannot create REST client` errors by ensuring the EKS cluster exists before the Kubernetes provider initializes |
| **Local state for `backend-setup`** | S3 bucket for remote state cannot use itself for state storage; local state is the chicken-and-egg solution |
| **Bootstrap via local Helm chart** | `kubernetes_manifest` fails during `plan` when CRDs don't yet exist; local Helm chart deploys the Application CRD after the Helm release is ready |
| **ArgoCD `insecure: true`** | Simplifies initial setup by avoiding TLS certificate management; acceptable for dev/UAT behind firewalls |
| **OWASP in scheduled scan only** | NVD API downloads take 8–15+ minutes and frequently timeout; removing from deploy pipelines keeps deployments fast |
| **GitOps values path mapping by branch** | Simple convention: `feature/*` → dev, `main` → uat, `release/*` → prod; no complex routing logic needed |
| **Template mirroring across environments** | All three environments share identical templates to prevent configuration drift |
| **Immutable image tags** | `latest` is banned; explicit short commit SHAs ensure reproducible deployments and easy rollbacks |
| **Production: no forced sync** | Production deployment updates GitOps values but relies on ArgoCD auto-sync; prevents accidental immediate deployments if the pipeline has a bug |
| **Single ArgoCD token** | Originally had `ARGOCD_AUTH_TOKEN` and `ARGOCD_TOKEN_PROD`; unified to a single token for simplicity |

---

## Related Documentation

Each sub-repository contains its own detailed documentation:

- [`demo-java-app/README.md`](./demo-java-app/README.md)
- [`terraform-aws-eks-argocd/README.md`](./terraform-aws-eks-argocd/README.md)
- [`gitops-manifests/README.md`](./gitops-manifests/README.md)
- [`reusable-pipeline-templates/README.md`](./reusable-pipeline-templates/README.md)

---

*Last updated: 2026-06-23*
