# Implementation

[← Back to README](../README.md)

## 1. Implementation Overview

The implementation evolved incrementally from a basic GitHub Actions workflow into a CI/CD pipeline capable of validating the VProfile application workload, performing vulnerability scanning, building a Docker image, and publishing that image to Amazon ECR.

The implementation progression was:

```text
Workflow Foundation
      ↓
Triggers & Action Inputs
      ↓
Artifacts & Conditions
      ↓
Security Scanning
      ↓
AWS / ECR Preparation
      ↓
Dockerfile Adaptation
      ↓
Build & Publish
```

The implementation described in this document represents the **CI/CD engineering performed around the existing VProfile application workload**.

---

# 2. Repository Structure

The project repository uses the following structure:

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

The primary CI/CD implementation is located at:

```text
.github/workflows/main.yml
```

---

# 3. Workflow Foundation

The initial workflow was structured around two jobs:

```text
Build
  ↓
Testing
```

The purpose of this initial structure was to establish the fundamental GitHub Actions execution model before adding security and publishing stages.

The Build job:

1. checks out the source repository
2. executes the Maven build
3. produces the application artifact

The Testing job:

1. checks out the repository
2. executes Maven tests
3. executes Checkstyle

The basic dependency relationship is:

```yaml
needs: Build
```

which produces:

```text
Build
  ↓
Testing
```

---

# 4. GitHub Actions Runner Configuration

The workflow uses GitHub-hosted Ubuntu runners.

Each job executes on its own runner.

Conceptually:

```text
Build
  ↓
Runner A

Testing
  ↓
Runner B
```

Because the runners are separate execution environments, the Testing job cannot assume that files created inside the Build job's workspace still exist.

Therefore each job that requires repository source performs its own checkout.

This is an important implementation detail of GitHub Actions' ephemeral runner model.

---

# 5. Source Checkout

The workflow uses the GitHub Actions checkout action to obtain the repository source.

The basic checkout step is:

```yaml
- name: Checkout code
  uses: actions/checkout@v4
```

Later in the implementation, full Git history is requested where required:

```yaml
- name: Checkout code
  uses: actions/checkout@v4
  with:
    fetch-depth: 0
```

The `with` block supplies inputs to the action.

The implementation therefore distinguishes between:

```text
Action
  ↓
Configuration through with
```

and:

```text
Shell command
  ↓
run
```

This becomes important later when configuring the Trivy action and AWS-related actions.

---

# 6. Build Implementation

The Build job runs Maven against the checked-out application source.

The build command is:

```bash
mvn install
```

The conceptual implementation is:

```text
Checkout
   ↓
Maven Install
   ↓
Build Output
```

The Maven build generates the application WAR under the Maven target directory.

The resulting artifact is subsequently persisted using GitHub Actions artifact storage.

---

# 7. Testing Implementation

The Testing job is configured to run after Build:

```yaml
needs: Build
```

The testing stage runs:

```bash
mvn test
```

followed by:

```bash
mvn checkstyle:checkstyle
```

The resulting flow is:

```text
Build
  ↓
Maven Test
  ↓
Checkstyle
  ↓
Testing Result
```

The Testing job remains a separate workflow job rather than being embedded into the Build job.

This makes the validation stages visible in the GitHub Actions job graph.

---

# 8. Workflow Triggers

The workflow was expanded to support several execution triggers.

The demonstrated trigger structure is:

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

These triggers provide four execution paths:

| Trigger | Purpose |
|---|---|
| `push` | Execute when changes are pushed to `main` |
| `pull_request` | Validate pull requests targeting `main` |
| `workflow_dispatch` | Allow manual execution |
| `schedule` | Execute on a configured schedule |

The exact schedule is configuration-specific and can be changed independently of the pipeline architecture.

---

# 9. Action Inputs

GitHub Actions allows action behavior to be configured through the `with` block.

For example:

```yaml
- uses: actions/checkout@v4
  with:
    fetch-depth: 0
```

The implementation uses this mechanism to control action behavior without replacing the action with custom shell commands.

The general model is:

```text
uses
  ↓
Action
  ↓
with
  ↓
Action Inputs
```

---

# 10. Repository Permissions

The workflow explicitly restricts repository-content access:

```yaml
permissions:
  contents: read
```

This configures the workflow's `GITHUB_TOKEN` with read-only repository-content access.

The implementation therefore avoids unnecessarily granting repository write permissions.

The intended security model is:

```text
GitHub Actions
      │
      ▼
GITHUB_TOKEN
      │
      └── contents: read
```

This is part of the project's least-privilege approach.

---

# 11. Artifact Persistence

GitHub-hosted runners are ephemeral.

Files created during one job do not automatically become persistent project artifacts.

The implementation therefore uses GitHub Actions artifact storage for important outputs.

The general pattern is:

```text
Runner
  │
  ▼
Generated Output
  │
  ▼
upload-artifact
  │
  ▼
GitHub Artifact Storage
```

Two important artifacts are used in this project:

```text
vprofile-app
trivy-scan-results
```

---

# 12. Build Artifact

The Build job produces a WAR file under the Maven target directory.

The artifact upload step follows the pattern:

```yaml
- name: Upload Artifact
  uses: actions/upload-artifact@v4
  with:
    name: vprofile-app
    path: target/*.war
```

The implementation therefore preserves the generated application artifact beyond the lifetime of the Build runner.

The artifact can be accessed from the GitHub Actions workflow run.

---

# 13. Conditional Execution

GitHub Actions supports conditions through `if`.

The implementation uses conditions to control when particular stages execute.

The most important condition is the publishing restriction:

```yaml
if: github.ref == 'refs/heads/main'
```

The full Git reference is used:

```text
refs/heads/main
```

rather than simply:

```text
main
```

This creates a branch-specific execution boundary.

---

# 14. Security Scan Implementation

The Security Scan job was added after the Build and Testing foundation had been established.

Its dependency is:

```yaml
needs: Build
```

The resulting topology is:

```text
             ┌──► Testing
Build ───────┤
             └──► Security_Scan
```

Testing and Security Scan do not depend on one another, so they can execute independently after Build.

---

# 15. Trivy Integration

The project uses the Trivy GitHub Action for filesystem vulnerability scanning.

The demonstrated configuration is:

```yaml
- name: Run Trivy vulnerability scanner
  uses: aquasecurity/trivy-action@0.28.0
  with:
    scan-type: fs
    scan-ref: .
    format: json
    exit-code: 0
    output: trivy-results.json
```

The configuration means:

| Input | Purpose |
|---|---|
| `scan-type: fs` | Perform a filesystem scan |
| `scan-ref: .` | Scan the checked-out repository |
| `format: json` | Produce machine-readable JSON |
| `exit-code: 0` | Do not fail the step because of vulnerabilities |
| `output` | Write results to `trivy-results.json` |

The scan therefore follows:

```text
Repository
    ↓
Trivy
    ↓
trivy-results.json
```

---

# 16. Trivy Result Artifact

The generated Trivy result is uploaded as a workflow artifact:

```yaml
- name: Upload Trivy scan results as artifact
  uses: actions/upload-artifact@v4
  with:
    name: trivy-scan-results
    path: trivy-results.json
```

The resulting flow is:

```text
Trivy
  ↓
trivy-results.json
  ↓
upload-artifact
  ↓
trivy-scan-results
```

This allows the scan output to remain available after the runner has been destroyed.

---

# 17. Security Gate Boundary

The current Trivy configuration deliberately uses:

```yaml
exit-code: 0
```

Therefore the implementation reports vulnerabilities without turning the scan into a blocking quality gate.

The current behavior is:

```text
Scan
 ↓
Generate Report
 ↓
Upload Result
 ↓
Continue
```

A future implementation could change the exit code behavior and introduce explicit vulnerability thresholds.

For example, a future security policy could conceptually become:

```text
Scan
 ↓
Evaluate Vulnerabilities
 ↓
Policy
 ├── Pass
 └── Fail
```

That is a future enhancement rather than a current project capability.

---

# 18. Amazon ECR Preparation

The publishing stage requires an Amazon ECR repository.

The practical uses an ECR repository for the application image.

The architecture is:

```text
GitHub Actions
      │
      ▼
Docker Image
      │
      ▼
Amazon ECR Repository
```

The AWS region used by the workflow must correspond to the region containing the ECR repository.

---

# 19. AWS IAM Configuration

The publishing workflow requires AWS credentials capable of authenticating with ECR.

The demonstrated implementation uses an IAM identity for the GitHub Actions integration.

The credentials consist of:

```text
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
```

The source material uses IAM access keys for the learning implementation.

These credentials are not stored directly in the workflow file.

---

# 20. GitHub Environment Configuration

The AWS publishing job uses a GitHub Environment:

```yaml
environment: production
```

The environment contains:

### Secrets

```text
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
```

### Variable

```text
AWS_REGION
```

The distinction is:

```text
Sensitive
   ↓
GitHub Secret

Non-sensitive configuration
   ↓
GitHub Variable
```

The workflow retrieves these values through expressions rather than hardcoding credentials.

---

# 21. AWS Credential Configuration

The workflow uses the AWS credential configuration action:

```yaml
- name: Configure AWS credentials
  uses: aws-actions/configure-aws-credentials@v1
  with:
    aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
    aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    aws-region: ${{ vars.AWS_REGION }}
```

The implementation therefore creates the following chain:

```text
GitHub Environment
       │
       ▼
Secrets / Variables
       │
       ▼
AWS Credential Action
       │
       ▼
AWS Session
```

---

# 22. Amazon ECR Login

After AWS credentials are configured, the workflow authenticates Docker with ECR:

```yaml
- name: Login to Amazon ECR
  id: login-ecr
  uses: aws-actions/amazon-ecr-login@v1
```

The step receives an ID:

```text
login-ecr
```

This allows subsequent steps to access outputs generated by that step.

The registry output is referenced as:

```text
${{ steps.login-ecr.outputs.registry }}
```

This demonstrates step-to-step data flow inside a GitHub Actions job.

---

# 23. Dockerfile Adaptation

The Dockerfile requires adaptation for the GitHub Actions environment.

The important difference is that the GitHub Actions runner already contains the application source because the workflow has performed a repository checkout.

The container build therefore does not need to obtain the application source independently.

The Docker build uses the repository source through Docker's build context.

The relevant pattern is:

```dockerfile
COPY ./ /app
WORKDIR /app
RUN mvn install
```

The multi-stage build then copies the generated WAR into the final runtime image.

---

# 24. Docker Build Context

The Docker build context is the repository root.

The implementation therefore uses:

```text
.
```

as the build context.

Conceptually:

```text
Repository Root
      │
      ├── application source
      ├── Dockerfile
      └── other build files
              │
              ▼
        Docker Build Context
```

This is important because:

```dockerfile
COPY ./ /app
```

can only access files within the Docker build context.

Using the Dockerfile's directory as the context instead of the repository root can therefore cause required application files to be unavailable to the build.

---

# 25. Multi-Stage Docker Build

The Docker image uses a multi-stage build.

The conceptual structure is:

```text
Stage 1
Build Environment
      │
      ├── Application Source
      ├── Maven
      └── Maven Build
             │
             ▼
        WAR Artifact
             │
             ▼
Stage 2
Runtime Environment
      │
      ├── Tomcat
      └── WAR Artifact
             │
             ▼
        Runtime Image
```

The purpose is to keep build tooling in the build stage rather than requiring it in the final runtime image.

The project uses this pattern as part of the Docker image construction process.

---

# 26. Build & Publish Job

The final workflow job is:

```text
BUILD_AND_PUBLISH
```

Its dependencies are:

```yaml
needs: [Build, Testing, Security_Scan]
```

The job is also restricted to the main branch:

```yaml
if: github.ref == 'refs/heads/main'
```

The implementation therefore establishes:

```text
Build
  +
Testing
  +
Security Scan
       ↓
BUILD_AND_PUBLISH
```

---

# 27. Build & Publish Sequence

The publishing job performs the following sequence:

```text
1. Checkout Source
       ↓
2. Configure AWS Credentials
       ↓
3. Login to ECR
       ↓
4. Resolve Image Identity
       ↓
5. Docker Build
       ↓
6. Docker Push
```

The resulting image is stored in Amazon ECR.

---

# 28. Step Outputs

The ECR login action exposes the registry through a step output.

The step is identified as:

```yaml
id: login-ecr
```

The output is consumed using:

```text
${{ steps.login-ecr.outputs.registry }}
```

The resulting image naming components are therefore:

```text
Registry
    +
Repository
    +
Tag
```

---

# 29. Image Naming

The final image follows:

```text
<registry>/<repository>:<tag>
```

The implementation uses:

```text
Registry
  =
steps.login-ecr.outputs.registry

Repository
  =
ECR repository

Tag
  =
github.sha
```

This produces an image reference conceptually similar to:

```text
<registry>/vprofile-app-image:<commit-sha>
```

---

# 30. Commit-SHA Tagging

The image tag uses the GitHub commit SHA:

```text
github.sha
```

The relationship is:

```text
Git Commit
    │
    ▼
github.sha
    │
    ▼
Docker Image Tag
    │
    ▼
Amazon ECR
```

This provides source-to-image traceability.

The published container can therefore be associated with the exact Git revision used by the workflow.

---

# 31. Docker Build Command

The publishing job builds the image using the repository root as the Docker build context.

Conceptually:

```bash
docker build   -f docker-files/app/multistage   -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG   .
```

The important implementation detail is the final:

```text
.
```

This establishes the repository root as the build context.

---

# 32. Docker Push

After the image has been built, the image is pushed to ECR:

```bash
docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
```

The final publishing sequence is:

```text
Docker Build
     ↓
Docker Image
     ↓
Commit-SHA Tag
     ↓
Docker Push
     ↓
Amazon ECR
```

---

# 33. Complete Implementation Flow

The complete implementation can be summarized as:

```text
GitHub Repository
       │
       ▼
Workflow Trigger
       │
       ▼
     Build
       │
       ├───────────────┐
       ▼               ▼
   Testing       Security Scan
       │               │
       │               ▼
       │        Trivy Artifact
       │
       └───────┬───────┘
               ▼
       BUILD_AND_PUBLISH
               │
               ▼
      Configure AWS
               │
               ▼
          ECR Login
               │
               ▼
         Docker Build
               │
               ▼
       Commit-SHA Tag
               │
               ▼
          Docker Push
               │
               ▼
          Amazon ECR
```

---

# 34. Implementation Troubleshooting

## 34.1 YAML Syntax Problems

A malformed workflow can fail before the jobs execute.

The implementation approach is:

```text
Workflow Error
      ↓
Read Reported Location
      ↓
Inspect YAML Structure
      ↓
Correct Syntax
      ↓
Commit
      ↓
Push
      ↓
Re-run Workflow
```

Common YAML problems include:

- incorrect indentation
- malformed lists
- incorrect mapping structure
- misplaced workflow keys

---

## 34.2 Incorrect Branch Reference

The publishing condition requires the full Git reference:

```text
refs/heads/main
```

rather than:

```text
main
```

Therefore the condition:

```yaml
if: github.ref == 'refs/heads/main'
```

is important for the intended branch-specific publishing behavior.

---

## 34.3 Docker Build Context Problems

If the Docker build is executed with the wrong context, the Dockerfile may not be able to access the application source.

The expected relationship is:

```text
Repository Root
      │
      ▼
Docker Build Context
      │
      ▼
COPY ./ /app
```

The build command must therefore use the repository root as its context.

---

## 34.4 AWS Region Mismatch

The configured:

```text
AWS_REGION
```

must correspond to the region containing the ECR repository.

A mismatch can prevent ECR authentication or repository access.

---

## 34.5 Secret Name Mismatch

The names configured in GitHub Environment Secrets must match the names referenced in the workflow.

Expected names include:

```text
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
```

A mismatch causes the workflow to receive incorrect or unavailable credential values.

---

# 35. Implementation Security Boundaries

The implementation deliberately keeps sensitive information outside the workflow source.

Credentials are represented through:

```text
${{ secrets.AWS_ACCESS_KEY_ID }}
${{ secrets.AWS_SECRET_ACCESS_KEY }}
```

rather than literal credential values.

The repository should never contain:

```text
AWS access keys
AWS secret keys
GitHub secret values
```

These values belong exclusively in the appropriate secret-management configuration.

---

# 36. Implementation Ownership Boundary

The implementation represented by this repository includes:

```text
Personally Configured / Performed
│
├── GitHub Actions workflow
├── Job dependencies
├── Workflow triggers
├── Conditions
├── Artifact handling
├── Permissions
├── Trivy integration
├── AWS/ECR integration
├── GitHub Environment configuration
├── Dockerfile adaptation
├── Docker build
├── Image tagging
└── Image publishing
```

The following remain outside the ownership boundary:

```text
Supplied / Existing Workload
│
├── VProfile application business logic
├── Original application architecture
└── Application functionality
```

The repository should therefore describe the application as the **workload** and the GitHub Actions pipeline as the **engineering project**.

---

# 37. Implementation Boundary

The current implementation ends at:

```text
Docker Image
     ↓
Amazon ECR
```

It does not implement:

```text
ECR
 ↓
ECS / EKS
 ↓
Running Application
```

It also does not implement:

- Terraform
- Kubernetes
- GitOps
- automated deployment
- automated rollback
- production observability

These are future project directions rather than current implementation claims.

---

# 38. Implementation Summary

The project progressed from a basic two-job workflow:

```text
Build
  ↓
Testing
```

to a complete CI/CD workflow:

```text
Build
  ├──► Testing
  │
  └──► Security Scan
           │
           ▼
    BUILD_AND_PUBLISH
           │
           ▼
       Docker
           │
           ▼
      Amazon ECR
```

The implementation demonstrates the practical integration of:

- GitHub Actions
- Maven
- GitHub Actions artifacts
- workflow triggers
- conditions
- repository permissions
- Trivy
- GitHub Environment Secrets
- AWS IAM credentials
- Amazon ECR
- Docker
- multi-stage builds
- commit-SHA image tagging

The central implementation principle is:

> **Build and validate the application first, then publish a traceable container image only through the controlled publishing stage.**
