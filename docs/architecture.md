# Architecture

[← Back to README](../README.md)

## 1. Architecture Overview

This project implements a GitHub Actions CI/CD workflow around the **VProfile Java application workload**.

The architecture separates the delivery process into two major responsibilities:

```text
VALIDATION
    │
    ├── Build
    ├── Testing
    └── Security Scan
          │
          ▼
PUBLISH
    │
    ├── Docker Build
    └── Amazon ECR
```

The project therefore represents the **CI/CD engineering around the application workload**, rather than development of the application itself.

The VProfile application is the workload being processed by the pipeline.

---

## 2. Application Ownership Boundary

The architecture should be understood as:

```text
Existing Application Workload
             +
     CI/CD Engineering
```

and not:

```text
Application Development
             +
     CI/CD Engineering
```

The VProfile application was used as the workload for the practical.

The engineering responsibility represented by this repository is the automation surrounding that workload:

- Source checkout
- Application build
- Testing
- Checkstyle execution
- Artifact persistence
- Security scanning
- Workflow orchestration
- Branch and job conditions
- AWS authentication
- Docker image construction
- Amazon ECR publication

Application business logic and original application architecture are outside this project's ownership boundary.

---

## 3. High-Level System Architecture

The complete workflow can be represented as:

```text
                     GitHub Repository
                            │
                            │ Trigger
                            ▼
                    GitHub Actions
                            │
                            ▼
                         Build
                            │
                  ┌─────────┴─────────┐
                  │                   │
                  ▼                   ▼
              Testing           Security Scan
                  │                   │
                  │                   │
                  └─────────┬─────────┘
                            ▼
                   BUILD_AND_PUBLISH
                            │
                  ┌─────────┴─────────┐
                  │                   │
                  ▼                   ▼
             Docker Build        ECR Login
                  │                   │
                  └─────────┬─────────┘
                            ▼
                     Docker Push
                            │
                            ▼
                       Amazon ECR
```

The key architectural boundary is:

```text
Source
  ↓
Validation
  ↓
Publishing
```

The workflow does not include a runtime deployment stage after ECR.

---

# 4. Pipeline Execution Architecture

## 4.1 Build Stage

`Build` is the first job in the dependency graph.

Conceptually:

```text
GitHub Source
     │
     ▼
Checkout
     │
     ▼
 Maven Build
     │
     ▼
WAR Artifact
```

The Build job runs on a GitHub-hosted runner.

The source repository is checked out into the runner workspace before the build commands execute.

The Maven build produces the application artifact.

---

## 4.2 Testing Stage

The `Testing` job depends on `Build`:

```yaml
needs: Build
```

Therefore:

```text
Build
  ↓
Testing
```

Testing executes on its own GitHub-hosted runner.

Because GitHub Actions jobs use separate runners, the Testing job performs its own repository checkout rather than assuming that files created by the Build job are available locally.

The testing stage includes:

```text
Maven Tests
     +
Checkstyle
```

Conceptually:

```text
Build
  ↓
Maven Test
  ↓
Checkstyle
  ↓
Pass / Fail
```

---

## 4.3 Security Scan Stage

The `Security_Scan` job also depends on `Build`:

```yaml
needs: Build
```

Therefore the dependency graph becomes:

```text
             ┌──► Testing
Build ───────┤
             └──► Security_Scan
```

Because Testing and Security Scan do not depend on each other, they can run in parallel after Build completes.

The security stage performs a Trivy filesystem scan:

```text
Repository
    │
    ▼
 Trivy
    │
    ▼
trivy-results.json
    │
    ▼
GitHub Artifact
```

The scan configuration uses:

```yaml
scan-type: fs
scan-ref: .
format: json
exit-code: 0
output: trivy-results.json
```

The scan therefore analyzes the checked-out repository filesystem and produces a machine-readable JSON result.

---

# 5. Job Dependency Graph

The final workflow uses a fan-out/fan-in topology.

```text
                         ┌──────────────┐
                         │   Testing    │
                         └──────┬───────┘
                                │
                                │
┌─────────┐                     │
│  Build  │─────────────────────┼──────► BUILD_AND_PUBLISH
└────┬────┘                     │
     │                          │
     │                   ┌──────┴────────┐
     └──────────────────►│ Security_Scan │
                         └───────────────┘
```

More precisely:

```text
Build
  │
  ├──► Testing
  │
  └──► Security_Scan
           │
           └──────────────┐
                          ▼
                  BUILD_AND_PUBLISH
```

The final job declares:

```yaml
needs: [Build, Testing, Security_Scan]
```

This creates the fan-in point.

The publishing job waits until all three required jobs have completed successfully.

### Why this topology?

Testing and Security Scan are independent validation concerns.

Therefore:

```text
Build
  ├──► Testing
  └──► Security Scan
```

is preferable to unnecessarily creating:

```text
Build
  ↓
Testing
  ↓
Security Scan
```

when Security Scan has no dependency on Testing.

The architecture therefore uses parallelism where there is no data or logic dependency.

---

# 6. Publishing Gate

`BUILD_AND_PUBLISH` is the final job in the workflow.

Its responsibility is to transform validated source into a Docker image and publish that image to Amazon ECR.

The publishing boundary is:

```text
Build
  +
Testing
  +
Security Scan
       │
       ▼
 Validation Complete
       │
       ▼
BUILD_AND_PUBLISH
       │
       ▼
 Docker Image
       │
       ▼
 Amazon ECR
```

The dependency declaration:

```yaml
needs: [Build, Testing, Security_Scan]
```

acts as the workflow's final quality gate.

If one of the required jobs fails, the publishing job does not proceed normally.

---

# 7. Branch Execution Boundary

The publishing job is additionally restricted to the `main` branch:

```yaml
if: github.ref == 'refs/heads/main'
```

This creates two separate controls:

```text
Workflow Trigger
       │
       ▼
Should the workflow execute?
       │
       ▼
Job Condition
       │
       ▼
Should publishing execute?
```

This allows validation workflows to run for other Git references without automatically publishing a Docker image.

The architecture therefore distinguishes:

```text
CI Validation
       ≠
Container Publication
```

---

# 8. Workflow Trigger Architecture

The workflow supports multiple ways of starting execution:

```text
                    ┌── push
                    │
                    ├── pull_request
GitHub Event ───────┼── workflow_dispatch
                    │
                    └── schedule
                            │
                            ▼
                     GitHub Actions
```

The demonstrated workflow includes:

```yaml
on:
  push:
    branches:
      - main

  pull_request:
    branches:
      - main

  workflow_dispatch:

  schedule:
    - cron: "10 14 * * 1-5"
```

The trigger architecture therefore supports:

- source pushes
- pull requests targeting `main`
- manual workflow execution
- scheduled execution

The workflow also uses:

```yaml
permissions:
  contents: read
```

to restrict the workflow's repository-content permission to read-only.

---

# 9. Runner Architecture

Each GitHub Actions job executes on its own runner.

Conceptually:

```text
Build
  │
  ▼
Runner A

Testing
  │
  ▼
Runner B

Security Scan
  │
  ▼
Runner C

BUILD_AND_PUBLISH
  │
  ▼
Runner D
```

The runners are ephemeral.

This has an important architectural consequence:

```text
Job A Workspace
      ≠
Job B Workspace
```

Files created inside one job are not automatically available in another job.

Therefore each job that needs repository source performs its own checkout.

For outputs that must survive the runner's lifetime, the workflow uses GitHub Actions artifacts.

---

# 10. Artifact Architecture

The project uses GitHub Actions artifacts to preserve important generated outputs.

## 10.1 Build Artifact

The Maven build produces a WAR file:

```text
Maven
  │
  ▼
target/*.war
  │
  ▼
upload-artifact
  │
  ▼
vprofile-app
```

The artifact is stored outside the ephemeral runner workspace and can be downloaded from the workflow run.

The practical explicitly uses `actions/upload-artifact@v4` for this purpose.

---

## 10.2 Security Scan Artifact

The Trivy job produces:

```text
trivy-results.json
```

The workflow then uploads it:

```text
Trivy
  │
  ▼
trivy-results.json
  │
  ▼
upload-artifact
  │
  ▼
trivy-scan-results
```

This follows the general:

```text
SCAN
  ↓
STORE
  ↓
DECIDE
```

pattern.

The current decision behavior is controlled by Trivy's `exit-code`.

---

# 11. Security Architecture

## 11.1 Repository Token

The workflow defines:

```yaml
permissions:
  contents: read
```

This gives the workflow's `GITHUB_TOKEN` read-only repository-content access.

The token is therefore not granted unnecessary repository write permissions.

---

## 11.2 Trivy Security Scan

The security stage is positioned after Build:

```text
Build
  │
  ▼
Security Scan
```

The scan operates on the repository filesystem:

```text
Checked-Out Repository
          │
          ▼
        Trivy
          │
          ▼
 Vulnerability Results
```

The current configuration uses:

```yaml
exit-code: 0
```

This means vulnerabilities are reported without causing the Trivy step to fail.

Therefore the current architecture should be understood as:

```text
Security Scan
     ↓
Report
     ↓
Continue
```

rather than:

```text
Security Scan
     ↓
Vulnerability Found
     ↓
Pipeline Blocked
```

A future security-gating implementation could change the exit-code behavior and establish an enforced vulnerability policy.

---

# 12. AWS Authentication Architecture

The publishing job uses an AWS authentication chain.

Conceptually:

```text
GitHub Environment
       │
       ├── AWS_ACCESS_KEY_ID
       ├── AWS_SECRET_ACCESS_KEY
       └── AWS_REGION
       │
       ▼
Configure AWS Credentials
       │
       ▼
AWS Session
       │
       ▼
Amazon ECR Login
       │
       ▼
Docker Push
```

Sensitive AWS credentials are stored as GitHub Environment Secrets.

The AWS region is stored as a GitHub Environment Variable.

The publishing job is associated with the production environment so that the environment-scoped configuration becomes available to the job.

---

# 13. Docker Architecture

The publishing stage transforms the application source into a container image.

The conceptual flow is:

```text
Application Source
       │
       ▼
 Docker Build Context
       │
       ▼
     Docker
       │
       ▼
 Docker Image
```

The Dockerfile is adapted for the GitHub Actions environment.

The source is already available on the runner because GitHub Actions has checked out the repository.

Therefore the Docker build can use the repository as its build context rather than requiring the Dockerfile to obtain the source independently.

Conceptually:

```text
GitHub Actions Runner
        │
        ├── application source
        ├── Dockerfile
        └── build context
                │
                ▼
           Docker Build
```

The Docker build uses the repository root as the build context.

---

# 14. Docker Image Identity

The image is tagged using the Git commit SHA.

Conceptually:

```text
Git Commit
    │
    │ github.sha
    ▼
Docker Image Tag
    │
    ▼
<registry>/<repository>:<commit-sha>
```

This creates a traceability relationship:

```text
Source Revision
      │
      ▼
Git SHA
      │
      ▼
Docker Image
      │
      ▼
Amazon ECR
```

The image can therefore be associated with the source revision that generated it.

---

# 15. Amazon ECR Architecture

Amazon ECR is the container registry used by the publishing stage.

The architecture is:

```text
GitHub Actions
      │
      │ Docker Build
      ▼
Docker Image
      │
      │ Docker Push
      ▼
Amazon ECR
```

The responsibilities are intentionally separated:

```text
Docker
  =
Builds the container image

Amazon ECR
  =
Stores the container image
```

The current project ends at the ECR boundary.

There is no runtime deployment stage in this project.

---

# 16. End-to-End Data and Artifact Flow

The complete architecture can be compressed into:

```text
                     SOURCE
                       │
                       ▼
              GitHub Repository
                       │
                  Trigger Event
                       │
                       ▼
                GitHub Actions
                       │
                       ▼
                     Build
                       │
                Maven / WAR
                       │
          ┌────────────┴────────────┐
          │                         │
          ▼                         ▼
       Testing                Security Scan
          │                         │
          │                    JSON Result
          │                         │
          │                    Artifact Store
          │                         │
          └────────────┬────────────┘
                       │
                       ▼
               Publishing Gate
                       │
                       ▼
                BUILD_AND_PUBLISH
                       │
              ┌────────┴─────────┐
              │                  │
              ▼                  ▼
        AWS Authentication   Docker Build
              │                  │
              └────────┬─────────┘
                       ▼
                  Docker Push
                       │
                       ▼
                  Amazon ECR
```

---

# 17. Architectural Decisions

## 17.1 Separate Build and Testing Jobs

Build and Testing are separate jobs so that the workflow explicitly represents the distinction between producing the application artifact and validating it.

---

## 17.2 Parallel Testing and Security Scanning

Testing and Security Scan both depend on Build but not on each other.

Therefore they run in parallel:

```text
             ┌──► Testing ───────┐
Build ───────┤                   ├──► Publish
             └──► Security Scan ─┘
```

This avoids introducing a dependency that does not exist.

---

## 17.3 Explicit Publishing Gate

The publishing job declares all three validation jobs as dependencies:

```yaml
needs: [Build, Testing, Security_Scan]
```

This creates a clear transition:

```text
Validation
    ↓
Publishing
```

---

## 17.4 Main-Branch Publishing

Publishing is restricted to:

```text
refs/heads/main
```

This separates general CI validation from image publication.

---

## 17.5 Artifact Persistence

Artifacts are explicitly uploaded because GitHub-hosted runners are ephemeral.

This applies both to:

```text
vprofile-app
```

and:

```text
trivy-scan-results
```

The workflow therefore does not depend on runner-local state surviving after job completion.

---

## 17.6 Least-Privilege Repository Permission

The workflow uses:

```yaml
permissions:
  contents: read
```

to avoid unnecessary repository write access.

---

## 17.7 Commit-SHA Image Tagging

The commit SHA is used as the image tag to create a direct source-to-image relationship.

---

# 18. Architectural Boundaries

This project intentionally does **not** claim the following capabilities:

- VProfile application development
- VProfile business-logic implementation
- Infrastructure as Code
- Terraform-based infrastructure provisioning
- Kubernetes deployment
- GitOps
- Automated rollback
- Zero-downtime deployment
- Production runtime orchestration
- Production observability
- Enterprise AWS identity federation
- Complete production security policy enforcement

The architecture ends at:

```text
Docker Image
     ↓
Amazon ECR
```

It does not continue into:

```text
ECR
 ↓
ECS / EKS
 ↓
Running Application
```

---

# 19. Future Architectural Evolution

The current architecture provides a foundation for future CI/CD evolution.

### Current Architecture

```text
GitHub
  ↓
GitHub Actions
  ↓
Build
  ↓
Testing + Security Scan
  ↓
Publishing Gate
  ↓
Docker
  ↓
Amazon ECR
```

### Possible Future Architecture

```text
GitHub
  ↓
GitHub Actions
  ↓
Build
  ↓
Testing
  ↓
Security Policy Gate
  ↓
Container Image Scan
  ↓
Amazon ECR
  ↓
ECS / EKS
  ↓
Deployment
  ↓
Observability
```

Other possible future improvements include:

```text
GitHub OIDC
     ↓
Short-Lived AWS Credentials
```

and:

```text
Terraform
     ↓
AWS Infrastructure
```

These are future architectural directions and are not completed capabilities of this project.

---

# 20. Architecture Summary

The entire project can be reduced to one mental model:

```text
                    GitHub
                      │
                      ▼
               GitHub Actions
                      │
                      ▼
                    Build
                      │
              ┌───────┴────────┐
              ▼                ▼
           Testing        Security Scan
              │                │
              └───────┬────────┘
                      ▼
              Publishing Gate
                      │
                      ▼
                Docker Build
                      │
                      ▼
              Commit-SHA Image
                      │
                      ▼
                 Amazon ECR
```

The central architectural principle is:

> **Validate first, then publish.**

The pipeline uses explicit job dependencies, parallel validation where dependencies are independent, artifact persistence across ephemeral runners, restricted repository permissions, environment-scoped AWS configuration, and commit-based container image identification to implement that model.
