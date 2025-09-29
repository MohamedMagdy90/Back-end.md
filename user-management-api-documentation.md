# User Management API Documentation

## Table of Contents
1. [Overview](#overview)
2. [Authentication](#authentication)
3. [Base URL](#base-url)
4. [Data Models](#data-models)
5. [API Endpoints](#api-endpoints)
6. [Error Handling](#error-handling)
7. [Security Considerations](#security-considerations)
8. [Rate Limiting](#rate-limiting)
9. [Audit Logging](#audit-logging)
10. [Database Schema](#database-schema)

---

## Overview

The User Management API provides comprehensive functionality for managing users in the Solution ERP system. This includes user creation, authentication, role management, permission control, and email configuration.

### Key Features
- User CRUD operations (Create, Read, Update, Delete)
- Role-based access control (RBAC)
- Email configuration per user
- Password management and validation
- Activity tracking and audit logging
- SMTP configuration for email delivery
- User profile photo management

### API Version
**Current Version:** v1
**Base Path:** `/api/v1`

---

## Authentication

### Authentication Method
The API uses JWT (JSON Web Token) for authentication.

### Authentication Headers
```http
Authorization: Bearer <jwt_token>
Content-Type: application/json
X-API-Version: v1
X-Request-ID: <unique_request_id>
```

### Token Structure
```json
{
  "header": {
    "alg": "HS256",
    "typ": "JWT"
  },
  "payload": {
    "userId": "user_1234567890",
    "email": "user@company.com",
    "roleId": "role_admin",
    "permissions": {},
    "iat": 1703123456,
    "exp": 1703209856,
    "iss": "solution-erp",
    "sub": "user_1234567890"
  }
}
```

---

## Base URL

### Production
```
https://api.solution-erp.com/api/v1
```

### Staging
```
https://api-staging.solution-erp.com/api/v1
```

### Development
```
http://localhost:3000/api/v1
```

---

## Data Models

### User Object

```typescript
interface User {
  id: string;                    // Unique identifier (e.g., "user_1234567890")
  fullName: string;               // User's full name
  email: string;                  // Email address (unique)
  password?: string;              // Hashed password (never returned in responses)
  roleId: string;                 // Role identifier
  phone?: string;                 // Phone number (optional)
  photo?: string;                 // Base64 encoded image or URL (optional)
  status: "active" | "inactive";  // Account status
  joinDate: string;               // ISO 8601 date string
  lastLogin?: string;             // ISO 8601 date string (optional)
  createdAt: string;              // ISO 8601 date string
  updatedAt: string;              // ISO 8601 date string
  emailSettings?: EmailSettings;  // Email configuration (optional)
  metadata?: UserMetadata;        // Additional metadata
}
```

### EmailSettings Object

```typescript
interface EmailSettings {
  enabled: boolean;               // Whether email is enabled for this user
  smtp: {
    host: string;                 // SMTP server hostname
    port: number;                 // SMTP port (usually 587, 465, or 25)
    username: string;             // SMTP username
    password: string;             // Encrypted SMTP password
    encryption: "TLS" | "SSL" | "None"; // Encryption method
  };
  alwaysUseSender: boolean;       // Always use this email as sender
  fromName?: string;              // Display name for emails
  replyTo?: string;               // Reply-to email address
  signature?: string;             // HTML email signature
  footer?: string;                // HTML email footer
}
```

### UserRole Object

```typescript
interface UserRole {
  id: string;                     // Role identifier
  name: string;                   // Role display name
  description: string;            // Role description
  permissions: Record<string, Permission>; // Module permissions
  active: boolean;                // Whether role is active
  createdAt: string;              // ISO 8601 date string
  updatedAt: string;              // ISO 8601 date string
  discountLimits?: DiscountLimits; // Optional discount limits
}
```

### Permission Object

```typescript
interface Permission {
  add: boolean;                   // Can create new records
  view: boolean;                  // Can view records
  delete: boolean;                // Can delete records
}
```

---

## API Endpoints

### 1. User Authentication

#### Login
**POST** `/auth/login`

##### Request Body
```json
{
  "email": "admin@company.com",
  "password": "SecurePass123!",
  "rememberMe": false,
  "companyId": "comp_123"
}
```

##### Response (200 OK)
```json
{
  "success": true,
  "data": {
    "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "refreshToken": "refresh_token_here",
    "expiresIn": 86400,
    "user": {
      "id": "user_1234567890",
      "fullName": "Admin User",
      "email": "admin@company.com",
      "roleId": "role_admin",
      "role": {
        "id": "role_admin",
        "name": "Administrator",
        "permissions": {}
      },
      "status": "active",
      "lastLogin": "2024-12-21T10:30:00Z"
    }
  }
}
```

#### Logout
**POST** `/auth/logout`

##### Request Headers
```http
Authorization: Bearer <jwt_token>
```

##### Response (200 OK)
```json
{
  "success": true,
  "message": "Successfully logged out"
}
```

### 2. User Management

#### Get All Users
**GET** `/users`

##### Query Parameters
```
?page=1
&limit=20
&sort=createdAt
&order=desc
&search=john
&status=active
&roleId=role_admin
&includeDeleted=false
```

##### Response (200 OK)
```json
{
  "success": true,
  "data": {
    "users": [
      {
        "id": "user_1234567890",
        "fullName": "John Doe",
        "email": "john.doe@company.com",
        "roleId": "role_manager",
        "phone": "+1234567890",
        "photo": "https://api.solution-erp.com/photos/user_1234567890.jpg",
        "status": "active",
        "joinDate": "2024-01-15T08:00:00Z",
        "lastLogin": "2024-12-21T09:15:00Z",
        "createdAt": "2024-01-15T08:00:00Z",
        "updatedAt": "2024-12-20T14:30:00Z"
      }
    ],
    "pagination": {
      "total": 150,
      "page": 1,
      "limit": 20,
      "totalPages": 8,
      "hasNext": true,
      "hasPrevious": false
    }
  }
}
```

#### Create User
**POST** `/users`

##### Request Body
```json
{
  "fullName": "Jane Smith",
  "email": "jane.smith@company.com",
  "password": "SecurePass456!",
  "roleId": "role_accountant",
  "phone": "+9876543210",
  "status": "active",
  "emailSettings": {
    "enabled": true,
    "smtp": {
      "host": "smtp.office365.com",
      "port": 587,
      "username": "jane.smith@company.com",
      "password": "smtp_password_here",
      "encryption": "TLS"
    },
    "alwaysUseSender": false,
    "fromName": "Jane Smith",
    "replyTo": "jane.smith@company.com",
    "signature": "<p>Best regards,<br/>Jane Smith<br/>Accountant</p>",
    "footer": "<p>This email is confidential</p>"
  }
}
```

##### Response (201 Created)
```json
{
  "success": true,
  "data": {
    "id": "user_9876543210",
    "fullName": "Jane Smith",
    "email": "jane.smith@company.com",
    "roleId": "role_accountant",
    "phone": "+9876543210",
    "status": "active",
    "joinDate": "2024-12-21T10:45:00Z",
    "createdAt": "2024-12-21T10:45:00Z",
    "updatedAt": "2024-12-21T10:45:00Z"
  },
  "message": "User created successfully"
}
```

---

## Error Handling

### Standard Error Response Format

```json
{
  "success": false,
  "error": {
    "code": "ERROR_CODE",
    "message": "Human-readable error message",
    "details": {
      "field": "Additional context"
    },
    "timestamp": "2024-12-21T15:00:00Z",
    "requestId": "req_abc123",
    "path": "/api/v1/users"
  }
}
```

### Common Error Codes

| Code | HTTP Status | Description |
|------|------------|-------------|
| `AUTH_REQUIRED` | 401 | Authentication required |
| `AUTH_FAILED` | 401 | Invalid credentials |
| `TOKEN_EXPIRED` | 401 | JWT token has expired |
| `TOKEN_INVALID` | 401 | Invalid JWT token |
| `PERMISSION_DENIED` | 403 | Insufficient permissions |
| `NOT_FOUND` | 404 | Resource not found |
| `VALIDATION_ERROR` | 400 | Request validation failed |
| `DUPLICATE_EMAIL` | 409 | Email already exists |
| `RATE_LIMIT_EXCEEDED` | 429 | Too many requests |
| `SERVER_ERROR` | 500 | Internal server error |

---

## Security Considerations

### Password Requirements

1. **Minimum Length**: 8 characters
2. **Complexity Requirements**:
   - At least one uppercase letter (A-Z)
   - At least one lowercase letter (a-z)
   - At least one number (0-9)
   - At least one special character (!@#$%^&*)
3. **Additional Rules**:
   - Cannot contain user's email or name
   - Cannot be a common password
   - Cannot be the same as last 5 passwords
   - Must be changed every 90 days (configurable)

### Password Hashing

- **Algorithm**: bcrypt with salt rounds of 12
- **Storage**: Only hashed passwords are stored
- **Comparison**: Constant-time comparison to prevent timing attacks

### Session Management

1. **Token Expiry**:
   - Access Token: 24 hours
   - Refresh Token: 7 days
   - Remember Me: 30 days

2. **Token Rotation**: New tokens issued on refresh
3. **Concurrent Sessions**: Maximum 3 active sessions per user
4. **Session Invalidation**: All sessions invalidated on password change

---

## Rate Limiting

### Default Limits

| Endpoint | Method | Limit | Window |
|----------|--------|-------|---------|
| `/auth/login` | POST | 5 requests | 15 minutes |
| `/auth/reset-password-request` | POST | 3 requests | 1 hour |
| `/users` | GET | 100 requests | 1 minute |
| `/users` | POST | 10 requests | 1 minute |
| `/users/{id}` | PUT | 20 requests | 1 minute |
| `/users/{id}` | DELETE | 5 requests | 1 minute |

### Rate Limit Headers

```http
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 1703209856
X-RateLimit-Reset-After: 45
```

---

## Database Schema

### Users Table

```sql
CREATE TABLE users (
    id VARCHAR(50) PRIMARY KEY,
    full_name VARCHAR(255) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    role_id VARCHAR(50) NOT NULL,
    phone VARCHAR(50),
    photo TEXT,
    status ENUM('active', 'inactive') DEFAULT 'active',
    join_date TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    last_login TIMESTAMP,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    deleted_at TIMESTAMP,
    FOREIGN KEY (role_id) REFERENCES roles(id),
    INDEX idx_email (email),
    INDEX idx_role_id (role_id),
    INDEX idx_status (status),
    INDEX idx_deleted_at (deleted_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### User Email Settings Table

```sql
CREATE TABLE user_email_settings (
    user_id VARCHAR(50) PRIMARY KEY,
    enabled BOOLEAN DEFAULT FALSE,
    smtp_host VARCHAR(255),
    smtp_port INT,
    smtp_username VARCHAR(255),
    smtp_password_encrypted TEXT,
    smtp_encryption ENUM('TLS', 'SSL', 'None') DEFAULT 'TLS',
    always_use_sender BOOLEAN DEFAULT FALSE,
    from_name VARCHAR(255),
    reply_to VARCHAR(255),
    signature TEXT,
    footer TEXT,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### Roles Table

```sql
CREATE TABLE roles (
    id VARCHAR(50) PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    description TEXT,
    permissions JSON NOT NULL,
    discount_limits JSON,
    active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_active (active)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

---

**Document Version:** 1.0.0
**Last Updated:** December 21, 2024
**Author:** Solution ERP Development Team
**Contact:** api-support@solution-erp.com

© 2024 Solution ERP. All rights reserved.