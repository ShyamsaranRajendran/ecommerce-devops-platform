# Environment Strategy

## Environment Overview

This document defines the three-tier environment strategy for the e-commerce platform deployment lifecycle.

## Environment Definition Table

| Environment | Purpose                                   | Deployment Strategy                       | Access               | Approval Required |
| ----------- | ----------------------------------------- | ----------------------------------------- | -------------------- | ----------------- |
| **Dev**     | Development and developer testing         | Auto deploy on push to `develop` branch   | All developers       | No                |
| **QA**      | Quality assurance and integration testing | Auto deploy on push to `release/*` branch | QA team, developers  | No                |
| **Prod**    | Production - serving real users           | Manual deploy from `main` branch          | Operations team only | Yes               |

## Environment Configuration

### Dev Environment

- **Purpose**: Rapid development and testing
- **Deployment**: Automated via Jenkins on every commit to `develop`
- **Infrastructure**: AWS EKS namespace `dev`
- **Database**: Shared dev databases with test data
- **Monitoring**: Basic logging enabled
- **Cost Optimization**: Scaled down during off-hours
- **Access**: Open to all development team members

### QA Environment

- **Purpose**: Integration testing, UAT, performance testing
- **Deployment**: Automated via Jenkins on push to `release/*` branches
- **Infrastructure**: AWS EKS namespace `qa`
- **Database**: QA-specific databases with sanitized production-like data
- **Monitoring**: Full monitoring with Prometheus & Grafana
- **Testing**: Automated smoke tests run post-deployment
- **Access**: QA team, developers (read-only)

### Prod Environment

- **Purpose**: Serving real customers
- **Deployment**: Manual trigger with approval gate
- **Infrastructure**: AWS EKS namespace `prod`
- **Database**: Production databases with backups and replication
- **Monitoring**: Full observability stack (Prometheus, Grafana, ELK)
- **High Availability**: Multiple replicas, auto-scaling enabled
- **Security**: Enhanced security policies, network policies
- **Access**: Restricted to operations/SRE team only

## Kubernetes Namespace Strategy

```yaml
# EKS Cluster: ecommerce-cluster

Namespaces:
├── dev          # Development environment
├── qa           # QA environment
└── prod         # Production environment
```

### Namespace Isolation

- **Network Policies**: Strict network policies between namespaces
- **Resource Quotas**: CPU/Memory limits per namespace
- **RBAC**: Role-based access control per namespace
- **Secrets**: Separate AWS Secrets Manager per environment

## Configuration Management

### Same Code, Different Configs

| Configuration              | Dev                     | QA                     | Prod              |
| -------------------------- | ----------------------- | ---------------------- | ----------------- |
| Database Connection String | dev-rds-postgres        | qa-rds-postgres        | prod-rds-postgres |
| API Gateway URL            | dev.api.ecommerce.local | qa.api.ecommerce.local | api.ecommerce.com |
| Replicas                   | 1                       | 2                      | 3+ (auto-scale)   |
| Log Level                  | DEBUG                   | INFO                   | WARN              |
| Monitoring                 | Basic                   | Full                   | Full + Alerts     |
| AWS Secrets Manager        | dev-secrets             | qa-secrets             | prod-secrets      |

### Configuration Sources

- **Kubernetes ConfigMaps**: Non-sensitive configuration
- **Kubernetes Secrets**: Sensitive data (managed via AWS Secrets Manager)
- **Helm Values**: Environment-specific values files
  - `values-dev.yaml`
  - `values-qa.yaml`
  - `values-prod.yaml`

## Deployment Flow

```
Code Push
  ↓
Feature Branch → Develop (Dev Auto-Deploy)
  ↓
Release Branch → QA (QA Auto-Deploy)
  ↓
Main Branch → Prod (Manual Approval Required)
```

## Environment Promotion Strategy

1. **Dev → QA**: Automatic after successful dev deployment and unit tests
2. **QA → Prod**: Manual promotion after:
   - All QA tests pass
   - Security scan passed
   - Performance tests passed
   - Approval from release manager

## Resource Allocation

| Environment | Node Count | Node Size       | Auto-scaling     |
| ----------- | ---------- | --------------- | ---------------- |
| Dev         | 2          | Standard_B2s    | No               |
| QA          | 2          | Standard_B2ms   | Yes (2-3 nodes)  |
| Prod        | 3          | Standard_D4s_v3 | Yes (3-10 nodes) |

## Backup Strategy

| Environment | Backup Frequency | Retention Period |
| ----------- | ---------------- | ---------------- |
| Dev         | Weekly           | 7 days           |
| QA          | Daily            | 30 days          |
| Prod        | Hourly           | 90 days          |

## Monitoring & Alerts

| Environment | Prometheus | Grafana | ELK | Alerts           |
| ----------- | ---------- | ------- | --- | ---------------- |
| Dev         | ✅         | ✅      | ❌  | ❌               |
| QA          | ✅         | ✅      | ✅  | Basic            |
| Prod        | ✅         | ✅      | ✅  | Full (PagerDuty) |

## Interview Key Points

> **Q: Why three environments?**
>
> - **Dev**: Fast feedback for developers
> - **QA**: Controlled testing environment
> - **Prod**: Production with manual gates for safety

> **Q: How do you manage different configs?**
>
> - Kubernetes ConfigMaps and Secrets
> - Helm values files per environment
> - Azure Key Vault for sensitive data
> - Same code, different configurations

> **Q: What prevents accidental prod deployment?**
>
> - Manual approval gate in Jenkins
> - RBAC restricting prod deployment access
> - Separate namespaces with network isolation
> - Code review requirements on main branch

> **Q: How do you ensure QA mirrors production?**
>
> - Same infrastructure setup (scaled down)
> - Production-like data (sanitized)
> - Same monitoring stack
> - Similar resource configurations
