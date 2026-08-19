# Limitations & Future Work

[← Back to README](../README.md)

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/2533c8e7-109a-4461-b924-8f7f211435de" />


## 1. Purpose

This document defines the boundaries of the VProfile GitHub Actions CI/CD project and identifies logical improvements that could extend the current implementation.

The purpose is not to make the project appear more complete than it is.

The purpose is to clearly distinguish:

```text
Completed Capability
        ↓
Current Limitation
        ↓
Logical Future Capability
```

Future capabilities described here are **not implemented capabilities** and should not be presented as completed work.

---

# 2. Current Project Boundary

The current project demonstrates:

```text
GitHub
   ↓
GitHub Actions
   ↓
Build
   ↓
Testing
   ↓
Security Scan
   ↓
Publishing Gate
   ↓
Docker Build
   ↓
Amazon ECR
```

The pipeline ends at:

```text
Docker Image
     ↓
Amazon ECR
```

It does not continue into a runtime deployment platform.

The project therefore demonstrates **CI/CD and container image publication**, not complete application deployment.

---

# 3. Limitation: The VProfile Application Is an Existing Workload

## Current State

The VProfile application is used as the workload processed by the CI/CD pipeline.

The project does not represent development of the application's business logic or original application architecture.

The engineering boundary is:

```text
Existing Application Workload
          +
     CI/CD Engineering
```

rather than:

```text
Application Development
          +
     CI/CD Engineering
```

## Impact

The repository should not claim ownership of:

- VProfile business logic
- Original application architecture
- Application functionality
- Application features that were supplied as part of the workload

The engineering contribution represented here is the delivery platform surrounding the workload.

## Future Work

A future project could use an application personally developed and owned end-to-end.

That would allow the repository to demonstrate:

```text
Application Development
        ↓
Source Control
        ↓
CI/CD
        ↓
Containerization
        ↓
Deployment
```

while maintaining a clear ownership boundary.

---

# 4. Limitation: No Runtime Deployment

## Current State

The pipeline publishes the Docker image to Amazon ECR:

```text
GitHub Actions
      ↓
Docker Build
      ↓
Amazon ECR
```

There is no deployment stage after ECR.

## Not Claimed

The project does not demonstrate:

- ECS deployment
- EKS deployment
- Kubernetes deployment
- GitOps
- automated application rollout
- automated rollback
- production runtime orchestration

## Impact

The workflow proves that a validated source revision can become a container image and that the image can be published to the configured registry.

It does not prove that the application can be successfully started and operated in a production runtime environment.

## Future Work

A natural next stage would be:

```text
GitHub Actions
      ↓
Amazon ECR
      ↓
ECS / EKS
      ↓
Running Application
```

A deployment stage could consume the exact image URI produced by the publishing job.

---

# 5. Limitation: Trivy Scan Is Informational Rather Than Blocking

## Current State

The Trivy security scan is configured with:

```yaml
exit-code: 0
```

The resulting behavior is:

```text
Security Scan
      ↓
Vulnerability Report
      ↓
Pipeline Continues
```

The current pipeline should not be described as enforcing a security policy.

A successful Trivy job means:

> The configured scan executed and produced its result.

It does **not** mean:

> The application contains no vulnerabilities.

## Future Work

A mature implementation could introduce a defined vulnerability policy:

```text
Trivy
  ↓
Vulnerability Results
  ↓
Severity / Policy Evaluation
  ├── Pass → Continue
  └── Fail → Stop Pipeline
```

The policy could define which vulnerability severities or conditions are release-blocking.

The scan could then become a true promotion gate.

---

# 6. Limitation: Source Filesystem Scan Rather Than Container Image Scan

## Current State

The configured Trivy scan uses:

```yaml
scan-type: fs
scan-ref: .
```

This scans the checked-out source filesystem and dependency manifests.

The current security stage occurs before the Docker image is built.

## Impact

The current scan does not represent a complete security assessment of the final container image.

The final image introduces additional layers such as:

```text
Base Image
    +
OS Packages
    +
Application Runtime
    +
Application Dependencies
```

## Future Work

Add a container-image scanning stage:

```text
Docker Build
      ↓
Container Image
      ↓
Trivy Image Scan
      ↓
Security Policy
      ↓
Publish / Reject
```

This would complement the existing filesystem scan.

---

# 7. Limitation: Long-Lived AWS Access Keys

## Current State

The demonstrated AWS authentication model uses:

```text
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
```

stored as GitHub Environment Secrets.

This is the learning implementation used for the project.

## Impact

Long-lived credentials create additional credential-management risk.

The workflow must depend on:

```text
GitHub Secrets
      ↓
AWS Access Keys
      ↓
AWS Authentication
```

Credential rotation, exposure, and lifecycle management therefore become important operational concerns.

## Future Work

Use GitHub Actions OIDC federation:

```text
GitHub Actions
      ↓
OIDC Identity Token
      ↓
AWS IAM Role
      ↓
Short-Lived Credentials
      ↓
Amazon ECR
```

This removes the need to store long-lived AWS access keys in GitHub Secrets.

---

# 8. Limitation: Broad ECR IAM Permission

## Current State

The learning implementation uses a broad ECR permission model for the publishing identity.

This is sufficient for demonstrating the GitHub Actions → ECR integration, but it is broader than necessary for a narrowly scoped publishing workflow.

## Impact

The identity has broader ECR permissions than are strictly necessary.

The current model is therefore suitable for learning the integration but is not the strongest least-privilege design.

## Future Work

Replace broad ECR permissions with a repository-scoped IAM policy.

Conceptually:

```text
GitHub Actions Identity
        ↓
Restricted IAM Role
        ↓
Specific ECR Repository
        ↓
Push / Required Operations Only
```

Combined with OIDC, this would produce a stronger authentication and authorization model.

---

# 9. Limitation: No Infrastructure as Code

## Current State

The project does not provide Terraform or another Infrastructure-as-Code implementation for the complete environment.

The repository therefore does not provide:

```text
terraform plan
      ↓
terraform apply
      ↓
Complete CI/CD Environment
```

## Impact

Recreating the complete environment requires manual configuration of resources such as:

- ECR
- IAM configuration
- GitHub Environment configuration
- GitHub repository configuration

## Future Work

Introduce Infrastructure as Code.

A future implementation could manage:

```text
Terraform
    ↓
AWS Infrastructure
    ├── IAM
    ├── ECR
    ├── Networking
    └── Runtime Infrastructure
```

and maintain the infrastructure configuration alongside the application delivery workflow.

---

# 10. Limitation: No Automated Deployment or Rollback

## Current State

The current pipeline stops after:

```text
Docker Push
     ↓
Amazon ECR
```

There is no automated deployment controller consuming the published image.

## Impact

A successful image publication does not automatically result in a new application version running in a runtime environment.

There is also no automated mechanism to return to a previous application version.

## Future Work

Extend the pipeline:

```text
Build
  ↓
Test
  ↓
Security
  ↓
ECR
  ↓
Deployment
  ↓
Health Validation
  ↓
Success / Rollback
```

A production-oriented deployment stage could introduce:

- versioned deployment artifacts
- health checks
- deployment verification
- rollback logic
- controlled promotion

---

# 11. Limitation: No Production Runtime Observability

## Current State

The project validates the CI/CD workflow and the resulting ECR image.

It does not implement application runtime observability.

There is no demonstrated:

```text
Application
    ↓
Metrics
    ↓
Logs
    ↓
Traces
    ↓
Alerting
```

## Impact

The project cannot demonstrate how an operator would detect and diagnose problems after deployment.

## Future Work

After introducing a runtime deployment stage, add:

```text
Application
      ↓
Logs
      ↓
Metrics
      ↓
Health Checks
      ↓
Alerts
      ↓
Operational Response
```

The exact observability platform would depend on the selected runtime architecture.

---

# 12. Limitation: No Production-Grade Deployment Environment

## Current State

The current project demonstrates the delivery path up to ECR.

It does not demonstrate:

- production environment provisioning
- multi-environment promotion
- staging-to-production promotion
- deployment approvals
- runtime scaling
- high availability
- disaster recovery

## Impact

The project demonstrates CI/CD mechanics but does not establish production operational maturity.

## Future Work

A broader delivery platform could introduce:

```text
Developer Change
      ↓
CI
      ↓
Security / Quality Gates
      ↓
Container Registry
      ↓
Development
      ↓
Staging
      ↓
Approval
      ↓
Production
```

This would turn the current CI pipeline into a more complete software delivery platform.

---

# 13. Limitation: No Immutable Runtime Deployment Model

## Current State

The workflow uses the Git commit SHA as the Docker image tag:

```text
<registry>/<repository>:<commit-sha>
```

This provides useful source-to-image traceability.

However, the project stops at image publication and does not demonstrate a runtime deployment that consumes this immutable reference.

## Impact

The project demonstrates immutable identification of the image but not immutable deployment.

## Future Work

A future deployment pipeline could carry the exact image reference through the deployment process:

```text
Git Commit
    ↓
Commit SHA
    ↓
Docker Image
    ↓
ECR
    ↓
Versioned Deployment Configuration
    ↓
Runtime
```

This would strengthen reproducibility and rollback capabilities.

---

# 14. Limitation: No Full Release Promotion Model

## Current State

The current workflow primarily distinguishes:

```text
Validation
    ↓
Publishing
```

and uses the `main` branch condition to control publication.

It does not demonstrate a multi-stage promotion process.

## Future Work

A more mature delivery model could use:

```text
Pull Request
     ↓
CI Validation
     ↓
Main
     ↓
Build Image
     ↓
ECR
     ↓
Staging
     ↓
Approval / Policy
     ↓
Production
```

This would separate artifact creation from environment promotion.

---

# 15. Limitation: GitHub-Hosted Runner Ephemeral State

## Current State

GitHub-hosted runners are ephemeral.

Files created during a job do not persist automatically after the runner is destroyed.

The project addresses this for important outputs by using GitHub Actions artifacts.

## Impact

Any output that is required after the job must be deliberately persisted.

The workflow therefore cannot rely on local runner state as durable storage.

## Future Work

For more complex pipelines, establish an explicit artifact lifecycle:

```text
Build
  ↓
Artifact
  ↓
Artifact Storage
  ↓
Downstream Job
  ↓
Deployment
```

For container delivery, the registry itself becomes the durable image store.

---

# 16. Limitation: Limited Security Policy Enforcement

The project includes:

```text
Repository Permission Restriction
+
Trivy Scan
+
Environment-Scoped Secrets
```

These are meaningful security controls.

However, the project does not demonstrate a complete security governance model.

It does not establish:

- organization-wide security policy
- centralized vulnerability policy
- container image admission controls
- comprehensive secret rotation automation
- runtime security controls
- enterprise identity federation

## Future Work

Security could evolve into a layered model:

```text
Source
  ↓
Dependency Scan
  ↓
Filesystem Scan
  ↓
Container Image Scan
  ↓
Policy Gate
  ↓
Registry
  ↓
Runtime Security
```

---

# 17. Limitation: No Complete Production CI/CD Platform

The current project should be understood as a focused CI/CD implementation rather than a complete enterprise delivery platform.

The demonstrated capability is:

```text
Source
  ↓
Build
  ↓
Validate
  ↓
Scan
  ↓
Containerize
  ↓
Publish
```

It does not attempt to demonstrate every DevOps capability.

In particular, it does not currently include:

```text
Infrastructure as Code
Runtime Deployment
GitOps
Production Observability
Automated Rollback
Enterprise Identity Federation
Multi-Environment Promotion
```

This boundary is intentional.

---

# 18. Future Work Roadmap

The logical evolution of the current project can be represented as:

```text
CURRENT
  │
  ▼
GitHub Actions CI/CD
  │
  ├── Build
  ├── Test
  ├── Checkstyle
  ├── Trivy
  ├── Docker
  └── ECR
  │
  ▼
NEXT
  │
  ├── OIDC Authentication
  ├── Repository-Scoped IAM
  ├── Enforced Security Policy
  └── Container Image Scanning
  │
  ▼
DEPLOYMENT
  │
  ├── ECS / EKS
  ├── Versioned Deployment
  ├── Health Validation
  └── Rollback
  │
  ▼
PLATFORM
  │
  ├── Terraform
  ├── Multi-Environment Promotion
  ├── Observability
  └── Operational Automation
```

Each stage builds naturally on the current architecture.

---

# 19. Recommended Future Priority

Not all future improvements have equal value.

A reasonable progression is:

### Priority 1 — Replace Long-Lived AWS Keys

```text
IAM Access Keys
      ↓
GitHub OIDC
      ↓
Short-Lived AWS Role Credentials
```

This addresses an important credential-management limitation.

---

### Priority 2 — Enforce Security Policy

Move from:

```text
Trivy
  ↓
Report
```

toward:

```text
Trivy
  ↓
Policy
  ↓
Pass / Fail
```

This turns the security scan into an actual release control.

---

### Priority 3 — Scan the Final Container Image

Move from source filesystem scanning alone toward:

```text
Source Scan
      +
Container Image Scan
```

This provides broader security coverage.

---

### Priority 4 — Add Deployment

Extend:

```text
ECR
  ↓
Runtime Platform
```

and make the exact published image the deployment input.

---

### Priority 5 — Add Infrastructure as Code

Once the runtime architecture is defined, codify the infrastructure using Terraform or an equivalent IaC approach.

---

### Priority 6 — Add Observability and Rollback

Finally, introduce:

```text
Deployment
   ↓
Health
   ↓
Observability
   ↓
Rollback
```

to move the project toward a more complete production delivery system.

---

# 20. What This Project Can Honestly Claim

When supported by personal execution evidence, the project can claim:

> GitHub Actions was used to automate the build and validation workflow around an existing Java application workload.

> Maven testing and Checkstyle were integrated into the CI workflow.

> Trivy was integrated as a filesystem vulnerability scan, with machine-readable results persisted as a workflow artifact.

> The workflow separates validation jobs from the publishing job using explicit GitHub Actions dependencies.

> The Docker image is built from the repository source using a multi-stage Docker build.

> AWS authentication and Amazon ECR integration were configured for GitHub Actions.

> The published image is tagged using the Git commit SHA, creating source-to-image traceability.

These claims describe the demonstrated engineering work without extending the project's scope beyond what was actually implemented.

---

# 21. What This Project Should Not Claim

The repository should not claim:

- production-ready deployment
- Kubernetes expertise demonstrated by this project
- ECS deployment demonstrated by this project
- Terraform infrastructure demonstrated by this project
- GitOps demonstrated by this project
- zero-downtime deployment
- automated rollback
- production-grade observability
- complete vulnerability remediation
- vulnerability-free application
- enterprise-grade AWS identity federation
- high availability
- disaster recovery
- ownership of the VProfile application's business logic

These capabilities are either outside the project's scope or explicitly identified as future improvements.

---

# 22. Final Boundary

The current project should be understood as:

```text
             CI/CD ENGINEERING
                    │
                    ▼
              GitHub Actions
                    │
          ┌─────────┴─────────┐
          ▼                   ▼
       Validation          Security
          │                   │
          └─────────┬─────────┘
                    ▼
               Docker Build
                    │
                    ▼
                Amazon ECR
```

The next major boundary is:

```text
Amazon ECR
     ↓
Runtime Deployment
```

That boundary is intentionally left for future work.

The project therefore demonstrates a focused and defensible DevOps capability:

> **Automating application validation, security scanning, containerization, and container image publication through GitHub Actions, while maintaining explicit ownership boundaries and source-to-image traceability.**

Future work should extend this foundation rather than obscure the fact that the current implementation intentionally stops at the container registry.
