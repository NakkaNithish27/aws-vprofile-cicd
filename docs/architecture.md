# Architecture

[← Back to README](../README.md)

## 1. Architecture Overview

This project implements an AWS-native CI/CD delivery workflow around the existing VProfile Java application workload.

The architecture separates the **software delivery path** from the **application runtime path**:

```text
                         CI/CD DELIVERY FLOW
┌──────────────┐
│   Bitbucket  │
│   aws-ci     │
└──────┬───────┘
       │
       │ Commit event
       ▼
┌──────────────────┐
│  AWS CodePipeline│
│   Orchestrator   │
└──────┬───────────┘
       │
       │ Trigger
       ▼
┌──────────────────┐
│   AWS CodeBuild  │
│   Build Engine   │
└──────┬───────────┘
       │
       │ Deployable artifact
       ▼
┌──────────────────────┐
│ AWS Elastic Beanstalk│
│   Deployment Target  │
└──────────┬───────────┘
           │
           │ Runtime
           ▼
    ┌───────────────┐
    │   VProfile    │
    │  Application  │
    └───────┬───────┘
            │
            │ Database connection
            ▼
      ┌───────────┐
      │ Amazon RDS│
      │  Database │
      └───────────┘
```

The architecture uses Bitbucket as the source repository, CodePipeline as the orchestration layer, CodeBuild as the build engine, Elastic Beanstalk as the deployment target, and RDS as the application's runtime database.

---

## 2. Application Ownership Boundary

The VProfile application is an **existing application workload** used by the project.

The architecture documented here therefore focuses on the DevOps engineering surrounding the application:

```text
Existing Application
        │
        │ deployed/configured through
        ▼
AWS Delivery Architecture
```

This repository does not claim ownership of the VProfile application's:

- business logic
- original application architecture
- authentication implementation
- original source code

The architectural responsibility represented by this project is the **source → build → deployment → runtime integration** around that workload.

---

## 3. Source Layer — Bitbucket

The source code is hosted in a Bitbucket repository.

The project uses the `aws-ci` branch as the configured pipeline source branch.

```text
Developer
   │
   │ git commit / git push
   ▼
Bitbucket
   │
   │ aws-ci branch
   ▼
CodePipeline
```

The project migrated the existing VProfile repository workflow from GitHub to Bitbucket. Bitbucket is used specifically as the **source-control platform**, while AWS services provide the CI/CD functionality.

Bitbucket's own CI/CD capabilities are not used in this architecture.

---

## 4. Event-Driven Trigger

The pipeline is event-driven.

A source change follows this control path:

```text
git push
   │
   ▼
Bitbucket
   │
   │ commit detected
   ▼
CodePipeline
   │
   ▼
Pipeline execution
```

The project does not depend on a manually started pipeline for normal operation.

A commit to the configured `aws-ci` branch automatically initiates the pipeline execution. The practical validation uses a small change to `README.md` to demonstrate this behavior.

### Control flow

```text
Source change
     ↓
Pipeline trigger
     ↓
Build
     ↓
Deployment
```

This is distinct from the application's runtime data flow.

---

## 5. CodePipeline — Orchestration Layer

AWS CodePipeline is the central orchestration component.

It connects the independent source, build, and deployment services:

```text
             CodePipeline
                  │
        ┌─────────┼─────────┐
        ▼         ▼         ▼
     Source     Build    Deploy
   Bitbucket   CodeBuild  Beanstalk
```

CodePipeline is responsible for:

1. Detecting the source change.
2. Obtaining the source revision.
3. Triggering CodeBuild.
4. Passing the resulting artifact through the pipeline.
5. Initiating deployment to Elastic Beanstalk.

CodePipeline therefore acts as the **glue between otherwise independent services** rather than being the component that performs the application build itself.

---

## 6. CodeBuild — Build Layer

AWS CodeBuild is the build engine.

Its responsibility is to transform the application source into a deployable artifact.

```text
Source
  │
  ▼
CodeBuild
  │
  ├── buildspec.yml
  ├── dependency setup
  ├── configuration
  ├── Maven build
  └── artifact creation
       │
       ▼
   Deployable artifact
```

The project uses `buildspec.yml` to define the build behavior.

The build includes environment-specific configuration of the application's `application.properties`, including RDS connection information, before the application is packaged. The resulting deployable artifact is a `.war` file.

---

## 7. Build Configuration

The build configuration follows the CodeBuild phase model:

```text
INSTALL
   │
   ▼
PRE_BUILD
   │
   ├── dependency/configuration preparation
   └── application configuration
   │
   ▼
BUILD
   │
   └── Maven build
   │
   ▼
POST_BUILD
   │
   ▼
ARTIFACT
```

The `sed` commands used during the build modify application configuration so the deployed application can connect to the configured RDS database.

A malformed `sed` command initially caused the CodeBuild project to fail. The failure was diagnosed from the build phase/log information, the missing delimiter was corrected, and the build was rerun successfully.

This troubleshooting is part of the implementation story rather than a separate architectural component.

---

## 8. Artifact Flow

The application source and the deployed application are separate stages of the delivery process.

```text
Bitbucket source
      │
      ▼
  CodeBuild
      │
      │ Maven
      ▼
  .war artifact
      │
      ▼
 CodePipeline
      │
      ▼
Elastic Beanstalk
```

The artifact is the handoff between the **build stage** and the **deployment stage**.

The pipeline therefore separates:

```text
Build responsibility
        ≠
Deployment responsibility
```

CodeBuild produces the application artifact.

CodePipeline orchestrates its movement to the deployment stage.

Elastic Beanstalk runs the deployed application.

---

## 9. Elastic Beanstalk — Deployment and Runtime Layer

Elastic Beanstalk is the deployment target.

The Beanstalk environment receives the application artifact and deploys it onto its managed application environment.

The project environment uses two instances with a 50% deployment batch size.

Conceptually:

```text
                Elastic Beanstalk
                       │
              ┌────────┴────────┐
              ▼                 ▼
        Application          Application
         Instance 1           Instance 2
```

During a rolling deployment:

```text
Batch 1
Instance 1 → new version
Instance 2 → existing version

        ↓

Batch 2
Instance 1 → new version
Instance 2 → new version
```

The practical material describes the deployment as a rolling update with one instance updated per batch.

---

## 10. Runtime Application Flow

Once deployment completes, the application runs independently of the CI/CD orchestration path.

The runtime relationship is:

```text
Elastic Beanstalk
       │
       ▼
VProfile Application
       │
       │ database connection
       ▼
    Amazon RDS
```

This is important:

> **RDS is not a stage in the CI/CD pipeline.**

RDS is part of the **application runtime architecture**.

The pipeline deploys the application; the deployed application then communicates with RDS at runtime.

---

## 11. Amazon RDS — Database Layer

Amazon RDS provides the relational database required by the VProfile application.

The database contains the schemas and tables required by the application.

The relationship is:

```text
VProfile Application
        │
        │ runtime database connection
        ▼
     Amazon RDS
```

RDS must exist before the deployed application can operate correctly because the application depends on its database.

The project therefore treats RDS as an infrastructure dependency of the runtime environment rather than as a CI/CD component.

---

## 12. Security Boundary

The application and database communicate through AWS security-group rules.

The important relationship is:

```text
Application / Beanstalk
        │
        │ authorized network access
        ▼
      RDS
```

The RDS security group contains a rule referencing the Beanstalk security group.

This creates an explicit dependency:

```text
Beanstalk Security Group
          │
          │ referenced by
          ▼
RDS Security Group
```

This relationship also affects cleanup.

The cross-reference must be removed before the Beanstalk environment can be deleted successfully.

---

## 13. IAM Boundary

CodePipeline uses an IAM service role to interact with the AWS services required by the pipeline.

Conceptually:

```text
CodePipeline
     │
     │ assumes
     ▼
IAM Service Role
     │
     ├── CodeBuild permissions
     ├── CodeConnections permissions
     ├── CodePipeline permissions
     └── Elastic Beanstalk permissions
```

A practical issue occurred during pipeline creation: the automatically generated role did not contain the required Elastic Beanstalk permissions.

The deployment stage therefore required an IAM permission correction before the pipeline could complete successfully.

This demonstrates an important architectural boundary:

> **The pipeline's ability to orchestrate a service depends on the IAM permissions granted to its service role.**

---

## 14. Bitbucket Connection

The Bitbucket connection is established through AWS CodeConnections.

The relationship is:

```text
Bitbucket
    │
    │ CodeConnections
    ▼
CodePipeline
```

CodePipeline uses this connection to access the configured Bitbucket repository and branch.

The pipeline itself does not replace Git or Bitbucket.

Instead:

```text
Bitbucket = source management
CodePipeline = delivery orchestration
```

---

## 15. Complete Control Flow

The complete CI/CD control flow is:

```text
Developer
    │
    │ git push
    ▼
Bitbucket
    │
    │ commit event
    ▼
CodePipeline
    │
    │ source stage
    ▼
Source Revision
    │
    │ build stage
    ▼
CodeBuild
    │
    │ buildspec.yml
    │ Maven
    ▼
Deployable Artifact
    │
    │ deploy stage
    ▼
Elastic Beanstalk
    │
    ▼
VProfile Application
```

This is the primary **delivery/control path**.

---

## 16. Complete Runtime Flow

The application runtime path is separate:

```text
Elastic Beanstalk
       │
       ▼
VProfile Application
       │
       │ database access
       ▼
Amazon RDS
```

Therefore:

```text
CI/CD FLOW
Bitbucket → CodePipeline → CodeBuild → Beanstalk

RUNTIME FLOW
Beanstalk → VProfile → RDS
```

Keeping these two flows separate is one of the most important architectural concepts in this project.

---

## 17. Combined Architecture

Putting both flows together:

```text
                         CI/CD CONTROL FLOW

Developer
    │
    │ git push
    ▼
┌──────────────┐
│   Bitbucket  │
│   aws-ci     │
└──────┬───────┘
       │
       │ commit event
       ▼
┌──────────────────┐
│  CodePipeline    │
│  Orchestrator    │
└──────┬───────────┘
       │
       │ trigger
       ▼
┌──────────────────┐
│    CodeBuild     │
│                  │
│ buildspec.yml    │
│ Maven            │
└──────┬───────────┘
       │
       │ artifact
       ▼
┌──────────────────────┐
│ Elastic Beanstalk    │
│                      │
│ ┌──────────────────┐ │
│ │ Application      │ │
│ │ Instance 1       │ │
│ └──────────────────┘ │
│ ┌──────────────────┐ │
│ │ Application      │ │
│ │ Instance 2       │ │
│ └──────────────────┘ │
└──────────┬───────────┘
           │
           │ runtime
           ▼
    ┌───────────────┐
    │ VProfile      │
    │ Application   │
    └───────┬───────┘
            │
            │ database access
            ▼
       ┌───────────┐
       │    RDS    │
       │ Database  │
       └───────────┘
```

---

## 18. Architectural Responsibilities

| Component | Architectural Responsibility |
|---|---|
| Bitbucket | Stores source code and provides the source-change event |
| CodeConnections | Connects AWS CodePipeline to Bitbucket |
| CodePipeline | Orchestrates source, build, and deployment stages |
| CodeBuild | Builds the application and produces the deployable artifact |
| `buildspec.yml` | Defines CodeBuild execution behavior |
| Maven | Builds the Java application |
| Artifact | Carries the built application between build and deployment |
| Elastic Beanstalk | Deploys and runs the application |
| EC2 instances | Host the deployed application within Beanstalk |
| RDS | Provides the application's relational database |
| IAM | Controls service permissions required for the workflow |
| Security Groups | Control network access between application and database resources |

---

## 19. Architectural Decisions

### Bitbucket for Source, AWS for CI/CD

The project deliberately uses Bitbucket as the Git source repository while AWS services handle CI/CD.

```text
Bitbucket
    ↓
Source only

AWS
    ↓
Build + orchestration + deployment
```

This represents a mixed-platform delivery pattern: one platform provides source management while another provides CI/CD.

### CodeBuild Separate from CodePipeline

CodeBuild and CodePipeline have different responsibilities:

```text
CodeBuild
= Build

CodePipeline
= Orchestration
```

This separation keeps the pipeline orchestrator responsible for workflow coordination rather than the actual application build.

### RDS Outside the CI/CD Pipeline

RDS is modeled as a runtime dependency rather than a pipeline stage.

```text
Pipeline
  ↓
Deploy application

Application
  ↓
Use database
```

### Managed AWS Services

The architecture uses managed AWS services for build, orchestration, deployment, and database capabilities.

The conceptual mapping is:

| Earlier Jenkins Model | AWS-Native Model |
|---|---|
| GitHub | Bitbucket |
| Jenkins build | CodeBuild |
| Jenkins pipeline | CodePipeline |
| Application runtime | Elastic Beanstalk |
| MySQL database | RDS |

The important architectural change is the shift toward managed AWS services while preserving the same fundamental source → build → deploy model.

---

## 20. Dependency-Driven Architecture

The project follows a dependency-aware implementation sequence.

The major dependencies are:

```text
Source Repository
       ↓
CodeBuild
       ↓
CodePipeline
       ↓
Elastic Beanstalk
       ↓
Application
       ↓
RDS
```

Each component provides something required by the next part of the workflow:

- CodeBuild requires source code.
- CodePipeline requires a configured source and build provider.
- Deployment requires a prepared Elastic Beanstalk environment.
- The running application requires RDS.
- IAM permissions are required for the pipeline to interact with its integrated services.

This dependency model also explains why cleanup must account for resource relationships.

---

## 21. Architecture Limitations

This architecture intentionally demonstrates the core source → build → deploy workflow rather than a complete production CI/CD platform.

The demonstrated pipeline contains:

```text
Source
  ↓
Build
  ↓
Deploy
```

The test stage was optional and was skipped in the demonstrated implementation.

The project therefore does not establish:

- a comprehensive automated testing stage
- multi-environment promotion
- staging-to-production approval gates
- Infrastructure as Code
- multi-region deployment
- enterprise disaster recovery
- a complete production security model

These are future architectural extensions rather than implemented components.

---

## 22. Architecture Summary

The architecture can be reduced to five core responsibilities:

```text
BITBUCKET
   ↓
SOURCE

CODEPIPELINE
   ↓
ORCHESTRATION

CODEBUILD
   ↓
BUILD

ELASTIC BEANSTALK
   ↓
DEPLOY + RUNTIME

RDS
   ↓
DATABASE
```

The most important distinction is:

```text
CodePipeline = controls the delivery flow

CodeBuild = performs the build

Elastic Beanstalk = runs the deployed application

RDS = supports the application's runtime
```

This separation of responsibilities is the core architectural model demonstrated by the project.

---

[← Back to README](../README.md)
