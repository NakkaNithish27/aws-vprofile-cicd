# Validation

[← Back to README](../README.md)

## 1. Validation Overview

Validation for this project is designed to verify the complete delivery chain rather than only individual AWS services.

The validation model follows:

```text
Source
   ↓
Pipeline Trigger
   ↓
Build
   ↓
Artifact
   ↓
Deployment
   ↓
Application Runtime
   ↓
Database Connectivity
```

The final validation also verifies that a new commit to the configured Bitbucket branch automatically starts a new pipeline execution.

---

## 2. Validation Strategy

Validation is performed at multiple levels:

```text
Level 1
Source validation
        ↓
Level 2
Build validation
        ↓
Level 3
Pipeline validation
        ↓
Level 4
Deployment validation
        ↓
Level 5
Application validation
        ↓
Level 6
Event-driven validation
```

Each level answers a different question.

| Validation Level | Question |
|---|---|
| Source | Can the pipeline obtain the intended source revision? |
| Build | Can CodeBuild successfully produce the application artifact? |
| Pipeline | Can CodePipeline orchestrate the complete workflow? |
| Deployment | Can the artifact be deployed to Elastic Beanstalk? |
| Application | Does the deployed application become usable? |
| Database | Can the application communicate with RDS? |
| Event Trigger | Does a source commit automatically initiate a new delivery cycle? |

---

# 3. Source Validation

## Objective

Verify that Bitbucket is correctly configured as the source repository and that the pipeline is monitoring the intended branch.

The configured branch is:

```text
aws-ci
```

The expected relationship is:

```text
Bitbucket
    │
    │ aws-ci
    ▼
CodePipeline
```

The VProfile source was migrated from GitHub to Bitbucket and CodePipeline monitors the Bitbucket repository for new commits.

### Validation

Confirm that:

- the repository exists in Bitbucket
- the expected source is present
- the `aws-ci` branch exists
- CodePipeline is configured against the intended repository
- CodePipeline can retrieve the source revision

### Expected result

```text
Source stage
     ↓
SUCCESS
```

---

# 4. SSH Authentication Validation

SSH authentication was configured for Git operations against Bitbucket.

The authentication path is:

```text
Git
 │
 ▼
SSH
 │
 ├── ~/.ssh/config
 ├── private key
 └── known_hosts
 │
 ▼
Bitbucket
```

A connection test can be performed with:

```bash
ssh -T git@bitbucket.org
```

The purpose is to verify that the configured SSH identity can authenticate with Bitbucket.

### Expected result

The Bitbucket connection should authenticate successfully without requiring a Git username/password workflow.

### Security boundary

Private keys must never be placed in the public repository.

Only sanitized configuration patterns should be documented.

---

# 5. CodeBuild Validation

## Objective

Verify that CodeBuild can:

1. obtain the source
2. initialize the build environment
3. execute the buildspec
4. configure the application
5. run Maven
6. generate the deployable artifact

The CodeBuild configuration uses `buildspec.yml` as the central build definition.

The build flow is:

```text
Source
   ↓
CodeBuild
   ↓
buildspec.yml
   ↓
Install
   ↓
Pre-build
   ↓
Build
   ↓
Post-build
   ↓
Artifact
```

---

# 6. Build Phase Validation

Each build phase provides a separate validation point.

### Install

Verify:

```text
Java runtime available
Required build tools available
```

### Pre-build

Verify:

```text
Dependencies installed
Maven available
Application configuration prepared
```

### Build

Verify:

```text
Maven build completes successfully
```

### Post-build

Verify:

```text
Post-build commands complete
```

### Artifacts

Verify:

```text
Expected application artifact exists
```

The expected output is a WAR-based deployment artifact.

---

# 7. CodeBuild Failure Validation

The project included a real build failure during the practical.

The failure occurred in the pre-build phase:

```text
sed -e expression, unterminated s command
```

The troubleshooting sequence was:

```text
Build
 ↓
Pre-build failure
 ↓
Inspect phase details
 ↓
Inspect build output
 ↓
Identify malformed sed expression
 ↓
Correct buildspec
 ↓
Re-run
 ↓
SUCCESS
```

The root cause was a malformed `sed` substitution.

The correct pattern is:

```text
sed s/SEARCH/REPLACE/
```

rather than:

```text
sed s/SEARCH/REPLACE
```

### What this validation proves

It demonstrates the ability to:

- identify the failing build phase
- inspect build output
- interpret the error
- trace the failure to configuration
- correct the configuration
- rerun the build

This is stronger engineering evidence than simply recording a successful build.

---

# 8. CodeBuild Artifact Validation

After a successful build, verify that the expected application artifact is produced.

The artifact flow is:

```text
Maven
  ↓
target/
  ↓
WAR artifact
  ↓
CodePipeline
```

### Expected result

```text
BUILD
  ↓
SUCCESS

ARTIFACT
  ↓
AVAILABLE
```

A successful artifact is a prerequisite for the deployment stage.

---

# 9. CodePipeline Validation

## Objective

Verify that CodePipeline successfully connects the individual components.

The expected pipeline is:

```text
SOURCE
  ↓
BUILD
  ↓
DEPLOY
```

CodePipeline acts as the orchestration layer rather than performing the Maven build itself.

### Validation checklist

| Stage | Expected Result |
|---|---|
| Source | Successful |
| Build | Successful |
| Deploy | Successful |

The optional test stage was skipped in the demonstrated implementation.

---

# 10. IAM Validation

The pipeline initially encountered a deployment permission problem.

The automatically created CodePipeline service role did not initially contain the required Elastic Beanstalk permissions.

The validation process therefore included:

```text
Pipeline
   ↓
Deploy stage
   ↓
Permission failure
   ↓
Inspect service role
   ↓
Identify missing permission
   ↓
Update IAM role
   ↓
Retry pipeline
   ↓
Deploy succeeds
```

### What this validates

The final successful deployment demonstrates that the pipeline service role has sufficient permissions for the configured deployment workflow.

It does **not** establish that the IAM configuration is production-grade or least-privilege.

---

# 11. Elastic Beanstalk Deployment Validation

## Objective

Verify that the artifact produced by CodeBuild can be successfully deployed to the configured Elastic Beanstalk environment.

The deployment path is:

```text
CodePipeline
      ↓
Artifact
      ↓
Elastic Beanstalk
      ↓
Application Environment
```

### Expected result

The Elastic Beanstalk deployment should complete successfully and the environment should become available for application access.

The project environment uses two application instances with a 50% rolling deployment configuration.

---

# 12. Application Validation

The application itself provides the next validation layer.

The expected flow is:

```text
Beanstalk Environment
        ↓
Application URL
        ↓
VProfile Login Page
        ↓
Successful Login
```

A successful application login is used as the application-level validation signal.

### What successful login demonstrates

A successful login provides evidence that:

- the application artifact was built
- the artifact was deployed
- the application started
- the application configuration was usable
- the application could reach its required backend services
- the application could communicate with RDS

It is therefore a high-value end-to-end validation rather than merely a UI check.

---

# 13. RDS Connectivity Validation

RDS is outside the CI/CD pipeline but is a runtime dependency of the deployed application.

The runtime relationship is:

```text
Elastic Beanstalk
       ↓
VProfile Application
       ↓
Amazon RDS
```

The application-level validation therefore indirectly validates the required database connection.

The complete runtime path is:

```text
Application
    ↓
Database configuration
    ↓
RDS endpoint
    ↓
Database
```

### Expected result

The application should successfully perform the operations requiring its database.

A successful application login is used by the practical as the final indication that the application can communicate with RDS.

---

# 14. Event-Driven Trigger Validation

This is the most important final validation.

## Objective

Verify that a source change automatically initiates the delivery workflow.

The expected flow is:

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
    ▼
CodeBuild
    │
    ▼
Elastic Beanstalk
```

---

## 15. Event Trigger Test

A small change can be committed to the repository, such as an update to `README.md`.

The validation sequence is:

```text
1. Modify repository
        ↓
2. git add
        ↓
3. git commit
        ↓
4. git push to aws-ci
        ↓
5. Bitbucket detects new commit
        ↓
6. CodePipeline starts
        ↓
7. Source stage succeeds
        ↓
8. CodeBuild runs
        ↓
9. Artifact generated
        ↓
10. Elastic Beanstalk deployment
        ↓
11. Application updated
```

No manual pipeline execution should be required after the event trigger is configured.

---

# 16. End-to-End Acceptance Test

The complete project passes its primary acceptance test when the following chain succeeds:

```text
┌──────────────────────────────┐
│ Bitbucket commit             │
└──────────────┬───────────────┘
               ↓
┌──────────────────────────────┐
│ CodePipeline automatically   │
│ starts                       │
└──────────────┬───────────────┘
               ↓
┌──────────────────────────────┐
│ CodeBuild succeeds           │
└──────────────┬───────────────┘
               ↓
┌──────────────────────────────┐
│ Deployable artifact produced │
└──────────────┬───────────────┘
               ↓
┌──────────────────────────────┐
│ Elastic Beanstalk deployment │
│ succeeds                     │
└──────────────┬───────────────┘
               ↓
┌──────────────────────────────┐
│ Application becomes          │
│ accessible                   │
└──────────────┬───────────────┘
               ↓
┌──────────────────────────────┐
│ Application can use RDS      │
└──────────────────────────────┘
```

This validates the project as an integrated CI/CD workflow rather than a collection of independently configured AWS services.

---

# 17. Validation Matrix

| Validation | Expected Result | What It Proves | Evidence |
|---|---|---|---|
| Bitbucket repository | Repository accessible | Source configured | Own environment screenshot if retained |
| `aws-ci` branch | Branch available | Correct pipeline source | Own environment screenshot if retained |
| SSH test | Authentication succeeds | Git SSH configuration works | Own environment output if retained |
| CodeBuild | Build succeeds | Build configuration works | CodeBuild execution |
| Buildspec | Phases complete | Build instructions work | Build logs |
| Artifact | WAR produced | Build output available | Build/artifact evidence |
| CodePipeline | Pipeline succeeds | Source/build/deploy integration works | Pipeline execution |
| IAM | Deploy stage succeeds | Required service permissions exist | Sanitized IAM evidence |
| Beanstalk | Deployment succeeds | Artifact can be deployed | Deployment evidence |
| Application | Login succeeds | Application is operational | Own application screenshot |
| RDS | Application works with DB | Runtime DB connectivity | Application validation |
| Commit trigger | Pipeline starts automatically | Event-driven delivery works | Commit + pipeline execution |

---

# 18. Evidence Strategy

The repository should contain only **high-signal evidence from the personally completed environment**.

Evidence should not consist of screenshots copied from the course material.

The strongest evidence items are:

### Evidence 1 — Successful CodeBuild

```text
CodeBuild
   ↓
BUILD SUCCEEDED
```

**Supports:**

> CodeBuild was configured successfully.

---

### Evidence 2 — Build Troubleshooting

```text
Failed pre-build
   ↓
sed error
   ↓
Corrected build
   ↓
Success
```

**Supports:**

> A real build failure was diagnosed and corrected.

---

### Evidence 3 — Successful CodePipeline

```text
Source ✓
Build ✓
Deploy ✓
```

**Supports:**

> The complete pipeline executed successfully.

---

### Evidence 4 — Elastic Beanstalk Deployment

```text
Deployment
     ↓
Successful
```

**Supports:**

> The build artifact reached the deployment target.

---

### Evidence 5 — Application Validation

```text
VProfile
   ↓
Login
   ↓
Success
```

**Supports:**

> The deployed application was operational.

---

### Evidence 6 — Automatic Trigger

```text
Bitbucket commit
       ↓
CodePipeline execution
```

**Supports:**

> The pipeline is event-driven.

---

# 19. Evidence Mapping

The evidence chain should look like:

```text
README Claim
      ↓
Architecture / Implementation
      ↓
Validation
      ↓
Own Environment Evidence
```

For example:

```text
Claim:
"Configured event-driven CI/CD"

        ↓

Architecture:
Bitbucket → CodePipeline → CodeBuild → Beanstalk

        ↓

Validation:
Push commit to aws-ci

        ↓

Evidence:
Pipeline automatically starts
```

Another example:

```text
Claim:
"Troubleshot CodeBuild failure"

        ↓

Implementation:
Malformed sed expression

        ↓

Validation:
Build failed → correction → successful rerun

        ↓

Evidence:
Own CodeBuild execution/logs
```

---

# 20. What Validation Does Not Prove

Successful validation of this project does **not** prove:

- production readiness
- enterprise scalability
- comprehensive automated testing
- Infrastructure as Code
- multi-region resilience
- disaster recovery
- production-grade secret management
- least-privilege IAM across the entire environment
- security compliance
- performance under production load

The validation proves that the demonstrated delivery workflow can successfully move a source change through the configured source, build, deployment, and runtime path.

---

# 21. Validation Boundaries

The project has three important validation boundaries.

### Boundary 1 — Build

```text
Source → CodeBuild → Artifact
```

This proves the application can be built.

It does not prove deployment works.

### Boundary 2 — Deployment

```text
Artifact → Elastic Beanstalk
```

This proves the artifact can be deployed.

It does not by itself prove the application is functioning correctly.

### Boundary 3 — Runtime

```text
Application → RDS
```

This proves the deployed application can operate against its database dependency.

The complete project requires all three boundaries to succeed.

---

# 22. Final Validation Model

The project's validation can therefore be compressed into:

```text
SOURCE
Bitbucket
   │
   │ commit
   ▼
TRIGGER
CodePipeline
   │
   ▼
BUILD
CodeBuild
   │
   ▼
ARTIFACT
WAR
   │
   ▼
DEPLOY
Elastic Beanstalk
   │
   ▼
RUNTIME
VProfile
   │
   ▼
DATABASE
RDS
```

The final acceptance condition is:

> **A source change pushed to the configured Bitbucket branch automatically triggers the pipeline, produces a successful build artifact, deploys the application to Elastic Beanstalk, and results in a functioning application that can use its RDS backend.**

This is the strongest validation statement supported by the project material.

---

[← Back to README](../README.md)
