# E-Commerce DevOps Project

> **Enterprise-grade microservices e-commerce platform on AWS EKS with complete CI/CD automation**

![Project Status](https://img.shields.io/badge/Status-In%20Development-yellow)
![AWS](https://img.shields.io/badge/Cloud-AWS-FF9900)
![Kubernetes](https://img.shields.io/badge/Orchestration-Kubernetes-326CE5)
![Java](https://img.shields.io/badge/Java-Spring%20Boot-6DB33F)

## 📋 Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Microservices](#microservices)
- [Tech Stack](#tech-stack)
- [Environments](#environments)
- [CI/CD Pipeline](#cicd-pipeline)
- [Repository Structure](#repository-structure)
- [Getting Started](#getting-started)
- [Documentation](#documentation)
- [Contributing](#contributing)

## 🎯 Overview

This project implements a production-ready e-commerce platform using **microservices architecture** deployed on **AWS Elastic Kubernetes Service (EKS)**. The platform demonstrates modern DevOps practices including infrastructure as code, continuous integration/deployment, comprehensive monitoring, and enterprise-grade security.

### Key Features

- ✅ Microservices architecture with Java Spring Boot and React
- ✅ Automated CI/CD pipeline with Jenkins
- ✅ Infrastructure as Code with Terraform
- ✅ Configuration management with Ansible
- ✅ Container orchestration with Kubernetes (EKS)
- ✅ Comprehensive monitoring with Prometheus & Grafana
- ✅ Centralized logging with ELK Stack
- ✅ Security scanning (SonarQube, Trivy)
- ✅ GitFlow branching strategy
- ✅ Blue-Green deployment for zero downtime

## 🏗️ Architecture

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         END USERS                                │
└────────────────────────────┬────────────────────────────────────┘
                             │ HTTPS
                             ↓
              AWS Application Load Balancer (ALB)
                             │
                             ↓
┌─────────────────────────────────────────────────────────────────┐
│                     Azure AKS Cluster                            │
│                                                                   │
│   ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐   │
│   │  Auth    │   │ Product  │   │  Order   │   │ Payment  │   │
│   │ Service  │   │ Service  │   │ Service  │   │ Service  │   │
│   │Java 8    │   │ Java 8   │   │ Java 8   │   │ Java 8   │   │
│   └────┬─────┘   └────┬─────┘   └────┬─────┘   └────┬─────┘   │
│        │              │              │              │           │
└────────┼──────────────┼──────────────┼──────────────┼───────────┘
         │              │              │              │
         ↓              ↓              ↓              ↓
    ┌────────┐    ┌────────┐    ┌────────┐    ┌────────┐
    │auth-db │    │prod-db │    │order-db│    │pay-db  │
    │Azure SQL    │Azure SQL    │Azure SQL    │Azure SQL
    └────────┘    └────────┘    └────────┘    └────────┘
```

### Architecture Principles

- **One Service = One Database**: Each microservice has its own dedicated database
- **No Shared Database**: Complete data isolation between services
- **REST Communication**: Services communicate via RESTful APIs
- **API Gateway Pattern**: Single entry point for all client requests

📖 [Detailed Architecture Documentation](docs/architecture.md)

## 🎯 Microservices

| Service               | Responsibility                            | Technology       | Port | Database              |
| --------------------- | ----------------------------------------- | ---------------- | ---- | --------------------- |
| **Auth Service**      | Authentication, JWT, RBAC                 | Java Spring Boot | 8081 | AWS RDS PostgreSQL    |
| **Product Service**   | Product catalog, search, details          | Java Spring Boot | 8082 | AWS RDS PostgreSQL    |
| **Cart Service**      | Shopping cart, temp state, expiration     | Java Spring Boot | 8085 | AWS ElastiCache Redis |
| **Order Service**     | Order lifecycle, status tracking          | Java Spring Boot | 8083 | AWS RDS PostgreSQL    |
| **Payment Service**   | Payment processing, webhooks              | Java Spring Boot | 8084 | AWS RDS PostgreSQL    |
| **Inventory Service** | Stock management, reservations, rollbacks | Java Spring Boot | 8086 | AWS RDS PostgreSQL    |
| **Frontend**          | React-based UI                            | React            | 3000 | N/A                   |

📖 [Microservices Documentation](docs/microservices.md)

## 🛠️ Tech Stack

### Development & Runtime

| Layer        | Technology         | Purpose                   |
| ------------ | ------------------ | ------------------------- |
| **Backend**  | Java Spring Boot   | Microservices development |
| **Frontend** | React              | User interface            |
| **Database** | AWS RDS PostgreSQL | Data persistence          |
| **API**      | REST               | Service communication     |

### DevOps & Infrastructure

| Layer             | Tool                    | Purpose                     |
| ----------------- | ----------------------- | --------------------------- |
| **SCM**           | GitHub / AWS CodeCommit | Source control              |
| **CI**            | Jenkins                 | Continuous Integration      |
| **CD**            | Jenkins + Helm          | Continuous Deployment       |
| **Containers**    | Docker                  | Containerization            |
| **Orchestration** | AWS EKS                 | Kubernetes management       |
| **IaC**           | Terraform               | Infrastructure provisioning |
| **Config Mgmt**   | Ansible                 | Configuration automation    |
| **Registry**      | AWS ECR                 | Docker image storage        |
| **Secrets**       | AWS Secrets Manager     | Secrets management          |

### Monitoring & Security

| Layer            | Tool           | Purpose                |
| ---------------- | -------------- | ---------------------- |
| **Monitoring**   | Prometheus     | Metrics collection     |
| **Dashboards**   | Grafana        | Visualization          |
| **Logging**      | ELK Stack      | Centralized logging    |
| **Code Quality** | SonarQube      | Static code analysis   |
| **Security**     | Trivy          | Vulnerability scanning |
| **Testing**      | JUnit, Postman | Unit & API testing     |

📖 [Complete Tool Stack Documentation](docs/tools.md)

## 🌍 Environments

The project uses a three-tier environment strategy:

| Environment | Purpose               | Deployment                    | Access         | Approval |
| ----------- | --------------------- | ----------------------------- | -------------- | -------- |
| **Dev**     | Development & testing | Auto (on push to `develop`)   | All developers | ❌       |
| **QA**      | Quality assurance     | Auto (on push to `release/*`) | QA team        | ❌       |
| **Prod**    | Production            | Manual (from `main`)          | Ops team only  | ✅       |

### Environment Isolation

- **Separate Kubernetes namespaces**: `dev`, `qa`, `prod`
- **Separate databases**: Per environment per service
- **Separate configurations**: ConfigMaps and Secrets per environment
- **Network policies**: Strict inter-namespace isolation

📖 [Environment Strategy Documentation](docs/environments.md)

## 🚀 CI/CD Pipeline

### CI Pipeline (Continuous Integration)

```
Code Push → Build → Unit Tests → SonarQube → Docker Build → Trivy Scan → Push to ECR
```

**CI Stages**:

1. **Code Checkout**: Clone repository
2. **Build**: Compile Java Spring Boot application with Maven
3. **Unit Tests**: Run test suite (80% coverage required)
4. **SonarQube**: Code quality & security analysis
5. **Docker Build**: Create container image
6. **Trivy Scan**: Security vulnerability scan
7. **Push to ECR**: Store image in AWS Elastic Container Registry
8. **Trigger CD**: Initiate deployment pipeline

### CD Pipeline (Continuous Deployment)

```
Terraform (Infra) → Ansible (Config) → Helm Deploy → Smoke Test → Approval (Prod) → Deploy
```

**CD Stages**:

1. **Terraform**: Provision/update infrastructure
2. **Ansible**: Configure servers and services
3. **Helm Deploy**: Deploy to Kubernetes
4. **Smoke Tests**: Validate deployment
5. **Manual Approval**: Production gate (manual)
6. **Production Deploy**: Blue-Green deployment
7. **Validation**: Post-deployment checks

### Deployment Strategy

- **Dev/QA**: Rolling updates (automated)
- **Production**: Blue-Green deployment (manual approval)
- **Rollback**: Automated on health check failure
- **Zero Downtime**: Guaranteed for production

📖 [CI/CD Flow Documentation](docs/ci-cd-flow.md)

## 📁 Repository Structure

```
ecommerce-devops/
├── docs/                          # Documentation
│   ├── architecture.md           # Architecture details
│   ├── microservices.md          # Service definitions
│   ├── environments.md           # Environment strategy
│   ├── tools.md                  # Tool stack details
│   ├── ci-cd-flow.md             # Pipeline documentation
│   └── git-strategy.md           # Branching strategy
│
├── services/                      # Microservices
│   ├── auth-service/             # Authentication service (Java Spring Boot)
│   ├── product-service/          # Product catalog service (Java Spring Boot)
│   ├── order-service/            # Order management service (Java Spring Boot)
│   └── payment-service/          # Payment processing service (Java Spring Boot)
│
├── frontend/                      # React frontend application
│
├── docker/                        # Dockerfiles and compose files
│
├── kubernetes/                    # Kubernetes manifests
│   ├── dev/                      # Dev environment manifests
│   ├── qa/                       # QA environment manifests
│   └── prod/                     # Production manifests
│
├── terraform/                     # Infrastructure as Code
│   └── modules/                  # Terraform modules
│
├── ansible/                       # Configuration Management
│   ├── playbooks/                # Ansible playbooks
│   ├── roles/                    # Ansible roles
│   └── inventory/                # Inventory files
│
├── helm/                          # Helm charts
│   ├── auth-service/             # Auth service chart
│   ├── product-service/          # Product service chart
│   ├── order-service/            # Order service chart
│   ├── payment-service/          # Payment service chart
│   └── frontend/                 # Frontend chart
│
├── jenkins/                       # Jenkins pipeline definitions
│   └── pipelines/                # Jenkinsfiles
│
├── monitoring/                    # Monitoring configurations
│   ├── prometheus/               # Prometheus config
│   ├── grafana/                  # Grafana dashboards
│   └── elk/                      # ELK stack config
│
└── README.md                      # This file
```

## 🚦 Getting Started

### Prerequisites

- Java 17+ and Maven
- Docker Desktop
- Node.js 18+ (for frontend)
- AWS CLI
- kubectl
- Terraform
- Helm 3
- Git

### Local Development Setup

1. **Clone the repository**

   ```bash
   git clone <repository-url>
   cd ecommerce-devops
   ```

2. **Run services locally with Docker Compose** (Coming soon)

   ```bash
   docker-compose up -d
   ```

3. **Access applications**
   - Frontend: http://localhost:3000
   - Auth Service: http://localhost:8081
   - Product Service: http://localhost:8082
   - Order Service: http://localhost:8083
   - Payment Service: http://localhost:8084

### Deployment to AWS

Deployment instructions will be added as the project progresses.

## 📚 Documentation

Comprehensive documentation is available in the [docs/](docs/) directory:

- [Architecture Overview](docs/architecture.md) - System architecture and design
- [Microservices Design](docs/microservices.md) - Service responsibilities and patterns
- [Environment Strategy](docs/environments.md) - Dev, QA, and Prod setup
- [DevOps Tools](docs/tools.md) - Complete toolchain documentation
- [CI/CD Pipeline](docs/ci-cd-flow.md) - Pipeline stages and logic
- [Git Strategy](docs/git-strategy.md) - Branching model and workflows

## 🌿 Git Workflow

### Branch Strategy

```
main                → Production
  ├── release/*     → Pre-production (QA)
  │     └── develop → Integration
  │           └── feature/* → Feature development
  └── hotfix/*      → Emergency fixes
```

### Branch Protection

- ✅ `main`: 2 approvals required, all checks must pass
- ✅ `develop`: 1 approval required, CI checks must pass
- ✅ No direct pushes to `main` or `develop`
- ✅ All commits must be signed

📖 [Git Strategy Documentation](docs/git-strategy.md)

## 🤝 Contributing

We follow a strict GitFlow workflow. Please follow these guidelines:

1. **Create a feature branch** from `develop`

   ```bash
   git checkout develop
   git checkout -b feature/ECOM-XXX-description
   ```

2. **Make your changes** and commit with meaningful messages

   ```bash
   git commit -m "feat(auth): add JWT validation"
   ```

3. **Push and create a Pull Request**

   ```bash
   git push origin feature/ECOM-XXX-description
   ```

4. **Wait for CI checks** and code review approval

5. **Merge after approval** and delete the feature branch

## 🎓 Learning Resources

This project is designed for learning and demonstrating:

- Microservices architecture
- Kubernetes orchestration
- CI/CD automation
- Infrastructure as Code
- DevOps best practices
- Cloud-native development

## 📊 Project Status

### ✅ Completed (Day 1)

- [x] Architecture design
- [x] Microservices definition
- [x] Environment strategy
- [x] Tool stack selection
- [x] CI/CD flow design
- [x] Git strategy
- [x] Repository structure
- [x] Documentation

### 🔄 In Progress

- [ ] Microservice implementation
- [ ] Docker containerization
- [ ] Kubernetes manifests
- [ ] Terraform modules
- [ ] Jenkins pipelines
- [ ] Monitoring setup

### 📅 Upcoming

- [ ] Service implementation
- [ ] Database schema
- [ ] API integration
- [ ] Frontend development
- [ ] Security hardening
- [ ] Performance optimization

## 📞 Contact & Support

For questions or support, please open an issue in the repository.

---

## 📝 License

This project is created for educational and demonstration purposes.

---

## 🏆 Key Interview Points

> **Q: What makes this project production-ready?**
>
> - Microservices architecture with clear separation of concerns
> - Complete CI/CD automation with quality gates
> - Infrastructure as Code for reproducibility
> - Comprehensive monitoring and logging
> - Security scanning at every stage
> - Zero-downtime deployment strategy

> **Q: How do you ensure high availability?**
>
> - Multiple replicas per service
> - Health checks and auto-restart
> - Blue-Green deployment
> - Auto-scaling (HPA and Cluster Autoscaler)
> - Multi-zone deployment in AKS

> **Q: How do you handle secrets?**
>
> - Azure Key Vault for centralized secret management
> - Kubernetes CSI driver for secret injection
> - No secrets in code or configuration files
> - Audit logs for secret access

> **Q: What's your deployment frequency?**
>
> - Dev: Multiple times per day (automatic)
> - QA: Daily (automatic on release branches)
> - Prod: Weekly or as needed (manual approval)

---

**Built with ❤️ for learning modern DevOps practices**
