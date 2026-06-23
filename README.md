# demo-java-app

A simple Spring Boot 3.2.5 web application that demonstrates a CI/CD pipeline.

> **Part of the DevOps Platform**: This is the application repository in a four-repo system. See the [workspace root README](../README.md) for the complete architecture guide.

---

## Table of Contents

1. [Overview](#overview)
2. [Technology Stack](#technology-stack)
3. [Prerequisites](#prerequisites)
4. [Local Development](#local-development)
5. [CI/CD Workflows](#cicd-workflows)
6. [Branch Naming Convention](#branch-naming-convention)
7. [Docker](#docker)
8. [Project Structure](#project-structure)
9. [Workflow Configuration Reference](#workflow-configuration-reference)
10. [Related Repositories](#related-repositories)

---

## Overview

This application serves a Thymeleaf-based UI at the root context (`/`). It showcases a minimal Spring Boot project structure with:
- Spring Boot 3.2.5
- Java 17
- Maven build system
- Docker multi-stage build
- GitHub Actions CI/CD integration
- ArgoCD GitOps deployment to three environments

---

## Technology Stack

| Layer | Technology |
|-------|------------|
| Language | Java 17 |
| Framework | Spring Boot 3.2.5 |
| Template Engine | Thymeleaf |
| Build Tool | Maven (requires 3.9+; no wrapper included) |
| Container | Docker (multi-stage build) |
| CI/CD | GitHub Actions |
| Deployment | ArgoCD GitOps |

---

## Prerequisites

- **Java 17** or later
- **Maven 3.9+** (must be installed locally; no `mvnw` included)
- **Docker** (for local container builds)

---

## Local Development

### Running Tests

```bash
mvn test
```

JaCoCo coverage report is generated automatically during the `test` phase in `target/site/jacoco/`.

### Running a Single Test Class

```bash
mvn test -Dtest=HomeControllerTest
```

### Running Locally

```bash
mvn spring-boot:run
```

The application starts on `http://localhost:8080`.

### Building the Application

```bash
mvn clean package -DskipTests
```

Creates `target/demo-1.0.jar`.

---

## CI/CD Workflows

All workflows delegate to the centralized [`reusable-pipeline-templates`](../reusable-pipeline-templates/) repository on branch `shared/java`.

### Workflow Files

| File | Trigger | Reusable Workflow | Target Environment |
|------|---------|-------------------|-------------------|
| `.github/workflows/feature.yml` | Push to `feature/*` | `java-on-feature-branch.yml` | Development |
| `.github/workflows/main.yml` | Push to `main` | `java-on-main-branch-update.yml` | UAT |
| `.github/workflows/release.yml` | Push to `release/*` | `java-on-release-branch.yml` | Production |
| `.github/workflows/pull-request.yml` | Pull request to any branch | `java-on-pull-request.yml` | None (CI only) |

### Pipeline Behavior

| Trigger | Build & Test | Docker Push | GitOps Deploy | ArgoCD Sync | Rollout Check |
|---------|--------------|-------------|---------------|-------------|---------------|
| `feature/*` | ✅ | ✅ | `development/values/` | ✅ | ✅ |
| `main` | ✅ | ✅ | `uat/values/` | ✅ | ✅ |
| `release/*` | ✅ | ✅ | `production/values/` | ❌ | ❌ |
| PR | ✅ | ❌ | ❌ | ❌ | ❌ |

- **Development & UAT**: Full automation including explicit ArgoCD sync trigger and rollout health check (up to 600s timeout).
- **Production**: Updates the GitOps values file but does not force an ArgoCD sync or rollout check. ArgoCD auto-sync handles the deployment.

### How the Pipeline Updates GitOps

1. Builds JAR, runs tests, generates Docker image with short commit SHA tag (e.g., `e4b6b02`)
2. Pushes image to `docker.io/francdocmain/demo-java-app:<tag>`
3. Clones [`gitops-manifests`](../gitops-manifests/) repo
4. Uses `sed` to update `image.repository` and `image.tag` in the target environment's `values/demo-java-app.yaml`
5. Commits and pushes changes
6. ArgoCD auto-syncs within ~30 seconds

---

## Branch Naming Convention

PR branches **must** use one of these prefixes:
- `feature/`
- `release/`
- `hotfix/`
- `bugfix/`

Invalid prefixes cause the `check-branch-name` job in `pull-request.yml` to fail with exit 1.

---

## Docker

### Multi-Stage Build

**Stage 1 (Build):** `maven:3.9.6-eclipse-temurin-17`
- Compiles and packages the JAR with `mvn clean package -DskipTests`

**Stage 2 (Runtime):** `eclipse-temurin:17-jre-alpine`
- Runs `target/demo-1.0.jar` on port `8080`

### Local Commands

```bash
# Build image
docker build -t demo-java-app .

# Run container
docker run -p 8080:8080 demo-java-app
```

---

## Project Structure

```
demo-java-app/
├── .github/workflows/
│   ├── feature.yml          # Deploy to dev (feature/* push)
│   ├── main.yml             # Deploy to UAT (main push)
│   ├── pull-request.yml     # CI checks (PRs)
│   └── release.yml          # Deploy to prod (release/* push)
├── src/
│   ├── main/java/com/example/demo/
│   │   ├── DemoApplication.java   # Spring Boot entry point
│   │   └── HomeController.java    # GET / controller
│   ├── main/resources/templates/
│   │   └── index.html             # Thymeleaf UI template
│   └── test/java/com/example/demo/
│       └── HomeControllerTest.java # @WebMvcTest
├── Dockerfile                     # Multi-stage build
├── pom.xml                        # Maven + JaCoCo + SonarCloud
└── README.md                      # This file
```

### Key Configuration

**`pom.xml`:**
- Spring Boot parent: `3.2.5`
- Java version: `17`
- Plugins: `spring-boot-maven-plugin`, `jacoco-maven-plugin` (0.8.11)
- SonarCloud: `sonar.projectKey=francdomain`, `sonar.organization=francdomain`, `sonar.host.url=https://sonarcloud.io`

---

## Workflow Configuration Reference

### `feature.yml`

```yaml
with:
  docker-registry: "docker.io"
  docker-image-name: "francdocmain/demo-java-app"
  gitops-repo: "francdomain/gitops-manifests"
  gitops-branch: "main"
  gitops-values-path: "development/values/demo-java-app.yaml"
  argocd-app-name: "demo-java-app-development"
  rollout-enabled: true
```

### `main.yml`

```yaml
with:
  docker-registry: "docker.io"
  docker-image-name: "francdocmain/demo-java-app"
  gitops-repo: "francdomain/gitops-manifests"
  gitops-branch: "main"
  gitops-values-path: "uat/values/demo-java-app.yaml"
  argocd-app-name: "demo-java-app-uat"
  rollout-enabled: true
```

### `release.yml`

```yaml
with:
  docker-registry: "docker.io"
  docker-image-name: "francdocmain/demo-java-app"
  gitops-repo: "francdomain/gitops-manifests"
  gitops-branch: "main"
  gitops-values-path: "production/values/demo-java-app.yaml"
  argocd-app-name: "demo-java-app-production"
  rollout-enabled: false
```

---

## Related Repositories

| Repository | Role | Link |
|-----------|------|------|
| `reusable-pipeline-templates` | Centralized GitHub Actions workflows | [../reusable-pipeline-templates/](../reusable-pipeline-templates/) |
| `gitops-manifests` | Helm charts per environment (ArgoCD watches this) | [../gitops-manifests/](../gitops-manifests/) |
| `terraform-aws-eks-argocd` | Infrastructure: AWS VPC → EKS → ArgoCD | [../terraform-aws-eks-argocd/](../terraform-aws-eks-argocd/) |

---

*For the complete platform architecture, implementation steps, and troubleshooting, see the (DOCUMENTATION.md).*
