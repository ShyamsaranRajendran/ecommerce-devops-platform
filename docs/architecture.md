# High-Level Architecture

## System Architecture Overview

This document provides a comprehensive view of the e-commerce platform architecture deployed on AWS Elastic Kubernetes Service (EKS).

## 🎯 Business Requirements

### Core E-Commerce Capabilities

The system must support the following core functionalities:

1. **User Management**

   - User registration with email validation
   - Secure login with JWT authentication
   - Role-based access control (RBAC)
   - User profile management

2. **Product Management**

   - Product catalog browsing
   - Product search and filtering
   - Product details viewing
   - Category-based organization

3. **Shopping Cart**

   - Add items to cart
   - Update item quantities
   - Remove items from cart
   - View cart summary
   - Cart expiration after inactivity

4. **Order Processing**

   - Order creation from cart
   - Order status tracking
   - Order history retrieval
   - Order cancellation (before payment)

5. **Payment Processing**

   - Payment initiation
   - Payment status tracking
   - Payment webhook handling
   - Mock payment gateway integration

6. **Inventory Management**
   - Real-time stock tracking
   - Inventory reservation during checkout
   - Inventory commit on payment success
   - Inventory release on payment failure
   - Stock consistency across operations

### Business Rules

- ✅ All operations must be accessible via REST APIs
- ✅ Inventory must be reserved (not deducted) until payment confirmation
- ✅ Orders must maintain ACID properties for critical transactions
- ✅ Users must be authenticated for cart and order operations
- ✅ Payment failures must automatically release reserved inventory

## 🏗️ Domain Boundaries & Microservices

### Service Responsibility Matrix

| Service               | Responsibility                                                                   | Database                  | Port |
| --------------------- | -------------------------------------------------------------------------------- | ------------------------- | ---- |
| **Auth Service**      | User management, authentication, JWT token generation & validation, RBAC         | auth-db (PostgreSQL)      | 8081 |
| **Product Service**   | Product catalog, search, filtering, product details, category management         | product-db (PostgreSQL)   | 8082 |
| **Cart Service**      | Shopping cart management, temporary cart state, cart items CRUD, cart expiration | cart-db (Redis)           | 8085 |
| **Order Service**     | Order lifecycle, order creation, status management, order history                | order-db (PostgreSQL)     | 8083 |
| **Payment Service**   | Payment orchestration, payment gateway integration, webhook handling             | payment-db (PostgreSQL)   | 8084 |
| **Inventory Service** | Stock management, inventory reservation, commit/rollback operations              | inventory-db (PostgreSQL) | 8086 |

### 🔒 Critical Rule: Service Isolation

**No service accesses another service's database directly.**

- Each service owns its data
- Inter-service communication via REST APIs only
- Database schemas are private to each service
- Services are independently deployable and scalable

## 📊 Core Entities & Data Models

### Auth Service Entities

#### User

```java
@Entity
@Table(name = "users")
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private UUID id;

    @Column(unique = true, nullable = false)
    private String email;

    @Column(nullable = false)
    private String passwordHash;

    @Enumerated(EnumType.STRING)
    private Role role; // CUSTOMER, ADMIN

    @CreationTimestamp
    private LocalDateTime createdAt;

    @UpdateTimestamp
    private LocalDateTime updatedAt;
}
```

### Product Service Entities

#### Product

```java
@Entity
@Table(name = "products")
public class Product {
    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private UUID id;

    @Column(nullable = false)
    private String name;

    @Column(columnDefinition = "TEXT")
    private String description;

    @Column(nullable = false)
    private BigDecimal price;

    private String category;

    private String imageUrl;

    @Column(nullable = false)
    private Boolean active = true;

    @CreationTimestamp
    private LocalDateTime createdAt;
}
```

### Inventory Service Entities

#### Inventory

```java
@Entity
@Table(name = "inventory")
public class Inventory {
    @Id
    @Column(name = "product_id")
    private UUID productId;

    @Column(nullable = false)
    private Integer availableQuantity;

    @Column(nullable = false)
    private Integer reservedQuantity;

    @Version
    private Long version; // Optimistic locking

    @UpdateTimestamp
    private LocalDateTime updatedAt;
}
```

### Cart Service Entities

#### Cart

```java
@RedisHash(value = "Cart", timeToLive = 3600) // 1 hour TTL
public class Cart {
    @Id
    private UUID id;

    private UUID userId;

    private List<CartItem> items = new ArrayList<>();

    private LocalDateTime expiresAt;

    private LocalDateTime createdAt;
}
```

#### CartItem

```java
public class CartItem {
    private UUID productId;
    private String productName;
    private BigDecimal price;
    private Integer quantity;
}
```

### Order Service Entities

#### Order

```java
@Entity
@Table(name = "orders")
public class Order {
    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private UUID id;

    @Column(nullable = false)
    private UUID userId;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private OrderStatus status; // CREATED, PAID, CANCELLED, COMPLETED

    @Column(nullable = false)
    private BigDecimal totalAmount;

    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL)
    private List<OrderItem> items = new ArrayList<>();

    @CreationTimestamp
    private LocalDateTime createdAt;

    @UpdateTimestamp
    private LocalDateTime updatedAt;
}
```

#### OrderItem

```java
@Entity
@Table(name = "order_items")
public class OrderItem {
    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private UUID id;

    @ManyToOne
    @JoinColumn(name = "order_id")
    private Order order;

    @Column(nullable = false)
    private UUID productId;

    @Column(nullable = false)
    private String productName;

    @Column(nullable = false)
    private BigDecimal price;

    @Column(nullable = false)
    private Integer quantity;
}
```

### Payment Service Entities

#### Payment

```java
@Entity
@Table(name = "payments")
public class Payment {
    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private UUID id;

    @Column(nullable = false, unique = true)
    private UUID orderId;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private PaymentStatus status; // PENDING, SUCCESS, FAILED

    @Column(nullable = false)
    private BigDecimal amount;

    private String provider; // MOCK, STRIPE, RAZORPAY

    private String transactionId;

    @CreationTimestamp
    private LocalDateTime createdAt;

    @UpdateTimestamp
    private LocalDateTime updatedAt;
}
```

## 🔄 Order Flow (Critical Business Logic)

### Order Creation Flow

```
1. User initiates checkout from Cart
   ↓
2. Cart Service validates cart items
   ↓
3. Order Service creates order (status: CREATED)
   ↓
4. Inventory Service reserves stock
   ├─ Success → Continue
   └─ Failure → Order CANCELLED, return error
   ↓
5. Payment Service initiates payment (status: PENDING)
   ↓
6. Return order details to user
```

### Payment Success Flow

```
1. Payment webhook received (SUCCESS)
   ↓
2. Payment Service updates payment status
   ↓
3. Order Service updates order (status: PAID)
   ↓
4. Inventory Service commits reservation
   ↓
5. Cart Service clears cart
   ↓
6. Notification sent to user
```

### Payment Failure Flow

```
1. Payment webhook received (FAILED)
   ↓
2. Payment Service updates payment status
   ↓
3. Order Service updates order (status: CANCELLED)
   ↓
4. Inventory Service releases reservation
   ↓
5. Notification sent to user
```

### Key Flow Principles

- ✅ **Inventory is RESERVED, not DEDUCTED** until payment success
- ✅ **Idempotent operations** for payment webhooks
- ✅ **Compensating transactions** for failures
- ✅ **Eventual consistency** for non-critical updates
- ✅ **Strong consistency** for inventory and payments

## Architecture Flow Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                        END USERS / CUSTOMERS                     │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             │ HTTPS
                             ↓
┌─────────────────────────────────────────────────────────────────┐
│          AWS Application Load Balancer (ALB) / Ingress          │
│                     (SSL Termination, WAF)                       │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ↓
┌─────────────────────────────────────────────────────────────────┐
│                      AWS EKS Cluster                             │
│                                                                   │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │ Namespace:   │  │ Namespace:   │  │ Namespace:   │          │
│  │    dev       │  │    qa        │  │    prod      │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
│                                                                   │
│  ┌───────────────────────────────────────────────────────┐      │
│  │              Frontend (React)                          │      │
│  │                 Port: 3000                             │      │
│  └──────────────────────────┬────────────────────────────┘      │
│                             │                                    │
│    ┌────────────────────────┼────────────────────────┐          │
│    │                        │                        │          │
│    ↓                        ↓                        ↓          │
│  ┌──────────┐   ┌──────────────┐   ┌──────────────┐           │
│  │Auth Svc  │   │ Product Svc  │   │  Cart Svc    │           │
│  │8081      │   │ 8082         │   │  8085        │           │
│  └────┬─────┘   └──────┬───────┘   └──────┬───────┘           │
│       │                │                   │                    │
│       │                │      ┌────────────┴────────┐           │
│       │                │      │                     │           │
│       │                │      ↓                     ↓           │
│       │                │  ┌──────────────┐   ┌──────────────┐  │
│       │                │  │ Order Svc    │   │Inventory Svc │  │
│       │                │  │ 8083         │   │ 8086         │  │
│       │                │  └──────┬───────┘   └──────┬───────┘  │
│       │                │         │                   │          │
│       │                │         └──────┐    ┌───────┘          │
│       │                │                │    │                  │
│       │                │         ┌──────▼────▼───┐              │
│       │                │         │ Payment Svc   │              │
│       │                │         │ 8084          │              │
│       │                │         └───────┬───────┘              │
│       │                │                 │                      │
└───────┼────────────────┼─────────────────┼──────────────────────┘
        │                │                 │
        ↓                ↓                 ↓
┌─────────────────────────────────────────────────────────────────┐
│                    AWS Data Layer                                │
│                                                                   │
│  ┌─────────┐  ┌──────────┐  ┌────────┐  ┌──────────┐          │
│  │ auth-db │  │product-db│  │cart-db │  │ order-db │          │
│  │AWS RDS  │  │ AWS RDS  │  │ Redis  │  │ AWS RDS  │          │
│  │Postgres │  │ Postgres │  │ElastiC.│  │ Postgres │          │
│  └─────────┘  └──────────┘  └────────┘  └──────────┘          │
│                                                                   │
│  ┌──────────┐  ┌─────────────┐                                  │
│  │payment-db│  │inventory-db │                                  │
│  │ AWS RDS  │  │  AWS RDS    │                                  │
│  │ Postgres │  │  Postgres   │                                  │
│  └──────────┘  └─────────────┘                                  │
└─────────────────────────────────────────────────────────────────┘
          │                  │                         │
          └──────────────────┴─────────────────────────┘
                             │
                             ↓
┌─────────────────────────────────────────────────────────────────┐
│                  Observability Stack                             │
│                                                                   │
│  ┌─────────────┐  ┌─────────────┐  ┌──────────────────┐        │
│  │ Prometheus  │  │  Grafana    │  │   ELK Stack      │        │
│  │  (Metrics)  │  │ (Dashboards)│  │ (Logging/Search) │        │
│  └─────────────┘  └─────────────┘  └──────────────────┘        │
└─────────────────────────────────────────────────────────────────┘
```

## Component Details

### 1. User Access Layer

- **AWS Application Load Balancer (ALB)**: Entry point for all traffic
  - SSL/TLS termination
  - Web Application Firewall (WAF)
  - Load balancing
  - Health probes

### 2. EKS Cluster (Kubernetes Orchestration)

- **Multiple Namespaces**:
  - `dev`: Development environment
  - `qa`: QA/Testing environment
  - `prod`: Production environment
- **Ingress Controller**: Routes traffic to appropriate services
- **Service Mesh (Future)**: Istio for advanced traffic management

### 3. Application Layer (Microservices)

#### Frontend Service

- **Technology**: React
- **Port**: 3000
- **Purpose**: User interface and customer experience
- **Communication**: Calls backend services via API Gateway

#### Auth Service

- **Technology**: Java Spring Boot
- **Port**: 8081
- **Database**: auth-db (AWS RDS PostgreSQL)
- **Responsibilities**:
  - User authentication
  - JWT token generation/validation
  - Role-based access control (RBAC)

#### Product Service

- **Technology**: Java Spring Boot
- **Port**: 8082
- **Database**: product-db (AWS RDS PostgreSQL)
- **Responsibilities**:
  - Product catalog management
  - Search and filtering functionality
  - Product details management
  - Category management

#### Cart Service

- **Technology**: Java Spring Boot
- **Port**: 8085
- **Database**: cart-db (AWS ElastiCache Redis)
- **Responsibilities**:
  - Shopping cart management
  - Cart items CRUD operations
  - Cart expiration handling
  - Temporary state management

#### Order Service

- **Technology**: Java Spring Boot
- **Port**: 8083
- **Database**: order-db (AWS RDS PostgreSQL)
- **Responsibilities**:
  - Order lifecycle management
  - Order creation from cart
  - Order status tracking
  - Order history retrieval

#### Payment Service

- **Technology**: Java Spring Boot
- **Port**: 8084
- **Database**: payment-db (AWS RDS PostgreSQL)
- **Responsibilities**:
  - Payment orchestration
  - Payment gateway integration (mock)
  - Webhook handling
  - Transaction management

#### Inventory Service

- **Technology**: Java Spring Boot
- **Port**: 8086
- **Database**: inventory-db (AWS RDS PostgreSQL)
- **Responsibilities**:
  - Real-time stock management
  - Inventory reservation
  - Inventory commit/rollback
  - Stock consistency enforcement

## ⚙️ Non-Functional Requirements

### 1. Consistency

| Aspect        | Requirement          | Implementation                        |
| ------------- | -------------------- | ------------------------------------- |
| **Inventory** | Strong consistency   | Optimistic locking with @Version      |
| **Payments**  | Strong consistency   | Idempotent API calls, transaction IDs |
| **Orders**    | Eventual consistency | Event-driven updates                  |
| **Cart**      | Eventual consistency | Redis with TTL                        |

### 2. Scalability

| Component          | Strategy                                |
| ------------------ | --------------------------------------- |
| **Stateless APIs** | Horizontal pod autoscaling (HPA)        |
| **Database**       | Read replicas for read-heavy operations |
| **Cache**          | Redis for cart and session data         |
| **Load Balancing** | AWS ALB with round-robin                |

### 3. Fault Tolerance

| Mechanism                     | Implementation                              |
| ----------------------------- | ------------------------------------------- |
| **Retry Logic**               | Exponential backoff for inter-service calls |
| **Circuit Breaker**           | Resilience4j for service communication      |
| **Idempotency**               | Unique request IDs for all operations       |
| **Compensating Transactions** | Inventory release on payment failure        |
| **Health Checks**             | Kubernetes liveness and readiness probes    |

### 4. Security

| Layer               | Implementation                           |
| ------------------- | ---------------------------------------- |
| **Authentication**  | JWT with RS256 signing                   |
| **Authorization**   | Role-based access control (RBAC)         |
| **API Security**    | Rate limiting, input validation          |
| **Data Encryption** | TLS in transit, encryption at rest (RDS) |
| **Secrets**         | AWS Secrets Manager integration          |

### 5. Performance

| Target                | Metric                           |
| --------------------- | -------------------------------- |
| **API Response Time** | P95 < 200ms                      |
| **Database Queries**  | < 50ms for simple queries        |
| **Order Creation**    | < 1 second end-to-end            |
| **Cart Operations**   | < 100ms (Redis)                  |
| **Throughput**        | 1000 requests/second per service |

### 6. Observability

| Component   | Tool                                        |
| ----------- | ------------------------------------------- |
| **Metrics** | Prometheus + Grafana                        |
| **Logging** | ELK Stack (Elasticsearch, Logstash, Kibana) |
| **Tracing** | Distributed tracing (future: Jaeger)        |
| **Alerts**  | Prometheus AlertManager                     |

## 🛠️ Technology Stack (Locked)

### Backend

| Technology          | Version | Purpose               |
| ------------------- | ------- | --------------------- |
| **Java**            | 17 LTS  | Programming language  |
| **Spring Boot**     | 3.2+    | Framework             |
| **Spring Data JPA** | Latest  | ORM / Database access |
| **Hibernate**       | Latest  | JPA implementation    |
| **Maven**           | 3.9+    | Build tool            |
| **Lombok**          | Latest  | Boilerplate reduction |

### Databases

| Technology          | Use Case                              |
| ------------------- | ------------------------------------- |
| **PostgreSQL 15**   | Primary database for all services     |
| **AWS RDS**         | Managed PostgreSQL hosting            |
| **Redis**           | Cart service (session/temporary data) |
| **AWS ElastiCache** | Managed Redis hosting                 |

### Communication

| Technology         | Purpose                                 |
| ------------------ | --------------------------------------- |
| **REST APIs**      | Synchronous inter-service communication |
| **Spring WebFlux** | Reactive programming (optional)         |
| **OpenAPI 3.0**    | API documentation                       |

### Security

| Technology              | Purpose                        |
| ----------------------- | ------------------------------ |
| **Spring Security**     | Authentication & authorization |
| **JWT (JJWT)**          | Token-based authentication     |
| **BCrypt**              | Password hashing               |
| **AWS Secrets Manager** | Secrets management             |

### Testing

| Technology         | Purpose                                 |
| ------------------ | --------------------------------------- |
| **JUnit 5**        | Unit testing                            |
| **Mockito**        | Mocking framework                       |
| **TestContainers** | Integration testing with real databases |
| **RestAssured**    | API testing                             |

### DevOps

| Technology    | Purpose                       |
| ------------- | ----------------------------- |
| **Docker**    | Containerization              |
| **AWS EKS**   | Kubernetes orchestration      |
| **Helm**      | Kubernetes package management |
| **Terraform** | Infrastructure as code        |
| **Jenkins**   | CI/CD pipelines               |

### 4. Data Layer

- **AWS RDS PostgreSQL**: One database per microservice (Auth, Product, Order, Payment, Inventory)
- **Isolation**: Each service has its own schema
- **Backups**: Automated backup strategy
- **Security**: Private endpoints, encryption at rest

### 5. Observability Stack

#### Prometheus

- **Purpose**: Metrics collection and storage
- **Metrics Collected**:
  - Application metrics (custom counters, gauges)
  - System metrics (CPU, memory, disk)
  - Kubernetes metrics (pod health, resource usage)

#### Grafana

- **Purpose**: Visualization and dashboards
- **Dashboards**:
  - Application performance
  - Infrastructure health
  - Business metrics (orders, revenue)
  - Custom alerts

#### ELK Stack (Elasticsearch, Logstash, Kibana)

- **Purpose**: Centralized logging
- **Components**:
  - **Elasticsearch**: Log storage and indexing
  - **Logstash**: Log aggregation and processing
  - **Kibana**: Log visualization and search

## Traffic Flow

### User Request Flow

```
User Browser
  ↓ (HTTPS Request)
AWS Application Load Balancer (ALB)
  ↓ (SSL Termination)
Kubernetes Ingress Controller
  ↓ (Route to Service)
Frontend Pod
  ↓ (API Call with JWT)
Auth Service (Token Validation)
  ↓ (Authorized Request)
Product/Order/Payment Service
  ↓ (Database Query)
AWS RDS PostgreSQL
  ↓ (Response)
Back to User
```

### Order Creation Flow

```
User places order on Frontend
  ↓
Frontend → Auth Service (Validate User)
  ↓
Frontend → Product Service (Check Inventory)
  ↓
Frontend → Order Service (Create Order)
  ↓
Order Service → Payment Service (Process Payment)
  ↓
Payment Service returns confirmation
  ↓
Order Service updates order status
  ↓
Response sent back to Frontend
```

## Security Architecture

### Network Security

- **Network Policies**: Restrict inter-pod communication
- **Private Endpoints**: Databases not exposed to internet
- **Azure Key Vault**: Secrets management
- **SSL/TLS**: End-to-end encryption

### Application Security

- **JWT Authentication**: Stateless authentication
- **RBAC**: Role-based access control
- **API Rate Limiting**: Prevent abuse
- **Input Validation**: Protect against injection attacks

## Scalability Strategy

### Horizontal Pod Autoscaler (HPA)

- Auto-scale based on CPU/Memory usage
- Custom metrics scaling (e.g., request rate)

### Cluster Autoscaler

- Add/remove nodes based on resource demands

### Database Scaling

- Read replicas for read-heavy operations
- Connection pooling
- Query optimization

## High Availability

- **Multi-zone deployment**: Pods distributed across availability zones
- **Replica sets**: Minimum 3 replicas in production
- **Health checks**: Liveness and readiness probes
- **Circuit breakers**: Prevent cascading failures
- **Database backups**: Automated backup and point-in-time restore

## Disaster Recovery

- **RTO (Recovery Time Objective)**: 1 hour
- **RPO (Recovery Point Objective)**: 15 minutes
- **Backup Strategy**: Hourly database backups
- **Geo-redundancy**: Cross-region replication for critical data

## Interview Key Points

> **Q: Why Azure AKS?**
>
> - Managed Kubernetes service
> - Azure integration (SQL, Key Vault, Monitor)
> - Auto-scaling capabilities
> - Enterprise-grade security

> **Q: How do services communicate?**
>
> - REST APIs over HTTP/HTTPS
> - Service discovery via Kubernetes DNS
> - API Gateway pattern for external access

> **Q: What happens if a service fails?**
>
> - Kubernetes automatically restarts failed pods
> - Health checks detect unhealthy instances
> - Load balancer routes traffic to healthy pods
> - Circuit breaker prevents cascading failures

> **Q: How do you monitor the system?**
>
> - Prometheus collects metrics
> - Grafana visualizes metrics
> - ELK stack for centralized logging
> - Alerts configured for critical issues

> **Q: How is data secured?**
>
> - Encryption at rest (Azure SQL)
> - Encryption in transit (TLS)
> - Secrets in Azure Key Vault
> - Network isolation via private endpoints
