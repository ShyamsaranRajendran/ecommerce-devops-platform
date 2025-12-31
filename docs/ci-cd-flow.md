# CI/CD Pipeline Flow

## Overview

This document defines the Continuous Integration and Continuous Deployment pipeline logic for the e-commerce platform. This is the **logical flow only** - no YAML/code implementation yet.

## CI Pipeline (Continuous Integration)

### CI Flow Diagram

```
Developer Push to Branch
         ↓
GitHub/AWS CodeCommit (Webhook Trigger)
         ↓
Jenkins CI Pipeline Start
         ↓
┌────────────────────────────────────────┐
│  Stage 1: Code Checkout                │
│  - Clone repository                    │
│  - Checkout specific branch            │
└────────────┬───────────────────────────┘
             ↓
┌────────────────────────────────────────┐
│  Stage 2: Build Application            │
│  - Restore dependencies (Maven)         │
│  - Compile code                        │
│  - Generate artifacts                  │
└────────────┬───────────────────────────┘
             ↓
┌────────────────────────────────────────┐
│  Stage 3: Run Unit Tests               │
│  - Execute unit test suite             │
│  - Generate test reports               │
│  - Code coverage analysis              │
│  ⚠ Exit if tests fail                  │
└────────────┬───────────────────────────┘
             ↓
┌────────────────────────────────────────┐
│  Stage 4: SonarQube Code Analysis      │
│  - Static code analysis                │
│  - Security vulnerability scan         │
│  - Code quality checks                 │
│  - Quality gate evaluation             │
│  ⚠ Exit if quality gate fails          │
└────────────┬───────────────────────────┘
             ↓
┌────────────────────────────────────────┐
│  Stage 5: Docker Image Build           │
│  - Build Docker image                  │
│  - Tag with version/commit SHA         │
│  - Multi-stage build for optimization  │
└────────────┬───────────────────────────┘
             ↓
┌────────────────────────────────────────┐
│  Stage 6: Trivy Security Scan          │
│  - Scan Docker image for CVEs          │
│  - Check dependencies                  │
│  - Generate security report            │
│  ⚠ Exit if critical vulnerabilities   │
└────────────┬───────────────────────────┘
             ↓
┌────────────────────────────────────────┐
│  Stage 7: Push to AWS Elastic           │
│    Container Registry (ECR)           │
│  - Authenticate with ECR                │
│  - Push Docker image                   │
│  - Tag image with environment          │
└────────────┬───────────────────────────┘
             ↓
┌────────────────────────────────────────┐
│  Stage 8: Trigger CD Pipeline          │
│  - Pass image tag to CD pipeline       │
│  - Send notification                   │
└────────────────────────────────────────┘
```

### CI Pipeline Details

#### Stage 1: Code Checkout

- **Action**: Clone the Git repository
- **Branch**: Automatically detected (multi-branch pipeline)
- **Credentials**: Jenkins credentials for GitHub/AWS CodeCommit
- **Output**: Source code in workspace

#### Stage 2: Build Application

- **Technology**: Java 17 + Maven
- **Commands**:
  - `mvn clean` - Clean previous builds
  - `mvn compile` - Compile application
  - `mvn package` - Create JAR/WAR package
- **Output**: Compiled binaries and artifacts
- **Failure Action**: Stop pipeline, notify team

#### Stage 3: Run Unit Tests

- **Framework**: JUnit 5 / TestNG
- **Commands**:
  - `mvn test` - Execute unit tests
  - Generate code coverage report (JaCoCo)
- **Coverage Threshold**: Minimum 80%
- **Output**: Test results (XML/HTML)
- **Failure Action**: Mark build as failed, stop pipeline

#### Stage 4: SonarQube Code Analysis

- **Analysis Type**: Static code analysis
- **Checks**:
  - Code smells
  - Bugs
  - Security vulnerabilities
  - Code duplications
  - Technical debt
- **Quality Gates**:
  - Coverage > 80%
  - No critical/blocker issues
  - Duplicated lines < 3%
- **Output**: Quality report URL
- **Failure Action**: Block merge if quality gate fails

#### Stage 5: Docker Image Build

- **Dockerfile Location**: Root of each service
- **Build Strategy**: Multi-stage build
  - Stage 1: Build application
  - Stage 2: Runtime (minimal image)
- **Image Tagging**:
  - `latest` for develop branch
  - `<branch-name>-<commit-sha>` for feature branches
  - `v<version>` for releases
- **Output**: Docker image

#### Stage 6: Trivy Security Scan

- **Scan Type**: Container image vulnerability scan
- **Checks**:
  - OS package vulnerabilities
  - Application dependency vulnerabilities
  - Misconfigurations
- **Severity Threshold**: Block on HIGH and CRITICAL
- **Output**: Security report (JSON/HTML)
- **Failure Action**: Stop pipeline if critical CVEs found

#### Stage 7: Push to ECR

- **Registry**: AWS Elastic Container Registry
- **Authentication**: AWS IAM Role / Access Keys
- **Naming Convention**:
  - `<account-id>.dkr.ecr.<region>.amazonaws.com/<service>:<tag>`
  - Example: `123456789.dkr.ecr.us-east-1.amazonaws.com/auth-service:v1.0.0`
- **Output**: Image URL in ECR

#### Stage 8: Trigger CD Pipeline

- **Condition**: Only on successful CI completion
- **Parameters Passed**:
  - Image tag
  - Git commit SHA
  - Environment (dev/qa/prod)
- **Notification**: Slack/Teams/Email notification

### CI Triggers

| Branch      | Trigger       | Auto-Deploy To              |
| ----------- | ------------- | --------------------------- |
| `feature/*` | On push       | None (CI only)              |
| `develop`   | On push       | Dev environment             |
| `release/*` | On push       | QA environment              |
| `main`      | On push/merge | Manual approval for Prod    |
| `hotfix/*`  | On push       | QA, then Prod with approval |

## CD Pipeline (Continuous Deployment)

### CD Flow Diagram

```
CI Pipeline Completes Successfully
         ↓
Jenkins CD Pipeline Start
         ↓
┌────────────────────────────────────────┐
│  Stage 1: Infrastructure Provisioning  │
│         (Terraform)                    │
│  - Plan infrastructure changes         │
│  - Apply Terraform (if changes)        │
│  - Create/Update EKS, RDS, etc.        │
└────────────┬───────────────────────────┘
             ↓
┌────────────────────────────────────────┐
│  Stage 2: Configuration Management     │
│         (Ansible)                      │
│  - Configure EKS nodes                 │
│  - Deploy monitoring agents            │
│  - Update system configurations        │
└────────────┬───────────────────────────┘
             ↓
┌────────────────────────────────────────┐
│  Stage 3: Deploy with Helm             │
│  - Helm chart selection                │
│  - Values file per environment         │
│  - Helm upgrade/install                │
│  - Apply Kubernetes manifests          │
└────────────┬───────────────────────────┘
             ↓
┌────────────────────────────────────────┐
│  Stage 4: Smoke Tests                  │
│  - Health check endpoints              │
│  - Basic API tests                     │
│  - Database connectivity               │
│  ⚠ Rollback if smoke tests fail        │
└────────────┬───────────────────────────┘
             ↓
┌────────────────────────────────────────┐
│  Stage 5: Manual Approval (Prod Only)  │
│  - Notify release manager              │
│  - Wait for manual approval            │
│  - Proceed only after approval         │
└────────────┬───────────────────────────┘
             ↓
┌────────────────────────────────────────┐
│  Stage 6: Production Deployment        │
│  - Blue-Green deployment strategy      │
│  - Deploy to prod namespace            │
│  - Monitor during rollout              │
└────────────┬───────────────────────────┘
             ↓
┌────────────────────────────────────────┐
│  Stage 7: Post-Deployment Validation   │
│  - Run integration tests               │
│  - Verify metrics in Prometheus        │
│  - Check logs in ELK                   │
│  - Send success notification           │
└────────────────────────────────────────┘
```

### CD Pipeline Details

#### Stage 1: Infrastructure Provisioning (Terraform)

- **Tool**: Terraform
- **Actions**:
  - `terraform init` - Initialize backend
  - `terraform plan` - Preview changes
  - `terraform apply` - Apply changes
- **Resources Provisioned**:
  - EKS cluster (if not exists)
  - AWS RDS databases
  - Virtual Private Clouds (VPC)
  - Load balancers
  - Storage accounts
- **State Management**: AWS S3 backend with DynamoDB locking
- **Output**: Infrastructure ready

#### Stage 2: Configuration Management (Ansible)

- **Tool**: Ansible
- **Playbooks**:
  - Configure AKS nodes
  - Deploy monitoring stack (Prometheus, Grafana)
  - Configure logging (ELK)
  - Update application configurations
- **Inventory**: Dynamic inventory from AWS
- **Output**: Configured infrastructure

#### Stage 3: Deploy with Helm

- **Tool**: Helm 3
- **Chart Location**: `helm/<service-name>`
- **Values Files**:
  - `values-dev.yaml`
  - `values-qa.yaml`
  - `values-prod.yaml`
- **Deployment Strategy**:
  - Rolling update (default)
  - Blue-Green (production)
  - Canary (advanced, future)
- **Commands**:
  - `helm upgrade --install <release> <chart> -f values-<env>.yaml`
- **Kubernetes Resources Deployed**:
  - Deployments
  - Services
  - Ingress
  - ConfigMaps
  - Secrets (from Key Vault)

#### Stage 4: Smoke Tests

- **Purpose**: Quick validation of deployment
- **Tests**:
  - Health check endpoint (`/health`)
  - Readiness check
  - Database connectivity
  - External API connectivity
- **Failure Action**: Automatic rollback with `helm rollback`
- **Success**: Proceed to next stage

#### Stage 5: Manual Approval (Production Only)

- **Trigger**: Required for production deployments
- **Approvers**: Release manager, DevOps lead
- **Notification**: Slack/Teams/Email
- **Information Provided**:
  - Changes being deployed
  - Test results
  - Security scan results
  - Rollback plan
- **Timeout**: 24 hours (auto-reject after)

#### Stage 6: Production Deployment

- **Strategy**: Blue-Green deployment
  - Blue: Current production
  - Green: New version
  - Switch traffic after validation
- **Monitoring**: Real-time monitoring during deployment
- **Progressive Rollout**: 25% → 50% → 100% of traffic
- **Rollback Ready**: Instant rollback capability

#### Stage 7: Post-Deployment Validation

- **Integration Tests**: Run full test suite
- **Metrics Validation**:
  - Check Prometheus metrics
  - Verify no error rate spike
  - Confirm response times
- **Logs Check**: Verify no errors in ELK
- **Notification**: Send deployment success to team

### Deployment Strategy by Environment

#### Dev Environment

- **Trigger**: Automatic on push to `develop`
- **Approval**: None
- **Strategy**: Rolling update
- **Rollback**: Automatic on failure
- **Notification**: Basic (Slack)

#### QA Environment

- **Trigger**: Automatic on push to `release/*`
- **Approval**: None
- **Strategy**: Rolling update
- **Testing**: Automated smoke + integration tests
- **Rollback**: Automatic on test failure
- **Notification**: Detailed (Slack + Email)

#### Production Environment

- **Trigger**: Manual from `main` branch
- **Approval**: Required (Release Manager)
- **Strategy**: Blue-Green deployment
- **Testing**: Full test suite
- **Monitoring**: Enhanced monitoring during rollout
- **Rollback**: Manual or automatic on critical errors
- **Notification**: Full (Slack + Email + PagerDuty)

## Rollback Strategy

### Automatic Rollback Triggers

- Smoke tests fail
- Health check fails
- Error rate > 5%
- Response time > 2x baseline

### Manual Rollback

```
Helm Rollback Command
         ↓
helm rollback <release> <revision>
         ↓
Previous Version Restored
         ↓
Validate Rollback Success
```

### Rollback Testing

- Monthly rollback drills in QA
- Document rollback procedures
- Practice rollback scenarios

## Pipeline Notifications

### Notification Channels

- **Slack**: Real-time updates
- **Email**: Detailed reports
- **PagerDuty**: Critical production issues

### Notification Events

| Event           | Dev | QA  | Prod |
| --------------- | --- | --- | ---- |
| Build started   | ❌  | ❌  | ✅   |
| Build failed    | ✅  | ✅  | ✅   |
| Tests failed    | ✅  | ✅  | ✅   |
| Security issues | ✅  | ✅  | ✅   |
| Deploy started  | ❌  | ❌  | ✅   |
| Deploy success  | ❌  | ✅  | ✅   |
| Deploy failed   | ✅  | ✅  | ✅   |

## Pipeline Monitoring

### Metrics to Track

- Build duration
- Test success rate
- Deployment frequency
- Mean time to recovery (MTTR)
- Change failure rate
- Lead time for changes

### DORA Metrics Goals

| Metric                | Target                 |
| --------------------- | ---------------------- |
| Deployment Frequency  | Multiple times per day |
| Lead Time for Changes | < 1 hour               |
| Mean Time to Recover  | < 1 hour               |
| Change Failure Rate   | < 15%                  |

## Security Gates

### Pipeline Security Checkpoints

1. **Code Commit**: Pre-commit hooks (linting, secrets scan)
2. **Build**: SonarQube quality gate
3. **Image Build**: Trivy security scan
4. **Deployment**: RBAC checks, approval gates
5. **Runtime**: Network policies, pod security policies

### Security Scan Results Handling

| Severity | Action                                 |
| -------- | -------------------------------------- |
| CRITICAL | Block deployment, notify security team |
| HIGH     | Block deployment, create ticket        |
| MEDIUM   | Allow with warning, create ticket      |
| LOW      | Allow, log for review                  |

## Interview Key Points

> **Q: Explain your CI/CD pipeline**
>
> - CI: Code → Build → Test → Scan → Dockerize → Push to ACR
> - CD: Terraform → Ansible → Helm Deploy → Smoke Test → Approval (Prod) → Deploy

> **Q: How do you ensure deployment safety?**
>
> - Automated tests at every stage
> - Security scans (SonarQube, Trivy)
> - Smoke tests after deployment
> - Manual approval for production
> - Blue-Green deployment strategy
> - Automatic rollback on failures

> **Q: What if a deployment fails in production?**
>
> - Automatic health checks detect failure
> - Helm rollback to previous version
> - Alert sent to on-call team
> - Post-incident review
> - Root cause analysis

> **Q: How do you handle database migrations?**
>
> - Migrations run before application deployment
> - Backward-compatible migrations
> - Rollback scripts available
> - Test migrations in Dev/QA first
> - Manual approval for production schema changes

> **Q: What's the typical deployment time?**
>
> - Dev: 5-10 minutes (auto)
> - QA: 10-15 minutes (auto)
> - Prod: 20-30 minutes (with approval and progressive rollout)

> **Q: How do you ensure zero-downtime deployments?**
>
> - Rolling updates with readiness probes
> - Blue-Green deployments in production
> - Load balancer health checks
> - Graceful shutdown handling
