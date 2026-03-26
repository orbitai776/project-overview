
# Admin Service Backend Documentation

Part of the Admin Managed System — provides RESTful APIs for user management, partner management, system-wide statistics, and token billing displayed on the Admin Dashboard.

# Admin Service Backend Documentation

Part of the **Admin Managed System** — provides RESTful APIs for user management, partner management, system-wide statistics, and token billing displayed on the Admin Dashboard.

---

## Table of Contents

1. [Overview](#1-overview)
2. [Authentication](#2-authentication)
3. [API Endpoints](#3-api-endpoints)
4. [Response Format](#4-response-format)
5. [Error Codes](#5-error-codes)
6. [Module Structure](#6-module-structure)
7. [Environment Variables](#7-environment-variables)
8. [Test Cases](#8-test-cases)

---

## 1. Overview

### System Architecture

```text
Admin Browser
    └─► Gateway (Golang)
              ├── Firebase Auth Verify
              └── Admin Service (NestJS)
                      ├── Auth Module
                      ├── Dashboard Module
                      ├── Users Module
                      └── Partners Module

### Tech Stack

 Component       Technology               
 --------------  ------------------------ 
 Runtime         Node.js                  
 Framework       NestJS (TypeScript)      
 Database        PostgreSQL + Supabase    
 Authentication  Internal JWT via Gateway 
 Observability   OpenTelemetry + Grafana  

### Base URL

```text
adminapiv1
```

All endpoints return JSON. HTTP status codes follow REST conventions.

---

## 2. Authentication

The Admin Service does not handle Firebase Auth directly. The Gateway verifies the Firebase token and forwards an Internal JWT with `role=admin` to the Admin Service. The Admin Service only verifies this Internal JWT.

All endpoints (except login) require

```text
Authorization Bearer internal_jwt
```

 Case                          HTTP  
 ----------------------------  ----- 
 Missing or invalid JWT        `401` 
 Valid JWT but role ≠ `admin`  `403` 

---

## 3. API Endpoints

### 3.1 Auth Module (`auth`)
 `POST login` Authenticate and receive Admin JWT.
 `GET me` Get current admin profile.

### 3.2 Dashboard Module (`dashboard`)
Accepts query parameter `period` (Allowed `7d`, `30d`, `90d`, `1y`. Default `30d`).

 `GET overview` High-level summary of the entire system (users, partners, services).
 `GET users` User statistics with daily chart data.
 `GET partners` Partner statistics including top token consumers.
 `GET services` AI service statistics broken down by type.

### 3.3 Users Module (`users`)
 `GET ` Get paginated list of users. Accepts queries `page`, `limit`, `search`, `status`.
 `GET id` Get user details by UUID.
 `PATCH idstatus` Update user status (e.g., ban user).

### 3.4 Partners Module (`partners`)
 `GET ` Get paginated list of partners.
 `GET idorganizations` Get organizations managed by a specific partner.
 `GET idtokens` Get current token balance and history.
 `POST idtokensadjust` Manually credit or debit tokens for a partner (Topup).

---

## 4. Response Format

### Success

```json
{
  success true,
  data {}
}
```

### Error

```json
{
  success false,
  error {
    code INVALID_INPUT,
    message Invalid period value. Allowed 7d, 30d, 90d, 1y,
    details null
  }
}
```

---

## 5. Error Codes

 HTTP   Code              Description                        
 -----  ----------------  ---------------------------------- 
 `400`  `INVALID_INPUT`   `period` value not in allowed list  Pagination limit exceeded 
 `401`  `UNAUTHORIZED`    Missing, invalid, or expired JWT   
 `403`  `FORBIDDEN`       JWT valid but role is not `admin`  
 `404`  `USER_NOT_FOUND`  Resource UUID does not exist      
 `429`  `RATE_LIMIT_EXCEEDED`  Too many requests 
 `500`  `INTERNAL_ERROR`  Unexpected server error            

---

## 6. Module Structure

```text
src
└── modules
    ├── admin            # Auth & Core logic
    ├── dashboard        # Dashboard statistics
    ├── users            # User management
    └── partners         # Partner & Token management
```

---

## 7. Environment Variables

Copy `.env.example` to `.env` and fill in the values.

```bash
# App
NODE_ENV=development
PORT=3000
API_PREFIX=adminapiv1

# Internal JWT (forwarded from Gateway)
JWT_SECRET=your_secret_key_min_32_chars
JWT_EXPIRES_IN=3600

# Database
DATABASE_URL=postgresqluserpassword@host5432dbname

# OpenTelemetry
OTEL_EXPORTER_OTLP_ENDPOINT=httpyour-otel-collector4318
OTEL_SERVICE_NAME=admin-service
```

---

## 8. Test Cases

### 8.1 Auth
 TC-AUTH-001 Successful login with valid `role=admin` - HTTP `200`.
 TC-AUTH-002 Invalid Firebase token - HTTP `401` (`UNAUTHORIZED`).
 TC-AUTH-003 Valid token but `role=user` - HTTP `403` (`FORBIDDEN`).
 TC-AUTH-004 Rate limit exceeded - HTTP `429` (`RATE_LIMIT_EXCEEDED`).
 TC-AUTH-005 Get profile `authme` with valid JWT - HTTP `200`.

### 8.2 Dashboard
 TC-DASH-001 Get overview with default period - HTTP `200` (`data.period === 30d`).
 TC-DASH-002 Get overview with valid period `period=7d` - HTTP `200`.
 TC-DASH-003 Access without JWT - HTTP `401`.

### 8.3 Users Management
 TC-USER-001 Get paginated list - HTTP `200` (default `limit=20`).
 TC-USER-002 Filter by status `active` - HTTP `200`.
 TC-USER-003 Filter by status `banned` - HTTP `200`.
 TC-USER-004 Search by email - HTTP `200`.
 TC-USER-005 Exceed max limit (`limit=200`) - HTTP `400`.
 TC-USER-006 Get existing user details - HTTP `200`.
 TC-USER-007 Get non-existent user UUID - HTTP `404` (`USER_NOT_FOUND`).

### 8.4 Partners & Tokens Management
 TC-PARTNER-001 Get paginated list of partners - HTTP `200`.
 TC-ORG-001 Get organization list of valid partner - HTTP `200`.
 TC-TOKEN-001 Get token balance - HTTP `200`.
 TC-TOKEN-002 Credit tokens successfully - HTTP `200` (`new_balance === previous_balance + 50000`).

### 8.5 Security & Common
 TC-SEC-001 Access protected endpoint without JWT - HTTP `401`.
 TC-SEC-002 Access with `role=user` or `role=partner` - HTTP `403`.
```

