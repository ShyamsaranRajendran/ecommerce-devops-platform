# API Contracts

## Overview

This document defines the REST API contracts for all microservices in the e-commerce platform. All APIs use JSON for request/response payloads and follow RESTful principles.

## Common Headers

All authenticated endpoints require:

```http
Authorization: Bearer <JWT_TOKEN>
Content-Type: application/json
```

## Common Response Codes

| Code | Meaning               | Usage                               |
| ---- | --------------------- | ----------------------------------- |
| 200  | OK                    | Successful GET/PUT/PATCH            |
| 201  | Created               | Successful POST                     |
| 204  | No Content            | Successful DELETE                   |
| 400  | Bad Request           | Invalid input                       |
| 401  | Unauthorized          | Missing/invalid token               |
| 403  | Forbidden             | Insufficient permissions            |
| 404  | Not Found             | Resource not found                  |
| 409  | Conflict              | Resource conflict (e.g., duplicate) |
| 500  | Internal Server Error | Server error                        |

---

## 🔐 Auth Service (Port 8081)

### 1. Register User

**Endpoint:** `POST /auth/register`

**Request:**

```json
{
  "email": "user@example.com",
  "password": "SecurePass123!",
  "confirmPassword": "SecurePass123!"
}
```

**Response:** `201 Created`

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "email": "user@example.com",
  "role": "CUSTOMER",
  "createdAt": "2025-12-31T10:30:00Z"
}
```

**Validation:**

- Email must be valid format
- Password min 8 characters, must contain uppercase, lowercase, number, special char
- Passwords must match

### 2. Login

**Endpoint:** `POST /auth/login`

**Request:**

```json
{
  "email": "user@example.com",
  "password": "SecurePass123!"
}
```

**Response:** `200 OK`

```json
{
  "accessToken": "eyJhbGciOiJSUzI1NiIs...",
  "tokenType": "Bearer",
  "expiresIn": 3600,
  "user": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "email": "user@example.com",
    "role": "CUSTOMER"
  }
}
```

### 3. Validate Token

**Endpoint:** `GET /auth/validate`

**Headers:** `Authorization: Bearer <token>`

**Response:** `200 OK`

```json
{
  "valid": true,
  "userId": "550e8400-e29b-41d4-a716-446655440000",
  "email": "user@example.com",
  "role": "CUSTOMER"
}
```

### 4. Get User Profile

**Endpoint:** `GET /auth/profile`

**Headers:** `Authorization: Bearer <token>`

**Response:** `200 OK`

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "email": "user@example.com",
  "role": "CUSTOMER",
  "createdAt": "2025-12-31T10:30:00Z"
}
```

---

## 📦 Product Service (Port 8082)

### 1. Get All Products

**Endpoint:** `GET /products`

**Query Parameters:**

- `page` (optional, default: 0)
- `size` (optional, default: 20)
- `category` (optional)
- `search` (optional)

**Response:** `200 OK`

```json
{
  "content": [
    {
      "id": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
      "name": "Laptop Pro 15",
      "description": "High-performance laptop",
      "price": 1299.99,
      "category": "Electronics",
      "imageUrl": "https://cdn.example.com/laptop.jpg",
      "active": true,
      "createdAt": "2025-12-30T08:00:00Z"
    }
  ],
  "pageable": {
    "pageNumber": 0,
    "pageSize": 20,
    "totalElements": 50,
    "totalPages": 3
  }
}
```

### 2. Get Product by ID

**Endpoint:** `GET /products/{id}`

**Response:** `200 OK`

```json
{
  "id": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
  "name": "Laptop Pro 15",
  "description": "High-performance laptop with latest specs",
  "price": 1299.99,
  "category": "Electronics",
  "imageUrl": "https://cdn.example.com/laptop.jpg",
  "active": true,
  "createdAt": "2025-12-30T08:00:00Z"
}
```

### 3. Create Product (Admin Only)

**Endpoint:** `POST /products`

**Headers:** `Authorization: Bearer <admin_token>`

**Request:**

```json
{
  "name": "Wireless Mouse",
  "description": "Ergonomic wireless mouse",
  "price": 29.99,
  "category": "Accessories",
  "imageUrl": "https://cdn.example.com/mouse.jpg"
}
```

**Response:** `201 Created`

```json
{
  "id": "8d0e7890-8536-51ef-b827-557766551111",
  "name": "Wireless Mouse",
  "description": "Ergonomic wireless mouse",
  "price": 29.99,
  "category": "Accessories",
  "imageUrl": "https://cdn.example.com/mouse.jpg",
  "active": true,
  "createdAt": "2025-12-31T12:00:00Z"
}
```

### 4. Update Product (Admin Only)

**Endpoint:** `PUT /products/{id}`

**Headers:** `Authorization: Bearer <admin_token>`

**Request:** (Same as Create)

**Response:** `200 OK`

### 5. Delete Product (Admin Only)

**Endpoint:** `DELETE /products/{id}`

**Headers:** `Authorization: Bearer <admin_token>`

**Response:** `204 No Content`

---

## 🛒 Cart Service (Port 8085)

### 1. Get Cart

**Endpoint:** `GET /cart`

**Headers:** `Authorization: Bearer <token>`

**Response:** `200 OK`

```json
{
  "id": "cart-550e8400-e29b-41d4-a716-446655440000",
  "userId": "550e8400-e29b-41d4-a716-446655440000",
  "items": [
    {
      "productId": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
      "productName": "Laptop Pro 15",
      "price": 1299.99,
      "quantity": 1
    }
  ],
  "totalAmount": 1299.99,
  "itemCount": 1,
  "expiresAt": "2025-12-31T13:30:00Z"
}
```

### 2. Add Item to Cart

**Endpoint:** `POST /cart/items`

**Headers:** `Authorization: Bearer <token>`

**Request:**

```json
{
  "productId": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
  "quantity": 2
}
```

**Response:** `200 OK`

```json
{
  "id": "cart-550e8400-e29b-41d4-a716-446655440000",
  "userId": "550e8400-e29b-41d4-a716-446655440000",
  "items": [
    {
      "productId": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
      "productName": "Laptop Pro 15",
      "price": 1299.99,
      "quantity": 2
    }
  ],
  "totalAmount": 2599.98,
  "itemCount": 2
}
```

### 3. Update Cart Item

**Endpoint:** `PUT /cart/items/{productId}`

**Headers:** `Authorization: Bearer <token>`

**Request:**

```json
{
  "quantity": 3
}
```

**Response:** `200 OK` (Same structure as Get Cart)

### 4. Remove Item from Cart

**Endpoint:** `DELETE /cart/items/{productId}`

**Headers:** `Authorization: Bearer <token>`

**Response:** `204 No Content`

### 5. Clear Cart

**Endpoint:** `DELETE /cart`

**Headers:** `Authorization: Bearer <token>`

**Response:** `204 No Content`

---

## 📋 Order Service (Port 8083)

### 1. Create Order from Cart

**Endpoint:** `POST /orders`

**Headers:** `Authorization: Bearer <token>`

**Request:**

```json
{
  "shippingAddress": {
    "street": "123 Main St",
    "city": "San Francisco",
    "state": "CA",
    "zipCode": "94102",
    "country": "USA"
  }
}
```

**Response:** `201 Created`

```json
{
  "id": "order-9f1e7780-9647-62fg-c938-668877662222",
  "userId": "550e8400-e29b-41d4-a716-446655440000",
  "status": "CREATED",
  "items": [
    {
      "productId": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
      "productName": "Laptop Pro 15",
      "price": 1299.99,
      "quantity": 1
    }
  ],
  "totalAmount": 1299.99,
  "shippingAddress": {
    "street": "123 Main St",
    "city": "San Francisco",
    "state": "CA",
    "zipCode": "94102",
    "country": "USA"
  },
  "createdAt": "2025-12-31T14:00:00Z"
}
```

### 2. Get Order by ID

**Endpoint:** `GET /orders/{id}`

**Headers:** `Authorization: Bearer <token>`

**Response:** `200 OK`

```json
{
  "id": "order-9f1e7780-9647-62fg-c938-668877662222",
  "userId": "550e8400-e29b-41d4-a716-446655440000",
  "status": "PAID",
  "items": [
    {
      "productId": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
      "productName": "Laptop Pro 15",
      "price": 1299.99,
      "quantity": 1
    }
  ],
  "totalAmount": 1299.99,
  "paymentId": "pay-1a2b3c4d-5e6f-7g8h-9i0j-1k2l3m4n5o6p",
  "shippingAddress": {
    "street": "123 Main St",
    "city": "San Francisco",
    "state": "CA",
    "zipCode": "94102",
    "country": "USA"
  },
  "createdAt": "2025-12-31T14:00:00Z",
  "updatedAt": "2025-12-31T14:05:00Z"
}
```

### 3. Get Order History

**Endpoint:** `GET /orders/history`

**Headers:** `Authorization: Bearer <token>`

**Query Parameters:**

- `page` (optional, default: 0)
- `size` (optional, default: 10)

**Response:** `200 OK`

```json
{
  "content": [
    {
      "id": "order-9f1e7780-9647-62fg-c938-668877662222",
      "status": "PAID",
      "totalAmount": 1299.99,
      "itemCount": 1,
      "createdAt": "2025-12-31T14:00:00Z"
    }
  ],
  "pageable": {
    "pageNumber": 0,
    "pageSize": 10,
    "totalElements": 5,
    "totalPages": 1
  }
}
```

### 4. Cancel Order

**Endpoint:** `POST /orders/{id}/cancel`

**Headers:** `Authorization: Bearer <token>`

**Response:** `200 OK`

```json
{
  "id": "order-9f1e7780-9647-62fg-c938-668877662222",
  "status": "CANCELLED",
  "message": "Order cancelled successfully. Inventory has been released."
}
```

**Note:** Can only cancel orders with status CREATED (before payment)

---

## 💳 Payment Service (Port 8084)

### 1. Initiate Payment

**Endpoint:** `POST /payments/{orderId}`

**Headers:** `Authorization: Bearer <token>`

**Request:**

```json
{
  "provider": "MOCK",
  "paymentMethod": "CREDIT_CARD"
}
```

**Response:** `201 Created`

```json
{
  "id": "pay-1a2b3c4d-5e6f-7g8h-9i0j-1k2l3m4n5o6p",
  "orderId": "order-9f1e7780-9647-62fg-c938-668877662222",
  "status": "PENDING",
  "amount": 1299.99,
  "provider": "MOCK",
  "paymentUrl": "https://mock-payment.example.com/pay/1a2b3c4d",
  "createdAt": "2025-12-31T14:02:00Z"
}
```

### 2. Get Payment Status

**Endpoint:** `GET /payments/{paymentId}`

**Headers:** `Authorization: Bearer <token>`

**Response:** `200 OK`

```json
{
  "id": "pay-1a2b3c4d-5e6f-7g8h-9i0j-1k2l3m4n5o6p",
  "orderId": "order-9f1e7780-9647-62fg-c938-668877662222",
  "status": "SUCCESS",
  "amount": 1299.99,
  "provider": "MOCK",
  "transactionId": "txn-987654321",
  "createdAt": "2025-12-31T14:02:00Z",
  "updatedAt": "2025-12-31T14:05:00Z"
}
```

### 3. Payment Webhook

**Endpoint:** `POST /payments/webhook`

**Headers:** `X-Webhook-Signature: <signature>`

**Request:**

```json
{
  "paymentId": "pay-1a2b3c4d-5e6f-7g8h-9i0j-1k2l3m4n5o6p",
  "orderId": "order-9f1e7780-9647-62fg-c938-668877662222",
  "status": "SUCCESS",
  "transactionId": "txn-987654321",
  "timestamp": "2025-12-31T14:05:00Z"
}
```

**Response:** `200 OK`

```json
{
  "received": true,
  "processed": true
}
```

**Note:** This endpoint is idempotent and uses transaction ID for deduplication

---

## 📊 Inventory Service (Port 8086)

### 1. Check Stock Availability

**Endpoint:** `GET /inventory/{productId}`

**Response:** `200 OK`

```json
{
  "productId": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
  "availableQuantity": 50,
  "reservedQuantity": 5,
  "totalQuantity": 55,
  "inStock": true
}
```

### 2. Reserve Inventory

**Endpoint:** `POST /inventory/reserve`

**Headers:** Internal service-to-service call

**Request:**

```json
{
  "orderId": "order-9f1e7780-9647-62fg-c938-668877662222",
  "items": [
    {
      "productId": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
      "quantity": 1
    }
  ]
}
```

**Response:** `200 OK`

```json
{
  "orderId": "order-9f1e7780-9647-62fg-c938-668877662222",
  "reserved": true,
  "reservationId": "res-3c4d5e6f-7g8h-9i0j-1k2l-3m4n5o6p7q8r"
}
```

**Response:** `409 Conflict` (Insufficient stock)

```json
{
  "error": "INSUFFICIENT_STOCK",
  "message": "Insufficient stock for product: Laptop Pro 15",
  "availableQuantity": 0,
  "requestedQuantity": 1
}
```

### 3. Commit Reservation

**Endpoint:** `POST /inventory/commit`

**Headers:** Internal service-to-service call

**Request:**

```json
{
  "orderId": "order-9f1e7780-9647-62fg-c938-668877662222",
  "reservationId": "res-3c4d5e6f-7g8h-9i0j-1k2l-3m4n5o6p7q8r"
}
```

**Response:** `200 OK`

```json
{
  "committed": true,
  "message": "Inventory committed successfully"
}
```

### 4. Release Reservation

**Endpoint:** `POST /inventory/release`

**Headers:** Internal service-to-service call

**Request:**

```json
{
  "orderId": "order-9f1e7780-9647-62fg-c938-668877662222",
  "reservationId": "res-3c4d5e6f-7g8h-9i0j-1k2l-3m4n5o6p7q8r"
}
```

**Response:** `200 OK`

```json
{
  "released": true,
  "message": "Inventory released successfully"
}
```

### 5. Update Stock (Admin Only)

**Endpoint:** `PUT /inventory/{productId}`

**Headers:** `Authorization: Bearer <admin_token>`

**Request:**

```json
{
  "quantity": 100,
  "operation": "ADD"
}
```

**Response:** `200 OK`

```json
{
  "productId": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
  "availableQuantity": 150,
  "reservedQuantity": 5,
  "totalQuantity": 155
}
```

---

## 🔄 Inter-Service Communication

### Service Dependencies

```
Cart Service
  ↓ (calls)
Product Service (get product details)

Order Service
  ↓ (calls)
Cart Service (get cart items)
Inventory Service (reserve stock)
Payment Service (initiate payment)

Payment Service
  ↓ (calls)
Order Service (update order status)
Inventory Service (commit/release)
```

### Authentication Between Services

Internal service-to-service calls use:

- Service account tokens OR
- API keys stored in AWS Secrets Manager

---

## 📝 Error Response Format

All errors follow a consistent format:

```json
{
  "timestamp": "2025-12-31T14:30:00Z",
  "status": 400,
  "error": "Bad Request",
  "message": "Validation failed for field 'email'",
  "path": "/auth/register",
  "errors": [
    {
      "field": "email",
      "message": "Email format is invalid"
    }
  ]
}
```

---

## ✅ API Design Principles

1. **RESTful**: Follow REST conventions for HTTP methods and status codes
2. **Consistent**: Same patterns across all services
3. **Versioned**: APIs can be versioned (e.g., `/v1/products`)
4. **Documented**: OpenAPI 3.0 specifications available
5. **Secure**: All authenticated endpoints require valid JWT
6. **Idempotent**: POST/PUT operations use idempotency keys where needed
7. **Paginated**: List endpoints support pagination
8. **Validated**: Input validation at API layer

---

## 🎯 Interview Key Points

> **Q: How do you ensure API consistency across services?**
>
> - Shared API design guidelines
> - Common error response format
> - Consistent naming conventions
> - OpenAPI specifications for all services

> **Q: How do you handle versioning?**
>
> - URL path versioning (e.g., `/v1/`, `/v2/`)
> - Backward compatibility maintained
> - Deprecation notices for old versions

> **Q: What about API security?**
>
> - JWT authentication for user operations
> - Service-to-service authentication via API keys
> - Rate limiting on all public endpoints
> - Input validation and sanitization

> **Q: How do you document APIs?**
>
> - OpenAPI 3.0 specifications
> - Swagger UI for interactive docs
> - Postman collections for testing
> - This document for contract definitions
