# Dashboard Module

Part of the **Admin Service** — provides RESTful APIs for system-wide statistics displayed on the Admin Dashboard.

---

## Table of Contents

1. [Overview](#1-overview)
2. [Authentication](#2-authentication)
3. [API Endpoints](#3-api-endpoints)
4. [Response Format](#4-response-format)
5. [Error Codes](#5-error-codes)
6. [Module Structure](#6-module-structure)
7. [Environment Variables](#7-environment-variables)
8. [Testcase](#8-Test-Cases)
---

## 1. Overview

### System Architecture

```
Admin Browser
    └─► Gateway (Golang)
              ├── Firebase Auth Verify
              └── Admin Service (NestJS)
                      └── Dashboard Module
                              ├── GET /dashboard/overview
                              ├── GET /dashboard/users
                              ├── GET /dashboard/partners
                              └── GET /dashboard/services
```

### Tech Stack

| Component      | Technology               |
| -------------- | ------------------------ |
| Runtime        | Nodejs                   |
| Framework      | NestJS (TypeScript)      |
| Database       | PostgreSQL + Supabase    |
| Authentication | Internal JWT via Gateway |
| Observability  | OpenTelemetry + Grafana  |

### Base URL

```
/admin/api/v1/dashboard
```

All endpoints return JSON. HTTP status codes follow REST conventions.

---

## 2. Authentication

Dashboard Module does **not** handle Firebase Auth directly. The Gateway verifies the Firebase token and forwards an **Internal JWT** with `role=admin` to Admin Service. Dashboard only verifies this Internal JWT.

All endpoints require:

```
Authorization: Bearer <internal_jwt>
```

| Case                         | HTTP  |
| ---------------------------- | ----- |
| Missing or invalid JWT       | `401` |
| Valid JWT but role ≠ `admin` | `403` |

---

## 3. API Endpoints

### Common Query Parameter

All four endpoints accept the same query param:

| Param    | Type   | Default | Allowed Values           |
| -------- | ------ | ------- | ------------------------ |
| `period` | string | `30d`   | `7d`, `30d`, `90d`, `1y` |

> Any value outside the allowed list returns `400 INVALID_INPUT`.

---

### 3.1 GET `/dashboard/overview`

Returns a high-level summary of the entire system — total users, partners, and AI services.

| Group      | Fields                                 |
| ---------- | -------------------------------------- |
| `users`    | total, new in period, active, growth % |
| `partners` | total, new in period, active           |
| `services` | total, active, total tokens used       |

---

### 3.2 GET `/dashboard/users`

Returns user statistics with daily chart data for the selected period.

| Field              | Description                                    |
| ------------------ | ---------------------------------------------- |
| `total_users`      | Total registered users                         |
| `new_users_today`  | Users registered today                         |
| `active_users_30d` | Users active in the last 30 days               |
| `banned_users`     | Currently banned users                         |
| `chart_data[]`     | Daily breakdown: `date`, `new_users`, `active` |

---

### 3.3 GET `/dashboard/partners`

Returns partner statistics including top token consumers.

| Field                      | Description                               |
| -------------------------- | ----------------------------------------- |
| `total_partners`           | Total partners                            |
| `active_partners`          | Currently active partners                 |
| `suspended_partners`       | Currently suspended partners              |
| `total_organizations`      | Total organizations across all partners   |
| `top_partners_by_tokens[]` | Top partners by token usage in the period |

---

### 3.4 GET `/dashboard/services`

Returns AI service statistics broken down by type, with daily token usage chart data.

| Field                   | Description                               |
| ----------------------- | ----------------------------------------- |
| `total_services`        | Total AI services                         |
| `active_services`       | Currently active services                 |
| `inactive_services`     | Currently inactive services               |
| `services_by_type`      | Count per service type                    |
| `total_tokens_used_all` | Total tokens consumed across all services |
| `chart_data[]`          | Daily breakdown: `date`, `tokens`         |

---

## 4. Response Format

### Success

```json
{
  "success": true,
  "data": {}
}
```

### Error

```json
{
  "success": false,
  "error": {
    "code": "INVALID_INPUT",
    "message": "Invalid period value. Allowed: 7d, 30d, 90d, 1y",
    "details": null
  }
}
```

---

## 5. Error Codes

| HTTP  | Code             | Description                        |
| ----- | ---------------- | ---------------------------------- |
| `400` | `INVALID_INPUT`  | `period` value not in allowed list |
| `401` | `UNAUTHORIZED`   | Missing, invalid, or expired JWT   |
| `403` | `FORBIDDEN`      | JWT valid but role is not `admin`  |
| `500` | `INTERNAL_ERROR` | Unexpected server error            |

---

## 6. Module Structure

```
src/
└── modules/
    └── dashboard/
        ├── dashboard.module.ts
        ├── dashboard.controller.ts   # Routes & Swagger decorators
        ├── dashboard.service.ts      # Business logic & DB queries
        └── dto/
            └── get-dashboard.dto.ts  # Query param validation
```

---

## 7. Environment Variables

Copy `.env.example` to `.env` and fill in the values.

```bash
# App
NODE_ENV=development
PORT=3000
API_PREFIX=admin/api/v1

# Internal JWT (forwarded from Gateway)
JWT_SECRET=your_secret_key_min_32_chars
JWT_EXPIRES_IN=3600

# Database
DATABASE_URL=postgresql://user:password@host:5432/dbname

# OpenTelemetry
OTEL_EXPORTER_OTLP_ENDPOINT=http://your-otel-collector:4318
OTEL_SERVICE_NAME=admin-service
```
## 8. Test Cases

---

### 8.1 Auth — POST `/auth/login`

#### TC-AUTH-001 — Đăng nhập thành công

- **Input:** `firebase_token` hợp lệ, tài khoản có `role=admin`
- **Expected HTTP:** `200`
- **Expected Response:**
  ```json
  { "success": true, "data": { "access_token": "<jwt>", "token_type": "Bearer" } }
  ```

#### TC-AUTH-002 — Firebase token không hợp lệ

- **Input:** `firebase_token` sai / hết hạn
- **Expected HTTP:** `401`
- **Expected Response:**
  ```json
  { "success": false, "error": { "code": "UNAUTHORIZED" } }
  ```

#### TC-AUTH-003 — Tài khoản không có quyền admin

- **Input:** `firebase_token` hợp lệ nhưng tài khoản là `role=user`
- **Expected HTTP:** `403`
- **Expected Response:**
  ```json
  { "success": false, "error": { "code": "FORBIDDEN" } }
  ```

#### TC-AUTH-004 — Vượt quá rate limit

- **Input:** Gửi request liên tiếp > ngưỡng rate limit (Redis)
- **Expected HTTP:** `429`
- **Expected Response:**
  ```json
  { "success": false, "error": { "code": "RATE_LIMIT_EXCEEDED" } }
  ```

---

### 8.2 Auth — GET `/auth/me`

#### TC-AUTH-005 — Lấy thông tin admin thành công

- **Input:** Header `Authorization: Bearer <valid_jwt>`
- **Expected HTTP:** `200`
- **Expected Response:** object chứa `id`, `email`, `role: "admin"`

#### TC-AUTH-006 — Không có token

- **Input:** Không có header `Authorization`
- **Expected HTTP:** `401`

---

### 8.3 Dashboard — GET `/dashboard/overview`

#### TC-DASH-001 — Lấy overview thành công với period mặc định

- **Input:** Không truyền `period`
- **Expected HTTP:** `200`
- **Expected:** `data.period === "30d"`, có đủ 3 key: `users`, `partners`, `services`

#### TC-DASH-002 — Lấy overview với period hợp lệ

- **Input:** `?period=7d`
- **Expected HTTP:** `200`
- **Expected:** `data.period === "7d"`

#### TC-DASH-003 — Không có JWT

- **Input:** Request không có Authorization header
- **Expected HTTP:** `401`

---

### 8.4 Users — GET `/users`

#### TC-USER-001 — Lấy danh sách thành công, pagination mặc định

- **Input:** Không có query param
- **Expected HTTP:** `200`
- **Expected:** `data.items` là array, `data.pagination.page === 1`, `limit === 20`

#### TC-USER-002 — Filter theo status `active`

- **Input:** `?status=active`
- **Expected HTTP:** `200`
- **Expected:** tất cả item trong `data.items` có `status === "active"`

#### TC-USER-003 — Filter theo status `banned`

- **Input:** `?status=banned`
- **Expected HTTP:** `200`
- **Expected:** tất cả item trong `data.items` có `status === "banned"`

#### TC-USER-004 — Search theo email

- **Input:** `?search=user@example.com`
- **Expected HTTP:** `200`
- **Expected:** `data.items` chỉ chứa các user có email khớp

#### TC-USER-005 — limit vượt quá max (100)

- **Input:** `?limit=200`
- **Expected HTTP:** `400`

---

### 8.5 Users — GET `/users/:id`

#### TC-USER-006 — Lấy chi tiết user tồn tại

- **Input:** `:id` là UUID hợp lệ, user tồn tại
- **Expected HTTP:** `200`
- **Expected:** response chứa đủ `id`, `name`, `email`, `status`

#### TC-USER-007 — User không tồn tại

- **Input:** `:id` là UUID không tồn tại trong DB
- **Expected HTTP:** `404`
- **Expected Response:**
  ```json
  { "success": false, "error": { "code": "USER_NOT_FOUND" } }
  ```
---

### 8.7 Partners — GET `/partners`

#### TC-PARTNER-001 — Lấy danh sách thành công

- **Input:** Không có query param
- **Expected HTTP:** `200`
- **Expected:** `data.items` là array, có `pagination`


### 8.8 Partners — GET `/partners/:id/organizations`

#### TC-ORG-001 — Lấy org list thành công

- **Input:** `:id` là partner hợp lệ có organizations
- **Expected HTTP:** `200`
- **Expected:** `data` là array, mỗi item có `org_name`, `org_code`, `status`

---

### 8.9 Tokens — GET `/partners/:id/tokens`

#### TC-TOKEN-001 — Lấy token balance thành công

- **Input:** `:id` hợp lệ
- **Expected HTTP:** `200`
- **Expected:** response có `balance`, `total_purchased`, `total_used`

### 8.10 Tokens — POST `/partners/:id/tokens/adjust`

#### TC-TOKEN-002 — Credit token thành công

- **Input:** `{ "type": "credit", "amount": 50000, "note": "Nap tien offline" }`
- **Expected HTTP:** `200`
- **Expected:** `data.new_balance === previous_balance + 50000`

---

### 8.11 Tổng hợp — Security & Common

#### TC-SEC-001 — Truy cập endpoint không có JWT

- **Input:** Bất kỳ endpoint protected, không có `Authorization` header
- **Expected HTTP:** `401`

#### TC-SEC-002 — JWT của role khác (user / partner)

- **Input:** Dùng JWT có `role=user` hoặc `role=partner`
- **Expected HTTP:** `403`
- **Expected Response:**
  ```json
  { "success": false, "error": { "code": "FORBIDDEN" } }
  ```

