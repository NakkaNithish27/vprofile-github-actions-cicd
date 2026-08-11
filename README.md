# VProfile GitHub Actions CI/CD Pipeline

A GitHub Actions CI/CD pipeline that automates application build and validation, performs vulnerability scanning, and gates Docker image publication to Amazon ECR.

---

## Overview

This project demonstrates an end-to-end CI/CD workflow built with **GitHub Actions** around the **VProfile Java application workload**.

The pipeline evolved from a basic build-and-test workflow into a gated container publishing pipeline:

```text
GitHub
   │
   ▼
GitHub Actions
   │
   ▼
  Build
   │
   ▼
┌──┴──────────────┐
│                 │
▼                 ▼
Testing       Security Scan
│                 │
└───────┬─────────┘
        ▼
 BUILD_AND_PUBLISH
        │
        ▼
   Docker Image
        │
        ▼
    Amazon ECR
```

The project focuses on the **CI/CD engineering around the application workload**, rather than application development.

---

## Application Ownership Boundary

The VProfile application was used as the workload for this practical.

I did **not** develop the VProfile application's Java business logic, authentication logic, or original application architecture.

My engineering work focuses on the delivery workflow around the existing workload, including:

- GitHub Actions workflow configuration
- CI job orchestration
- Workflow triggers and branch conditions
- Artifact persistence
- Repository permission configuration
- Trivy vulnerability scanning
- GitHub Environment Secrets and Variables
- AWS ECR integration
- Dockerfile adaptation for CI execution
- Docker image build and tagging
- Docker image publication to Amazon ECR
- Workflow validation

This distinction is intentional:

```text
Existing Application Workload
          ≠
CI/CD Engineering Around the Workload
```

---

## Engineering Objective

The objective was to build a GitHub Actions workflow that moves the application through a controlled delivery process:

```text
Source Change
     ↓
Build
     ↓
Testing
     ↓
Security Scan
     ↓
Validation Gate
     ↓
Docker Build
     ↓
Amazon ECR
```

The final workflow separates **validation** from **publishing**, allowing build, testing, and security concerns to be represented as distinct jobs before the container image is published.

---

## Architecture

The final workflow uses the following job structure:

```text
                  ┌───────────────┐
                  │    Testing    │
                  └───────┬───────┘
                          │
                          │
┌─────────┐               ▼
│  Build  │───────► BUILD_AND_PUBLISH
└────┬────┘               ▲
     │                    │
     │            ┌───────┴────────┐
     └───────────►│ Security_Scan  │
                  └────────────────┘
```

The execution model is:

```text
Build
  │
  ├──► Testing
  │
  └──► Security_Scan
           │
           └──────┐
                  ▼
          BUILD_AND_PUBLISH
                  │
                  ▼
              Docker
                  │
                  ▼
                 ECR
```

`Testing` and `Security_Scan` depend on the successful completion of `Build`.

The final publishing job depends on the validation jobs and is restricted to the `main` branch.

For the detailed pipeline architecture and dependency model:

**[Architecture →](docs/architecture.md)**

---

## My Engineering Contribution

### GitHub Actions CI/CD

- Created and configured the GitHub Actions workflow.
- Structured the workflow into separate jobs.
- Configured job dependencies using `needs`.
- Configured GitHub-hosted runners.
- Configured workflow triggers.
- Configured action inputs.
- Configured branch-aware conditions.
- Configured repository token permissions.

### Build & Testing

- Automated the Maven build.
- Automated Maven test execution.
- Automated Checkstyle execution.
- Persisted the generated application artifact using GitHub Actions artifacts.

### Security

- Integrated Trivy filesystem vulnerability scanning.
- Configured JSON scan output.
- Persisted Trivy results as a workflow artifact.
- Established the security-scan stage as an independent validation job.

### AWS & ECR

- Created and configured the ECR destination used by the pipeline.
- Configured AWS authentication for GitHub Actions.
- Configured GitHub Environment Secrets for sensitive AWS credentials.
- Configured a GitHub Environment Variable for the AWS region.
- Integrated Amazon ECR authentication into the workflow.
- Configured Docker image publication to ECR.

### Docker

- Adapted the Dockerfile for the GitHub Actions build context.
- Used the repository source already available on the GitHub Actions runner.
- Configured the Docker build to use the repository root as its build context.
- Used a multi-stage Docker build.
- Tagged the resulting image using the Git commit SHA.
- Published the image to Amazon ECR.

---

## Key Engineering Decisions

### Separate Validation Jobs

Testing and security scanning represent different validation concerns.

They therefore execute as separate jobs after the Build job:

```text
Build
  ├──► Testing
  └──► Security Scan
```

This allows the independent validation stages to execute without unnecessarily serializing the entire workflow.

### Explicit Publishing Gate

The publishing stage depends on:

```yaml
needs: [Build, Testing, Security_Scan]
```

This establishes a clear boundary between:

```text
Validation
    ↓
Publishing
```

The Docker image is not published until the required upstream jobs have completed successfully.

### Branch-Aware Publishing

The publishing job is restricted to the main branch using the GitHub reference:

```yaml
if: github.ref == 'refs/heads/main'
```

This allows validation workflows to execute for other Git references without automatically publishing images.

### Ephemeral Runner Awareness

GitHub-hosted runners are temporary execution environments.

Generated files such as Maven build outputs therefore do not automatically survive the workflow run.

The project uses GitHub Actions artifacts to preserve important outputs.

### Least-Privilege Repository Access

The workflow restricts the repository contents permission:

```yaml
permissions:
  contents: read
```

This avoids granting unnecessary write access to the workflow's `GITHUB_TOKEN`.

### Environment-Scoped AWS Credentials

Sensitive AWS credentials are stored as GitHub Environment Secrets rather than being written directly into workflow configuration.

The AWS region is stored separately as a non-sensitive GitHub Environment Variable.

### Commit-SHA Image Tagging

The Docker image is tagged using the Git commit SHA:

```text
<registry>/<repository>:<commit-sha>
```

This provides a direct relationship between the published image and the source revision that produced it.

---

## Validation

The completed workflow is validated through:

- Successful GitHub Actions workflow execution
- Successful Build job
- Successful Testing job
- Security Scan execution
- Downloadable build artifact
- Downloadable Trivy scan result
- Successful Docker image creation
- Docker image availability in Amazon ECR
- Commit SHA correspondence between the Git revision and ECR image tag

The detailed validation strategy and evidence mapping are documented here:

**[Validation →](docs/validation.md)**

---

## Project Boundaries

This project demonstrates **GitHub Actions CI/CD and container image publishing**.

It does **not** demonstrate:

- Development of the VProfile application
- VProfile business-logic implementation
- Application architecture development
- Infrastructure as Code
- Terraform-based infrastructure provisioning
- Kubernetes deployment
- GitOps
- Automated rollback
- Zero-downtime deployment
- Production observability
- Enterprise-grade deployment orchestration

The current Trivy configuration is informational rather than a blocking vulnerability gate.

The AWS authentication approach demonstrated by the practical uses IAM access keys stored in GitHub Secrets rather than GitHub OIDC federation.

The pipeline ends at Amazon ECR; deployment of the image to a runtime platform is outside this project's scope.

For the complete boundaries and logical next steps:

**[Limitations & Future Work →](docs/limitations-and-future-work.md)**

---

## Technologies

| Area | Technology |
|---|---|
| CI/CD | GitHub Actions |
| Source Control | Git / GitHub |
| Build | Apache Maven |
| Testing | Maven |
| Code Quality | Checkstyle |
| Security Scanning | Trivy |
| Containerization | Docker |
| Container Registry | Amazon ECR |
| Cloud Platform | AWS |
| Workflow Definition | YAML |
| Configuration | GitHub Secrets / Variables |

---

## Repository Documentation

The repository uses a layered documentation structure so the README remains concise while preserving deeper engineering memory.

### Architecture

Pipeline topology, job relationships, execution flow, security boundaries, validation/publishing separation, and major architectural decisions.

**[Read Architecture →](docs/architecture.md)**

### Implementation

Workflow construction, triggers, artifacts, conditions, permissions, Trivy integration, AWS/ECR configuration, Dockerfile adaptation, and Docker publishing implementation.

**[Read Implementation →](docs/implementation.md)**

### Validation

Validation strategy, workflow verification, artifact verification, ECR validation, commit-to-image traceability, and evidence mapping.

**[Read Validation →](docs/validation.md)**

### Limitations & Future Work

Current project boundaries, security limitations, deployment boundaries, and logical future improvements.

**[Read Limitations & Future Work →](docs/limitations-and-future-work.md)**

---

## Evidence

High-signal evidence from the completed environment should be maintained under:

```text
evidence/screenshots/
```

Evidence should demonstrate meaningful engineering claims such as:

- Successful final GitHub Actions workflow execution
- Final job dependency graph
- Build artifact availability
- Trivy scan result
- Docker image in Amazon ECR
- Commit SHA used as the image tag

Only evidence from my own completed environment should be presented as proof of personal execution.

Course screenshots, lecture material, and copied course artifacts should not be presented as personal execution evidence.

---

## Project Flow

The resulting engineering workflow can be summarized as:

```text
Developer Change
       │
       ▼
     GitHub
       │
       ▼
 GitHub Actions
       │
       ▼
     Build
       │
       ├──────────────┐
       ▼              ▼
   Testing       Security Scan
       │              │
       └──────┬───────┘
              ▼
        Publish Gate
              │
              ▼
         Docker Build
              │
              ▼
        Commit-SHA Tag
              │
              ▼
          Amazon ECR
```

The project demonstrates how GitHub Actions can coordinate application validation, security scanning, containerization, and container registry publication as one controlled CI/CD workflow.

---

## Future Direction

Logical future improvements include:

```text
Current CI/CD Pipeline
        ↓
GitHub OIDC Authentication
        ↓
Repository-Scoped AWS IAM
        ↓
Enforced Vulnerability Policy
        ↓
Container Image Scanning
        ↓
ECS / EKS Deployment
        ↓
Infrastructure as Code
        ↓
Automated Rollback
        ↓
Observability
```

These capabilities are **future improvements**, not completed capabilities claimed by this repository.

---

## Repository Structure

```text
vprofile-github-actions-cicd/
│
├── README.md
│
├── .github/
│   └── workflows/
│       └── main.yml
│
├── docs/
│   ├── architecture.md
│   ├── implementation.md
│   ├── validation.md
│   └── limitations-and-future-work.md
│
├── evidence/
│   └── screenshots/
│
└── .gitignore
```

The repository intentionally excludes generated build outputs, temporary runner files, secrets, course material, and application artifacts that are not appropriate for public redistribution.

---

## Project Summary

This project demonstrates a practical GitHub Actions CI/CD workflow around an existing Java application workload:

```text
GitHub
   ↓
GitHub Actions
   ↓
Maven Build
   ↓
Testing
   ↓
Trivy Security Scan
   ↓
Validation Gate
   ↓
Docker Build
   ↓
Amazon ECR
```

The core engineering contribution is the **automation and integration of the software delivery workflow around the existing application workload**.

The repository is intentionally focused on the engineering work performed, the evidence supporting that work, and the technical decisions behind the implementation rather than reproducing the learning material or claiming ownership of supplied application components.
