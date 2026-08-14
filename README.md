# AWS VProfile CI/CD

> AWS-native CI/CD delivery workflow for an existing Java application workload using Bitbucket, AWS CodePipeline, AWS CodeBuild, Elastic Beanstalk, and Amazon RDS.

## Overview

This project implements an end-to-end AWS-native CI/CD workflow around the VProfile application workload.

The delivery flow connects source control, automated builds, artifact handling, and application deployment:

```text
Bitbucket
    │
    │ Commit
    ▼
AWS CodePipeline
    │
    ▼
AWS CodeBuild
    │
    │ Maven build
    │ buildspec.yml
    ▼
Deployable Artifact
    │
    ▼
AWS Elastic Beanstalk
    │
    ▼
VProfile Application
    │
    ▼
Amazon RDS
```

The resulting workflow allows a change committed to the configured Bitbucket branch to trigger the pipeline automatically and progress through source, build, and deployment stages.

---

## Application Ownership Boundary

The VProfile application was used as an existing application workload for this project.

The engineering work represented in this repository focuses on the **DevOps and AWS delivery environment around the application**, including source-control integration, build automation, CI/CD orchestration, deployment, troubleshooting, and validation.

This project does **not** claim ownership of:

- VProfile application development
- VProfile business logic
- The application's original architecture
- The application's authentication implementation
- The original application source code

The application and course-provided materials should therefore not be treated as original application-development work by the project author.

---

## Engineering Contribution

The project demonstrates practical work across the application delivery lifecycle:

### Source Control

- Configured Bitbucket as the source repository.
- Configured SSH-based Git authentication.
- Migrated the existing repository workflow to Bitbucket.
- Configured the CI/CD source branch for automated pipeline triggering.

### Build Automation

- Configured AWS CodeBuild.
- Configured the build process using `buildspec.yml`.
- Used Maven to build the Java application.
- Configured environment-specific application settings during the build.
- Configured build artifact generation.
- Configured CloudWatch-based build logging.

### CI/CD Orchestration

- Created an AWS CodePipeline.
- Connected Bitbucket as the source provider.
- Connected CodeBuild as the build provider.
- Connected Elastic Beanstalk as the deployment target.
- Configured event-driven pipeline execution from source changes.

### AWS Integration

- Configured the application environment using Elastic Beanstalk.
- Configured Amazon RDS for the application database.
- Configured security-group connectivity between the application and database tiers.
- Configured IAM permissions required for pipeline deployment.

### Troubleshooting

The implementation included practical troubleshooting of:

- A CodeBuild failure caused by a malformed `sed` command.
- A CodePipeline deployment permission issue caused by missing Elastic Beanstalk permissions in the automatically created service role.

The failures were diagnosed, corrected, and revalidated as part of the delivery workflow.

---

## Architecture

The project separates the major responsibilities of the delivery system:

| Component | Responsibility |
|---|---|
| Bitbucket | Source repository and change event |
| AWS CodePipeline | CI/CD orchestration |
| AWS CodeBuild | Application build |
| `buildspec.yml` | Build instructions |
| Maven | Java application build |
| Artifact | Deployable application output |
| Elastic Beanstalk | Application deployment and runtime |
| Amazon RDS | Application database |

CodePipeline acts as the orchestrator rather than performing the build or running the application itself.

For the detailed architecture and component relationships:

**[→ Architecture](docs/architecture.md)**

---

## Validation

The implementation was validated across multiple stages:

```text
Source
  ↓
Bitbucket commit detected
  ↓
CodePipeline triggered
  ↓
CodeBuild completed successfully
  ↓
Application artifact deployed
  ↓
Elastic Beanstalk deployment completed
  ↓
Application became accessible
  ↓
Application successfully connected to RDS
```

The final validation also included a source change that automatically triggered a new pipeline execution.

Detailed validation methodology and evidence mapping:

**[→ Validation](docs/validation.md)**

---

## Key Engineering Lessons

This project demonstrates several reusable DevOps engineering patterns:

### Orchestration vs. Execution

CodePipeline orchestrates the delivery workflow while CodeBuild performs the application build.

```text
CodePipeline
     │
     ├── Source
     ├── Build
     └── Deploy
            │
            └── CodeBuild performs the build
```

### Test Components Before Integration

The CodeBuild project was validated independently before being integrated into CodePipeline.

This isolates component-level failures before introducing pipeline-level complexity.

### IAM Permissions Must Match the Complete Workflow

An automatically created CodePipeline service role did not initially contain the required Elastic Beanstalk permissions.

The deployment failure demonstrated the importance of verifying that service-role permissions cover every integration used by a pipeline.

### Event-Driven Delivery

After configuration, a source commit can initiate the delivery workflow automatically:

```text
Bitbucket Commit
       ↓
Pipeline Trigger
       ↓
Build
       ↓
Deploy
```

### Dependency-Aware Cleanup

AWS resource relationships also affect deletion order.

For example, security-group references must be removed before dependent resources can be deleted successfully.

---

## Project Boundaries

This project demonstrates a working AWS-native CI/CD implementation around an existing application workload.

It does **not** establish:

- Application development ownership.
- Infrastructure as Code using Terraform or another IaC framework.
- A complete enterprise CI/CD platform.
- A comprehensive automated testing strategy.
- Production-grade secret management.
- Multi-environment promotion workflows.
- Multi-region deployment.
- Disaster recovery architecture.
- Enterprise production readiness.

These capabilities represent potential future improvements rather than completed project functionality.

---

## Technologies

### AWS

- AWS CodePipeline
- AWS CodeBuild
- AWS Elastic Beanstalk
- Amazon RDS
- Amazon S3
- Amazon CloudWatch
- AWS IAM
- AWS CodeConnections

### Source Control & Build

- Git
- Bitbucket
- SSH
- Maven
- Java

---

## Repository Navigation

| Document | Purpose |
|---|---|
| [Architecture](docs/architecture.md) | System architecture, component relationships, traffic and delivery flow |
| [Implementation](docs/implementation.md) | Implementation sequence, configuration, deployment, and troubleshooting |
| [Validation](docs/validation.md) | Validation strategy, results, and evidence mapping |
| [Limitations & Future Work](docs/limitations-and-future-work.md) | Project boundaries and potential future evolution |
| [Evidence](evidence/screenshots/) | High-signal evidence from the completed environment |

---

## Evidence

Evidence in this repository is limited to high-signal artifacts that demonstrate meaningful implementation or validation.

Examples include:

- Successful CodeBuild execution
- Successful CodePipeline execution
- Elastic Beanstalk deployment
- Application validation
- Bitbucket commit triggering the pipeline
- Relevant troubleshooting evidence

Only evidence from the personally completed environment should be used as proof of execution.

---

## Project Summary

This project demonstrates the implementation of an AWS-native application delivery workflow:

```text
Source
  ↓
Build
  ↓
Artifact
  ↓
Orchestration
  ↓
Deployment
  ↓
Runtime Validation
```

The primary engineering focus is the integration and operation of the delivery pipeline around an existing application workload.

---

[← Back to Project Root](.)
