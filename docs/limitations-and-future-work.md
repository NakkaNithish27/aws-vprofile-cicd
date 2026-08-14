# Limitations and Future Work

[← Back to README](../README.md)

## 1. Overview

This project demonstrates a working AWS-native CI/CD workflow around the existing VProfile application workload.

The implemented delivery path is:

```text
Bitbucket
    ↓
CodePipeline
    ↓
CodeBuild
    ↓
Elastic Beanstalk
    ↓
VProfile Application
    ↓
Amazon RDS
```

The project successfully demonstrates source integration, automated build, artifact generation, deployment orchestration, runtime validation, troubleshooting, and event-driven delivery.

However, the implementation is intentionally limited to the core CI/CD workflow demonstrated in the section. It should not be interpreted as a complete enterprise production platform.

---

# 2. Application Ownership Limitation

The VProfile application is an existing application workload used for the project.

Therefore, this project does not claim ownership of:

- VProfile application development
- VProfile business logic
- the original application architecture
- the original authentication implementation
- the original application source code

The primary engineering contribution is the DevOps environment surrounding the application:

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

Future work should continue to preserve this distinction between the application workload and the DevOps engineering implemented around it.

---

# 3. CI/CD Testing Limitation

The demonstrated pipeline contains:

```text
Source
   ↓
Build
   ↓
Deploy
```

The optional test stage was skipped in the demonstrated implementation.

Therefore, the pipeline does not currently provide a dedicated automated testing stage between build and deployment.

### Impact

A successful build does not necessarily prove that the application behaves correctly.

The current validation relies heavily on:

```text
Build success
     ↓
Deployment success
     ↓
Application access/login
```

### Future Work

Introduce an automated test stage:

```text
Build
  ↓
Automated Tests
  ↓
Deploy
```

Potential test categories could include:

- unit tests
- integration tests
- application smoke tests
- deployment validation tests

The exact testing strategy should be determined by the application's requirements.

---

# 4. Single Delivery Environment Limitation

The demonstrated architecture focuses on a single application deployment environment.

There is no implemented promotion model such as:

```text
Development
    ↓
Staging
    ↓
Production
```

### Impact

The project does not demonstrate:

- environment promotion
- staging validation
- production approval gates
- controlled production releases

### Future Work

Extend the pipeline into multiple environments:

```text
Source
   ↓
Build
   ↓
Test
   ↓
Development
   ↓
Staging
   ↓
Approval
   ↓
Production
```

This would provide stronger separation between development and production deployment.

---

# 5. Infrastructure as Code Limitation

The demonstrated implementation relies heavily on AWS console configuration.

The project does not establish a complete Infrastructure as Code implementation.

Resources such as:

- Elastic Beanstalk
- RDS
- CodeBuild
- CodePipeline
- IAM
- security groups
- S3 artifact storage

are not represented as a complete reusable IaC stack in this project.

### Impact

Recreating the environment requires repeating configuration steps.

This reduces:

- repeatability
- version-controlled infrastructure
- environment consistency
- automated provisioning

### Future Work

Introduce Infrastructure as Code using a suitable tool such as Terraform or AWS-native infrastructure tooling.

A future architecture could become:

```text
Terraform
    │
    ├── Networking
    ├── RDS
    ├── Elastic Beanstalk
    ├── IAM
    ├── CodeBuild
    └── CodePipeline
```

Infrastructure changes could then be reviewed and version-controlled alongside the project.

---

# 6. Secret Management Limitation

The demonstrated implementation uses application configuration changes during the build process.

The `buildspec.yml` modifies application configuration to supply environment-specific database information.

This demonstrates the configuration mechanism, but it is not a complete production secret-management solution.

### Risk

Database credentials should not be:

- committed to Git
- embedded directly in source code
- exposed in logs
- exposed through command history
- hardcoded into build configuration

### Future Work

Move sensitive values into a dedicated secret-management mechanism.

Conceptually:

```text
Secret Store
     │
     │ secure retrieval
     ▼
CodeBuild / Runtime
     │
     ▼
Application
```

The exact implementation should ensure that credentials are not unnecessarily exposed during the build or deployment process.

---

# 7. IAM Least-Privilege Limitation

The project included a practical IAM troubleshooting scenario where the CodePipeline service role lacked required Elastic Beanstalk permissions.

The immediate objective was to make the workflow function correctly.

The project does not establish a comprehensive least-privilege IAM design.

### Current Model

```text
CodePipeline
     ↓
Service Role
     ↓
AWS Service Permissions
```

### Future Work

Review and minimize permissions according to the actual actions required by:

- CodePipeline
- CodeBuild
- CodeConnections
- Elastic Beanstalk
- S3

A future implementation should explicitly document:

```text
Principal
    ↓
Action
    ↓
Resource
```

for each major permission boundary.

---

# 8. Observability Limitation

CloudWatch Logs are used for CodeBuild build output.

However, the project does not establish a comprehensive application observability platform.

The current logging model is primarily focused on build execution:

```text
CodeBuild
    ↓
CloudWatch Logs
```

It does not establish a full monitoring solution covering:

- application metrics
- infrastructure metrics
- centralized application logs
- distributed tracing
- alerting
- dashboards
- SLO/SLI monitoring

### Future Work

Extend the architecture with application and infrastructure observability:

```text
Application
    │
    ├── Logs
    ├── Metrics
    └── Traces
          │
          ▼
     Observability
          │
     ┌────┼────┐
     ▼    ▼    ▼
  Logs Metrics Alerts
```

---

# 9. High Availability and Resilience Limitation

The demonstrated Elastic Beanstalk environment uses two application instances and a rolling deployment configuration.

This demonstrates basic deployment availability behavior, but it does not constitute a complete disaster-recovery architecture.

The project does not demonstrate:

- multi-region deployment
- cross-region disaster recovery
- backup restoration testing
- regional failover
- recovery-time objectives
- recovery-point objectives

### Future Work

A more resilient architecture could introduce:

```text
Region A
   │
   ├── Application
   └── Database
        │
        │ replication / backup
        ▼
Region B
   │
   ├── Application
   └── Recovery Database
```

The exact design would depend on availability and recovery requirements.

---

# 10. Scalability Limitation

The demonstrated environment is designed to prove the CI/CD workflow rather than establish production-scale capacity.

The project does not include detailed:

- load testing
- capacity planning
- autoscaling validation
- performance benchmarking
- database performance testing

### Future Work

Introduce workload testing:

```text
Load Test
    ↓
Application
    ↓
Metrics
    ↓
Capacity Analysis
    ↓
Scaling Configuration
```

The resulting data could be used to determine appropriate instance counts, scaling policies, and database capacity.

---

# 11. Security Hardening Limitation

The project demonstrates the basic security boundaries required for the workflow, including IAM and security groups.

It does not establish a comprehensive production security program.

The project does not cover:

- centralized security monitoring
- vulnerability scanning
- dependency scanning
- container/image scanning
- WAF configuration
- formal compliance controls
- security incident response
- continuous security assessment

### Future Work

Introduce security checks into the delivery lifecycle:

```text
Source
   ↓
Build
   ↓
Security Scan
   ↓
Test
   ↓
Deploy
```

This would shift security validation earlier into the delivery process.

---

# 12. Deployment Strategy Limitation

The demonstrated deployment uses Elastic Beanstalk rolling deployment behavior.

This proves controlled deployment of the application, but the project does not implement more advanced release strategies.

It does not demonstrate:

- blue/green deployment
- canary deployment
- progressive delivery
- automated rollback policies
- production traffic shifting

### Future Work

A future release strategy could introduce:

```text
Current Version
      │
      ├──────────────┐
      ▼              ▼
Version A         Version B
                  │
                  ▼
             Validation
                  │
             Traffic Shift
                  │
                  ▼
              Production
```

---

# 13. Automated Rollback Limitation

The project validates successful deployment but does not establish a comprehensive automated rollback mechanism based on application health.

A production-grade workflow should be able to detect a failed release and return the environment to a known-good version.

### Future Work

Introduce:

```text
Deploy
  ↓
Health Check
  ↓
Success? ── Yes ──→ Continue
  │
  No
  ↓
Rollback
  ↓
Known-Good Version
```

Rollback conditions could eventually include:

- failed health checks
- failed smoke tests
- elevated error rates
- application startup failure
- deployment health degradation

---

# 14. Pipeline Governance Limitation

The demonstrated pipeline focuses on technical delivery.

It does not implement comprehensive organizational release controls such as:

- approval workflows
- change-management integration
- deployment windows
- separation of duties
- audit-oriented release records
- production authorization

### Future Work

A production-oriented pipeline could introduce approval gates:

```text
Build
  ↓
Test
  ↓
Staging
  ↓
Validation
  ↓
Approval
  ↓
Production
```

This would make the deployment process more suitable for environments with formal change controls.

---

# 15. Multi-Region Limitation

The demonstrated architecture operates within a single AWS region.

There is no multi-region deployment or active/passive regional failover.

### Future Work

A future implementation could investigate:

- multi-region application deployment
- cross-region database replication
- global traffic routing
- regional failover
- disaster-recovery automation

The design should be driven by actual availability requirements rather than adding multi-region complexity without a business need.

---

# 16. Reproducibility Limitation

Because much of the environment is configured through the AWS console, exact reproduction depends on manually repeating configuration steps.

This is useful for learning the AWS services but less suitable for repeatable engineering workflows.

### Future Work

Move toward:

```text
Configuration
      ↓
Version Control
      ↓
Infrastructure as Code
      ↓
Automated Provisioning
      ↓
Repeatable Environment
```

The goal would be to make a new environment reproducible from version-controlled definitions.

---

# 17. Documentation and Evidence Limitation

The repository contains documentation describing the architecture, implementation, validation, and future work.

However, evidence is intentionally limited to high-signal artifacts.

The project does not attempt to document every AWS console click.

This is deliberate.

The evidence should demonstrate meaningful engineering outcomes rather than reproduce the entire learning material.

### Future Work

Future iterations can improve evidence organization by mapping each important claim to:

```text
Claim
  ↓
Validation Procedure
  ↓
Observed Result
  ↓
Screenshot / Log
```

This makes the repository easier for another engineer or interviewer to evaluate.

---

# 18. Future Architecture

The demonstrated project can evolve from:

```text
Bitbucket
    ↓
CodePipeline
    ↓
CodeBuild
    ↓
Elastic Beanstalk
    ↓
RDS
```

toward a more complete production-oriented delivery platform:

```text
                         Source
                           │
                           ▼
                       Bitbucket
                           │
                           ▼
                    ┌─────────────┐
                    │ CodePipeline│
                    └──────┬──────┘
                           │
                           ▼
                       CodeBuild
                           │
                    ┌──────┴──────┐
                    ▼             ▼
               Unit Tests     Security Scan
                    │             │
                    └──────┬──────┘
                           ▼
                        Artifact
                           │
                           ▼
                        Staging
                           │
                           ▼
                    Integration Tests
                           │
                           ▼
                       Approval
                           │
                           ▼
                      Production
                           │
                           ▼
                  Observability Layer
                    ┌──────┼──────┐
                    ▼      ▼      ▼
                  Logs   Metrics Alerts
                           │
                           ▼
                      RDS / Backend
```

This represents a possible evolution rather than functionality currently implemented by the project.

---

# 19. Prioritized Future Work

Future improvements should be implemented incrementally rather than all at once.

### Priority 1 — Automated Testing

Add a test stage between build and deployment.

```text
Build
  ↓
Test
  ↓
Deploy
```

### Priority 2 — Secret Management

Remove sensitive configuration from build-time handling where possible and introduce a dedicated secret-management approach.

### Priority 3 — Infrastructure as Code

Convert manually configured AWS infrastructure into version-controlled infrastructure definitions.

### Priority 4 — Multi-Environment Delivery

Introduce development, staging, and production environments with controlled promotion.

### Priority 5 — Observability

Add application metrics, centralized logging, health monitoring, and alerting.

### Priority 6 — Security Automation

Add dependency, vulnerability, and configuration scanning into the delivery workflow.

### Priority 7 — Resilience

Introduce backup validation, recovery procedures, and stronger availability architecture based on actual requirements.

---

# 20. Learning and Career Value

The current implementation provides a foundation for several broader DevOps capabilities.

The project demonstrates practical understanding of:

```text
Git
 ↓
Source Control
 ↓
CI/CD
 ↓
Build Automation
 ↓
AWS
 ↓
IAM
 ↓
Deployment
 ↓
Runtime Validation
 ↓
Troubleshooting
```

Future iterations can progressively expand this foundation into:

```text
CI/CD
 +
Infrastructure as Code
 +
Testing
 +
Security
 +
Observability
 +
Reliability
```

This provides a natural progression from learning individual AWS services toward designing and operating a complete DevOps delivery system.

---

# 21. Final Limitations Statement

The project should be evaluated according to what it actually demonstrates.

It successfully demonstrates the core workflow:

```text
Bitbucket
    ↓
CodePipeline
    ↓
CodeBuild
    ↓
Artifact
    ↓
Elastic Beanstalk
    ↓
VProfile
    ↓
RDS
```

It also demonstrates practical troubleshooting of:

- a CodeBuild configuration error
- an IAM deployment permission issue

The project does **not** claim to be a complete enterprise production platform.

The major future evolution areas are:

```text
Automated Testing
        ↓
Secret Management
        ↓
Infrastructure as Code
        ↓
Multi-Environment Promotion
        ↓
Security Automation
        ↓
Observability
        ↓
Resilience
        ↓
Production Governance
```

These improvements should be treated as deliberate next iterations rather than missing requirements for the current learning project.

---

[← Back to README](../README.md)
