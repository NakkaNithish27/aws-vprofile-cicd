# Implementation

[← Back to README](../README.md)

## 1. Implementation Overview

This project was implemented as a dependency-driven AWS CI/CD workflow around the existing VProfile application workload.

The implementation sequence followed the dependency relationships between the components:

```text
Elastic Beanstalk
       ↓
Amazon RDS
       ↓
Bitbucket
       ↓
CodeBuild
       ↓
CodePipeline
       ↓
End-to-End Validation
```

The implementation can be grouped into five engineering areas:

1. Application runtime preparation
2. Source-control migration and authentication
3. Automated build configuration
4. CI/CD pipeline orchestration
5. End-to-end validation and troubleshooting

---

## 2. Application Runtime Preparation

### 2.1 Elastic Beanstalk Environment

Elastic Beanstalk was prepared as the deployment target before configuring the CI/CD pipeline.

The reason for doing this first was straightforward:

```text
CodePipeline
     │
     │ deploys to
     ▼
Elastic Beanstalk
```

The deployment target therefore needed to exist before CodePipeline could reference it.

The VProfile application was treated as an existing workload rather than application code developed as part of this project.

---

### 2.2 Amazon RDS

Amazon RDS was configured as the database dependency for the application.

The application expects a database named:

```text
accounts
```

The database instance was therefore provisioned with the required initial database name.

The runtime relationship is:

```text
Elastic Beanstalk
       │
       ▼
VProfile Application
       │
       ▼
Amazon RDS
       │
       └── accounts database
```

---

### 2.3 Database Credentials

The practical used an auto-generated RDS master password.

The important operational consideration was to capture the generated credentials when they were presented because the generated password is not subsequently displayed in the same way.

For this repository:

> **No database password, credential, token, or secret should be committed.**

Any documentation or configuration examples must use placeholders such as:

```text
<RDS_USERNAME>
<RDS_PASSWORD>
<RDS_ENDPOINT>
```

The source material highlights the security risk of exposing database credentials in command-line arguments or other visible locations.

---

### 2.4 Security-Group Connectivity

The application environment and RDS instance were connected through security-group rules.

The important relationship is:

```text
Beanstalk Security Group
          │
          │ allowed database access
          ▼
RDS Security Group
```

The RDS security group references the Beanstalk security group so that application instances can communicate with the database.

This creates a resource dependency that later affects cleanup as well.

---

## 3. Bitbucket Repository Setup

### 3.1 Source Migration

The VProfile source used in the earlier Jenkins workflow was migrated from GitHub to Bitbucket.

The migration preserves the Git repository rather than creating a new unrelated source history.

The conceptual migration flow was:

```text
Existing GitHub Repository
          │
          ▼
        Clone
          │
          ▼
Checkout branches
          │
          ▼
Fetch tags
          │
          ▼
Remove old origin
          │
          ▼
Add Bitbucket origin
          │
          ▼
Push branches
          │
          ▼
Push tags
```

The migration pattern preserves repository history, branches, and tags.

---

### 3.2 Bitbucket Branch

The pipeline uses:

```text
aws-ci
```

as the source branch.

The final source flow is therefore:

```text
Developer
    │
    │ git push
    ▼
Bitbucket
    │
    │ aws-ci
    ▼
CodePipeline
```

The Bitbucket repository is used for **source control only** in this project. AWS services provide the CI/CD functionality.

---

## 4. SSH Authentication

### 4.1 SSH Key Pair

SSH authentication was configured for Git operations against Bitbucket.

SSH keys were generated for the Bitbucket authentication workflow.

The important distinction is:

```text
EC2 SSH key
    ≠
Git/Bitbucket SSH authentication
```

The same SSH key-generation mechanism is used, but the purpose here is authenticating Git operations with Bitbucket.

---

### 4.2 SSH Config

The local SSH configuration maps Bitbucket to the appropriate private key.

Conceptually:

```text
~/.ssh/config

bitbucket.org
      │
      └── IdentityFile → Bitbucket private key
```

This allows Git to automatically select the appropriate identity when connecting to Bitbucket.

The SSH configuration is a general-purpose routing mechanism and can support different identities for different hosts.

---

### 4.3 Known Hosts

SSH host verification was handled through:

```text
~/.ssh/known_hosts
```

An explicit connection test using:

```bash
ssh -T git@bitbucket.org
```

can establish the Bitbucket host fingerprint and simultaneously verify that the SSH authentication configuration is functioning.

The important outcome is that Git can authenticate to Bitbucket without relying on a username/password workflow.

---

## 5. CodeBuild Configuration

### 5.1 Artifact Storage

An S3 bucket was prepared for build artifacts.

The bucket needs to be in the same AWS region as the CodeBuild project.

The dependency is:

```text
S3 Artifact Bucket
        ↓
CodeBuild Project
        ↓
Artifact Output
```

The bucket was created before the CodeBuild project because the bucket is selected as part of the CodeBuild artifact configuration.

---

### 5.2 CodeBuild Project

The CodeBuild project was configured as the managed build environment.

The project used:

```text
Source:
Bitbucket

Branch:
aws-ci
```

The Bitbucket connection was established through AWS CodeConnections so that CodeBuild could access the repository.

Conceptually:

```text
Bitbucket
    │
    ▼
CodeBuild
    │
    ▼
Build Artifact
```

CodeBuild provides ephemeral build compute rather than requiring a permanently running Jenkins server.

---

### 5.3 Buildspec

The central CodeBuild configuration is:

```text
buildspec.yml
```

It defines:

- runtime configuration
- pre-build commands
- build commands
- post-build commands
- artifact configuration

The buildspec uses version:

```yaml
version: 0.2
```

The buildspec acts as the build contract for CodeBuild.

---

### 5.4 Build Phases

The build follows the CodeBuild phase structure:

```text
install
   ↓
pre_build
   ↓
build
   ↓
post_build
   ↓
artifacts
```

#### Install

The build environment uses:

```text
Java Corretto 17
```

as the configured Java runtime.

#### Pre-build

The pre-build phase prepares the build environment and application configuration.

The practical includes:

```text
apt-get update
apt-get install jq
download Maven
extract Maven
create Maven symlink
update application.properties
```

The Maven installation follows:

```text
wget
  ↓
tar xzf
  ↓
ln -s
  ↓
mvn
```

The source material specifies Maven 3.9.8 for this build process.

---

### 5.5 Environment-Specific Configuration

The build process modifies:

```text
src/main/resources/application.properties
```

to replace development/default database values with the target RDS configuration.

Conceptually:

```text
Default Configuration
        │
        │ sed
        ▼
Target Environment Configuration
```

The substitutions cover:

```text
jdbc.password
jdbc.username
database endpoint
```

The pattern is:

```bash
sed -i 's/OLD_VALUE/NEW_VALUE/' application.properties
```

The important engineering concept is that environment-specific configuration is applied during the build rather than requiring production values to remain hardcoded in the source repository.

### Repository safety

Actual credentials must never be committed.

Use placeholders in public examples:

```text
<RDS_USERNAME>
<RDS_PASSWORD>
<RDS_ENDPOINT>
```

---

### 5.6 Maven Build

The primary build command is:

```bash
mvn install
```

This compiles the application, executes the Maven lifecycle, and produces the deployable application artifact.

The post-build configuration also invokes:

```bash
mvn package
```

The source material notes that `mvn install` already performs packaging, making the additional `mvn package` redundant for this particular project.

The repository should document this as an implementation characteristic rather than presenting the duplicate command as a requirement.

---

### 5.7 Artifact Configuration

The resulting application artifact is a WAR-based deployment artifact.

The build output is collected from:

```text
target/vprofile-v2
```

and passed to the artifact/deployment flow.

Conceptually:

```text
Source
  ↓
Maven
  ↓
target/
  ↓
WAR artifact
  ↓
CodePipeline
  ↓
Elastic Beanstalk
```

---

### 5.8 CloudWatch Build Logs

CodeBuild uses ephemeral compute.

After the build finishes, the underlying build environment is not retained as a persistent server.

Therefore, build output needs a persistent logging destination.

CloudWatch Logs provides that persistence:

```text
CodeBuild
    │
    │ build output
    ▼
CloudWatch Logs
    │
    ├── Log Group
    └── Log Stream
```

This allows build output to be inspected after the build environment has disappeared.

---

## 6. CodeBuild Troubleshooting

### 6.1 Failed Build

The CodeBuild build initially failed during the pre-build phase.

The relevant error was:

```text
sed -e expression, unterminated s command
```

The problem was a malformed `sed` substitution.

A substitution requires the delimiter structure:

```text
sed s/SEARCH/REPLACE/
```

The failing form was missing the final delimiter:

```text
sed s/SEARCH/REPLACE
```

The troubleshooting sequence was:

```text
Build started
    ↓
Pre-build failure
    ↓
Inspect phase details
    ↓
Inspect tail of build logs
    ↓
Identify malformed sed command
    ↓
Correct buildspec
    ↓
Run build again
    ↓
Build succeeds
```

The troubleshooting approach was to identify the failed phase, inspect the relevant build output, map the error to the buildspec, correct the configuration, and rerun the build.

---

### 6.2 Why the Failure Was Useful

The failure demonstrated a practical CI/CD troubleshooting pattern:

```text
Pipeline failure
      ↓
Identify failed stage
      ↓
Identify failed phase
      ↓
Read actual error
      ↓
Map error to configuration
      ↓
Correct configuration
      ↓
Re-run
```

The repository should preserve this as an engineering troubleshooting example rather than merely recording that the build eventually succeeded.

---

## 7. CodePipeline Configuration

### 7.1 Pipeline Type

CodePipeline was created as a **custom pipeline** because the required combination was:

```text
Bitbucket
   +
CodeBuild
   +
Elastic Beanstalk
```

The custom pipeline option provides control over each stage.

---

### 7.2 Pipeline Stages

The resulting pipeline contains the core stages:

```text
SOURCE
   ↓
BUILD
   ↓
DEPLOY
```

The optional test stage was skipped in the demonstrated implementation.

---

### 7.3 Source Stage

The source stage connects:

```text
Bitbucket
     │
     │ CodeConnections
     ▼
CodePipeline
```

Configuration includes:

```text
Provider: Bitbucket
Repository: VProfile repository
Branch: aws-ci
```

CodePipeline monitors the configured branch for changes.

---

### 7.4 Build Stage

The build stage connects CodePipeline to the previously configured CodeBuild project:

```text
CodePipeline
     │
     ▼
CodeBuild
     │
     ▼
Build Artifact
```

CodePipeline does not perform the Maven build itself.

Instead:

```text
CodePipeline = orchestration
CodeBuild    = build execution
```

This separation is central to the architecture.

---

### 7.5 Deploy Stage

The deployment stage connects CodePipeline to Elastic Beanstalk:

```text
CodePipeline
     │
     │ artifact
     ▼
Elastic Beanstalk
     │
     ▼
Application
```

The Beanstalk environment uses two application instances and a 50% rolling deployment batch size.

---

## 8. IAM Troubleshooting

### 8.1 Service Role

When the CodePipeline was created, AWS automatically created an IAM service role.

The role needs permissions to interact with the services used by the pipeline.

Conceptually:

```text
CodePipeline Service Role
        │
        ├── CodeConnections
        ├── CodeBuild
        └── Elastic Beanstalk
```

---

### 8.2 Deployment Permission Failure

The automatically generated role did not initially contain the required Elastic Beanstalk permissions.

The pipeline therefore encountered a deployment-stage permission problem.

The troubleshooting flow was:

```text
Pipeline created
      ↓
Source stage
      ↓
Build stage
      ↓
Deploy stage fails
      ↓
Inspect IAM service role
      ↓
Identify missing Beanstalk permissions
      ↓
Update role permissions
      ↓
Release pipeline change
      ↓
Pipeline succeeds
```

The deployment problem was resolved by updating the pipeline service role with the required Elastic Beanstalk permissions.

---

### 8.3 IAM Lesson

The important engineering lesson is:

> Creating a pipeline does not automatically guarantee that its service role has every permission required by every integrated service.

The complete workflow must be considered:

```text
Pipeline
   │
   ├── Source → required permissions
   ├── Build  → required permissions
   └── Deploy → required permissions
```

IAM troubleshooting is therefore part of integrating the pipeline, not an unrelated AWS administration task.

---

## 9. End-to-End Pipeline Execution

After the individual components were configured, the pipeline was executed as a complete workflow.

The final flow was:

```text
Bitbucket commit
       ↓
CodePipeline detects change
       ↓
Source stage
       ↓
CodeBuild
       ↓
Maven build
       ↓
WAR artifact
       ↓
Elastic Beanstalk deployment
       ↓
Application startup
       ↓
RDS connectivity
```

The project therefore moved from component-level configuration to complete integration.

---

## 10. Deployment Validation

The deployed Beanstalk environment was accessed through its environment URL.

The validation sequence was:

```text
Beanstalk URL
      ↓
Load balancer
      ↓
VProfile login page
      ↓
Successful login
```

A successful application login was used as the end-to-end validation signal.

It demonstrates that:

1. The artifact was successfully built.
2. The artifact reached Elastic Beanstalk.
3. The application started successfully.
4. The generated application configuration was usable.
5. The application could connect to RDS.

---

## 11. Event-Driven Deployment Validation

The final validation introduced a new source change.

A small change to the repository was committed and pushed to:

```text
aws-ci
```

The expected flow was:

```text
git push
   ↓
Bitbucket
   ↓
CodePipeline detects commit
   ↓
Pipeline starts automatically
   ↓
CodeBuild
   ↓
Elastic Beanstalk
```

No manual pipeline execution was required after the event trigger was configured.

This validates that the project is an event-driven CI/CD workflow rather than only a manually connected source/build/deployment environment.

---

## 12. Validation of the Complete Chain

The final validation can be represented as:

```text
┌─────────────────────────────┐
│ 1. Source Change            │
│    Bitbucket commit         │
└──────────────┬──────────────┘
               ↓
┌─────────────────────────────┐
│ 2. Pipeline Trigger         │
│    CodePipeline             │
└──────────────┬──────────────┘
               ↓
┌─────────────────────────────┐
│ 3. Build                    │
│    CodeBuild + Maven        │
└──────────────┬──────────────┘
               ↓
┌─────────────────────────────┐
│ 4. Artifact                 │
│    WAR output               │
└──────────────┬──────────────┘
               ↓
┌─────────────────────────────┐
│ 5. Deployment               │
│    Elastic Beanstalk        │
└──────────────┬──────────────┘
               ↓
┌─────────────────────────────┐
│ 6. Runtime                  │
│    VProfile + RDS           │
└─────────────────────────────┘
```

Each stage provides evidence for the next stage.

---

## 13. Implementation Sequence

The complete implementation can be reconstructed as:

### Phase 1 — Prepare Runtime

```text
Create Elastic Beanstalk
        ↓
Create RDS
        ↓
Configure application/database connectivity
```

### Phase 2 — Prepare Source

```text
GitHub repository
        ↓
Clone
        ↓
Preserve branches/tags
        ↓
Configure Bitbucket SSH
        ↓
Push repository to Bitbucket
```

### Phase 3 — Prepare Build

```text
Create S3 artifact bucket
        ↓
Create CodeBuild project
        ↓
Connect Bitbucket
        ↓
Configure build environment
        ↓
Configure buildspec
        ↓
Configure artifacts
        ↓
Configure CloudWatch logs
```

### Phase 4 — Prepare Pipeline

```text
Create CodePipeline
        ↓
Configure Bitbucket source
        ↓
Configure CodeBuild build
        ↓
Configure Beanstalk deployment
        ↓
Configure IAM permissions
```

### Phase 5 — Validate

```text
Run pipeline
    ↓
Diagnose failures
    ↓
Correct configuration
    ↓
Run again
    ↓
Validate application
    ↓
Push new commit
    ↓
Validate automatic trigger
```

---

## 14. Cleanup and Resource Dependencies

Cleanup must account for AWS resource dependencies.

A critical dependency exists between the RDS security group and the Beanstalk security group:

```text
RDS Security Group
       │
       │ references
       ▼
Beanstalk Security Group
```

If the reference remains, Beanstalk deletion can fail because its security group is still referenced.

The dependency-aware cleanup therefore requires removing the security-group reference before deleting the Beanstalk environment.

The practical cleanup sequence is:

```text
1. Remove database/security-group dependency
2. Delete Beanstalk application/environment
3. Delete CodePipeline
4. Delete CodeBuild if desired
5. Retain Bitbucket if useful for future recreation
```

The exact cleanup order should always be based on the actual resource relationships present in the environment.

---

## 15. Implementation Boundaries

This implementation document describes the engineering work performed around the application workload.

It should not be interpreted as claiming that the project author:

- developed VProfile
- authored its business logic
- designed its original application architecture
- implemented its authentication system
- created the original application source

The project contribution is the surrounding delivery system:

```text
Source Control
      ↓
Build Automation
      ↓
CI/CD Orchestration
      ↓
Deployment
      ↓
Runtime Integration
      ↓
Validation
```

---

## 16. Implementation Summary

The implementation established the following working chain:

```text
Bitbucket
    │
    │ aws-ci commit
    ▼
CodePipeline
    │
    │ source
    ▼
CodeBuild
    │
    ├── buildspec.yml
    ├── Maven
    ├── configuration injection
    └── WAR artifact
    │
    ▼
Elastic Beanstalk
    │
    ▼
VProfile Application
    │
    ▼
Amazon RDS
```

The most important implementation lessons were:

- Build infrastructure in dependency order.
- Keep source management separate from CI/CD orchestration.
- Use CodeBuild for build execution and CodePipeline for orchestration.
- Treat `buildspec.yml` as the build contract.
- Keep environment-specific values out of source-controlled code.
- Preserve build logs because CodeBuild uses ephemeral compute.
- Diagnose failures from the actual failing phase and logs.
- Verify IAM permissions across the complete pipeline.
- Validate the entire chain rather than assuming that a successful build means a successful deployment.
- Validate event-driven behavior with a real source commit.
- Respect AWS resource dependencies during cleanup.

These implementation patterns form the practical foundation of the AWS-native CI/CD project.

---

[← Back to README](../README.md)
