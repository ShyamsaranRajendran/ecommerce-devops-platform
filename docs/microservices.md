# Microservices Architecture

## Service Definitions

This document defines the microservices architecture for the e-commerce platform, with clear responsibilities and technology choices.

## Service Responsibility Table

| Service               | Responsibility                                                                             | Technology       | Port |
| --------------------- | ------------------------------------------------------------------------------------------ | ---------------- | ---- |
| **Auth Service**      | User authentication, signup, login, JWT token management, role-based access control (RBAC) | Java Spring Boot | 8081 |
| **Product Service**   | Product catalog management, search functionality, product details, category management     | Java Spring Boot | 8082 |
| **Cart Service**      | Shopping cart management, temporary cart state, cart items CRUD, cart expiration           | Java Spring Boot | 8085 |
| **Order Service**     | Order lifecycle management, order creation, status tracking, order history                 | Java Spring Boot | 8083 |
| **Payment Service**   | Payment processing, payment gateway integration, transaction management, webhook handling  | Java Spring Boot | 8084 |
| **Inventory Service** | Stock management, inventory reservation, commit/rollback operations, stock consistency     | Java Spring Boot | 8086 |
| **Frontend**          | User interface, customer-facing application, admin dashboard                               | React            | 3000 |

## Architectural Principles

### ✅ One Service = One Database

- Each microservice has its own dedicated database
- No shared database between services
- Ensures loose coupling and independent scalability
- Allows service-specific database technology choices

### ✅ No Shared Database

- Auth Service → AWS RDS PostgreSQL (auth-db)
- Product Service → AWS RDS PostgreSQL (product-db)
- Cart Service → AWS ElastiCache Redis (cart-db)
- Order Service → AWS RDS PostgreSQL (order-db)
- Payment Service → AWS RDS PostgreSQL (payment-db)
- Inventory Service → AWS RDS PostgreSQL (inventory-db)

### ✅ Communication via REST APIs

- Inter-service communication through RESTful APIs
- Synchronous HTTP/HTTPS calls
- JSON as data exchange format
- API Gateway pattern for frontend-to-backend communication

## Service Communication Flow

```
Frontend
  ↓
API Gateway / Ingress
  ↓
  ├─→ Auth Service (JWT validation)
  ├─→ Product Service
  ├─→ Order Service → Payment Service
  └─→ Payment Service
```

## Database Strategy

| Service         | Database   | Type               | Purpose                              |
| --------------- | ---------- | ------------------ | ------------------------------------ |
| Auth Service    | auth-db    | AWS RDS PostgreSQL | User credentials, roles, permissions |
| Product Service | product-db | AWS RDS PostgreSQL | Product catalog, inventory           |
| Order Service   | order-db   | AWS RDS PostgreSQL | Orders, order items, status          |
| Payment Service | payment-db | AWS RDS PostgreSQL | Transactions, payment status         |

## Service Independence

- **Deployment**: Each service can be deployed independently
- **Scaling**: Scale services based on individual load
- **Development**: Separate teams can work on different services
- **Technology**: Freedom to choose appropriate tech stack per service
- **Failure Isolation**: One service failure doesn't bring down the entire system

## Interview Key Points

> **Q: Why microservices over monolith?**
>
> - Independent deployment and scaling
> - Technology flexibility
> - Better fault isolation
> - Easier to understand and maintain smaller codebases

> **Q: How do services communicate?**
>
> - RESTful APIs for synchronous communication
> - Each service exposes REST endpoints
> - API Gateway acts as single entry point

> **Q: What if a service fails?**
>
> - Circuit breaker pattern to prevent cascading failures
> - Service mesh (future: Istio) for resilience
> - Health checks and auto-restart in Kubernetes
