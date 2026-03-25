# Admin Managed — Frontend Week 1 Documentation


## Table of Contents

1. [Tổng Quan Công Việc Tuần 1](#1-tổng-quan-công-việc-tuần-1)
2. [Sitemap & Màn Hình Phụ Trách](#2-sitemap--màn-hình-phụ-trách)
3. [Dựng Structure Hệ Thống](#3-dựng-structure-hệ-thống)
4. [Firebase Authentication](#4-firebase-authentication)
5. [Mockup Admin Login](#5-mockup-admin-login)
6. [Mockup Data Toàn Hệ Thống](#6-mockup-data-toàn-hệ-thống)
7. [Test Cases](#7-test-cases)

---

## 1. Tổng Quan Công Việc Tuần 1

| # | Hạng Mục | Mô Tả |
|---|---------|-------|
| 1 | Dựng structure hệ thống | Khởi tạo project Vue 3, cấu trúc thư mục, routing, state management |
| 2 | Integrate Firebase | Tích hợp Firebase Auth cho luồng đăng nhập Admin |
| 3 | Mockup Admin Login | Thiết kế và dựng giao diện trang đăng nhập |
| 4 | Mockup toàn bộ data | Dựng static mock data và giao diện cho tất cả màn hình |

---

## 2. Sitemap & Màn Hình Phụ Trách

```
Admin Managed
├── 3.1  Login Page
├── 3.2  Dashboard
│   ├── 3.2.1  Users Overview
│   ├── 3.2.2  Partners Overview
│   └── 3.2.3  Services Overview
└── 3.3  Managed
    ├── 3.3.1  Users
    └── 3.3.2  Partner
        ├── 3.3.2.1  Organization
        └── 3.3.2.2  Tokens
```

---

## 3. Dựng Structure Hệ Thống

### Mô Tả

Thiết lập toàn bộ skeleton của dự án Vue 3 — cấu trúc thư mục theo module, hệ thống routing có phân quyền, state management và layout chung cho Admin.

### Cấu Trúc Thư Mục

```
src/
  modules/
    admin/
      auth/          # Login, Firebase auth
      dashboard/     # Dashboard + Overview widgets
      users/         # Users management
      partners/      # Partner, Organization, Tokens
  router/            # Định nghĩa routes + route guard
  stores/            # Pinia stores
  plugins/           # Firebase, Axios config
  shared/
    layouts/         # AdminLayout (sidebar + topbar)
    components/      # Components dùng chung
  mocks/             # Static mock data
```

### Các Thành Phần Layout Chung

- **Sidebar** — navigation giữa các màn hình
- **Topbar** — thông tin user đang login, nút logout
- **Breadcrumb** — hiển thị vị trí hiện tại trong hệ thống
- **AdminLayout** — wrapper bao gồm sidebar + topbar + content area

---

## 4. Firebase Authentication

### Mô Tả

Tích hợp Firebase Authentication để xử lý đăng nhập cho Admin. Sau khi đăng nhập thành công, mọi request gửi lên Gateway đều kèm theo Firebase `idToken` trong header để Gateway xác thực.

### Vai Trò Từng Thành Phần

| Thành Phần | Vai Trò |
|------------|---------|
| **Frontend** | Gọi Firebase SDK, lấy `idToken`, đính kèm vào mọi request |
| **Gateway** | Nhận request, verify `idToken` với Firebase Admin SDK |
| **Backend** | Nhận thông tin user đã được Gateway xác thực |
| **Firebase** | Identity provider, cấp và quản lý JWT token |


### Luồng Đăng Nhập

```
User nhập email / password
        │
        ▼
[Frontend] Gọi Firebase signIn
        │
        ▼
[Firebase] Xác thực thông tin
        │
   ┌────┴────┐
Thất bại   Thành công
   │           │
   ▼           ▼
Hiển thị   Nhận về idToken
lỗi        + thông tin user
                │
                ▼
        Lưu user vào store
                │
                ▼
        Redirect → /dashboard
```

### Luồng Gọi API Sau Khi Login

```
[Frontend] Cần gọi API
        │
        ▼
Lấy idToken từ Firebase   ← tự động refresh nếu gần hết hạn
        │
        ▼
Gắn vào header:
Authorization: Bearer <idToken>
        │
        ▼
[Gateway] Verify idToken
        │
   ┌────┴────┐
Invalid    Valid
   │           │
   ▼           ▼
401        Decode payload
Unauthorized   → Forward → Backend
```

### Luồng Refresh Token

Firebase SDK tự động refresh `idToken` trước khi hết hạn. Frontend không cần xử lý thủ công — chỉ cần luôn lấy token mới từ SDK thay vì lưu tĩnh.

### Thông Tin User Object Trả Về Từ Firebase

Sau khi login thành công, Firebase trả về object với các field thực tế sau (đã xác nhận qua console):

| Field | Mô Tả | Ví Dụ |
|-------|-------|-------|
| `uid` | Firebase User ID (định danh duy nhất) | `"4d0m4UBBMVZADREEnIZdWZC2a2"` |
| `email` | Email của user | `"tranthanh00001992@gmail.com"` |
| `emailVerified` | Email đã xác minh chưa | `true` |
| `displayName` | Tên hiển thị | `"Thanh Tran"` |
| `photoURL` | Ảnh đại diện (từ Google) | URL ảnh Google profile |
| `providerId` | Provider đăng nhập | `"google.com"` |
| `isAnonymous` | Có phải anonymous không | `false` |

**stsTokenManager** (quản lý token nội bộ):

| Field | Mô Tả |
|-------|-------|
| `accessToken` | Firebase `idToken` — đính vào header mỗi request |
| `refreshToken` | Dùng để refresh `accessToken` khi hết hạn |
| `expirationTime` | Unix timestamp thời điểm hết hạn (1 giờ) |

### Thông Tin JWT Payload (Sau Khi Gateway Decode)

| Field | Mô Tả |
|-------|-------|
| `uid` | Firebase User ID |
| `email` | Email của user |
| `email_verified` | Email đã xác minh chưa |
| `name` | Display name của user |
| `picture` | URL ảnh đại diện |
| `iat` | Thời điểm cấp token |
| `exp` | Thời điểm hết hạn (1 giờ sau `iat`) |

### Provider Đăng Nhập

Dự án hiện đang dùng **Google Sign-In** (`providerId: 'google.com'`). Cần xác nhận với team có bổ sung thêm Email/Password hay không.

> **Lưu ý:** `operationType: 'signIn'` trả về trong response — có thể dùng để phân biệt các loại auth operation khi cần debug.

### Environment Variables


```
VITE_FIREBASE_API_KEY
VITE_FIREBASE_AUTH_DOMAIN
VITE_FIREBASE_PROJECT_ID
VITE_FIREBASE_STORAGE_BUCKET
VITE_FIREBASE_MESSAGING_SENDER_ID
VITE_FIREBASE_APP_ID
VITE_API_BASE_URL
```

### Firebase Error Mapping

| Firebase Error Code | Thông Báo Hiển Thị Cho User |
|--------------------|-----------------------------|
| `auth/user-not-found` | Email không tồn tại trong hệ thống. |
| `auth/wrong-password` | Sai email hoặc mật khẩu. |
| `auth/invalid-email` | Email không đúng định dạng. |
| `auth/too-many-requests` | Quá nhiều lần thử. Vui lòng thử lại sau. |
| `auth/network-request-failed` | Lỗi kết nối mạng. |
| HTTP `401` từ Gateway | Tự động logout + redirect về `/login` |

---

## 5. Mockup Admin Login

### Mô Tả

Thiết kế giao diện trang đăng nhập dành riêng cho Admin. Tuần 1 tập trung vào mockup UI — chưa có API thực, dùng Firebase Auth trực tiếp.

### Thành Phần Giao Diện

| Thành Phần | Mô Tả |
|------------|-------|
| Logo / Branding | Hiển thị logo platform |
| Input Email | Nhập địa chỉ email |
| Input Password | Nhập mật khẩu, có toggle ẩn/hiện |
| Button Đăng Nhập | Submit form đăng nhập |
| Thông báo lỗi | Hiển thị inline bên dưới input tương ứng |

### Behavior

- Validate email format trước khi gọi Firebase
- Sau khi submit: hiển thị loading state trên button
- Đăng nhập thành công → redirect `/dashboard`
- Đăng nhập thất bại → hiển thị lỗi inline theo error mapping ở mục 4
- Đã login mà vào lại `/login` → tự redirect sang `/dashboard`
- Chưa login mà vào route cần auth → redirect về `/login`

---

## 6. Mockup Data Toàn Hệ Thống

### Mô Tả

Dựng static mock data cho tất cả màn hình để UI hiển thị đúng cấu trúc và layout — chưa kết nối API thực. Mock data cần bám sát schema API thực (confirm với Backend) để sau dễ thay thế.

### 6.1 Dashboard (3.2)

Hiển thị 3 widget tổng quan:

| Widget | Dữ Liệu Hiển Thị |
|--------|-----------------|
| Users Overview | Total users · Active · Inactive · New this month |
| Partners Overview | Total partners · Active · Pending |
| Services Overview | Total services · Running · Stopped |

### 6.2 Users Management (3.3.1)

Bảng danh sách người dùng với các cột: Avatar · Tên · Email · Role · Status · Ngày tạo

Chức năng tuần 1 (mockup):
- Filter theo Status (Active / Inactive)
- Tìm kiếm theo tên hoặc email
- Phân trang

### 6.3 Partner Management (3.3.2)

Danh sách partner, mỗi partner có 2 tab con:

**Tab Organization (3.3.2.1)**
- Tên công ty, địa chỉ, email liên hệ, trạng thái hoạt động
- Danh sách thành viên của partner

**Tab Tokens (3.3.2.2)**
- Bảng: Token name · Created date · Last used · Status

---

## 7. Test Cases

### 7.1 Firebase Authentication

| ID | Tên Test Case | Bước Thực Hiện | Kết Quả Mong Đợi | Priority |
|----|--------------|----------------|------------------|---------|
| TC-A01 | Login thành công | Nhập email/pass hợp lệ → Đăng Nhập | Vào `/dashboard`, user lưu vào store | P0 |
| TC-A02 | Sai mật khẩu | Email đúng, pass sai → Đăng Nhập | Hiện "Sai email hoặc mật khẩu." | P0 |
| TC-A03 | Email không tồn tại | Nhập email chưa có tài khoản | Hiện "Email không tồn tại trong hệ thống." | P0 |
| TC-A04 | Email sai format | Nhập "abc" vào email | Hiện "Email không đúng định dạng." | P0 |
| TC-A05 | Để trống email | Không nhập email → Đăng Nhập | Hiện "Vui lòng nhập email." | P1 |
| TC-A06 | Để trống password | Không nhập password → Đăng Nhập | Hiện "Vui lòng nhập mật khẩu." | P1 |
| TC-A07 | Toggle show/hide password | Click icon ẩn → click icon hiện | Password ẩn/hiện đúng | P2 |
| TC-A08 | Chưa login vào route cần auth | Truy cập `/dashboard` khi chưa login | Redirect về `/login` | P0 |
| TC-A09 | Đã login vào lại `/login` | Login xong → vào URL `/login` | Redirect về `/dashboard` | P1 |
| TC-A10 | Token đính vào request | Login → gọi bất kỳ API | Header `Authorization: Bearer <token>` có trong request | P0 |
| TC-A11 | Token tự động refresh | Sau 1 giờ → gọi API | Request thành công, không cần login lại | P1 |
| TC-A12 | Nhận 401 từ Gateway | Simulate 401 response | Tự động logout + redirect `/login` | P0 |
| TC-A13 | Logout | Click nút Logout | Xóa store, redirect `/login` | P0 |

### 7.2 Dashboard & Overview Widgets

| ID | Tên Test Case | Bước Thực Hiện | Kết Quả Mong Đợi | Priority |
|----|--------------|----------------|------------------|---------|
| TC-D01 | Dashboard hiển thị sau login | Login thành công → quan sát | 3 widget hiển thị với mock data | P0 |
| TC-D02 | Users Overview | Xem widget Users | Hiển thị đủ: Total, Active, Inactive, New | P1 |
| TC-D03 | Partners Overview | Xem widget Partners | Hiển thị đủ: Total, Active, Pending | P1 |
| TC-D04 | Services Overview | Xem widget Services | Hiển thị đủ: Total, Running, Stopped | P1 |
| TC-D05 | Responsive layout | Xem ở 1280px và 1440px | Layout không vỡ, sidebar/topbar đúng vị trí | P2 |

### 7.3 Users Management

| ID | Tên Test Case | Bước Thực Hiện | Kết Quả Mong Đợi | Priority |
|----|--------------|----------------|------------------|---------|
| TC-U01 | Hiển thị danh sách | Vào trang Users | Bảng đủ cột, mock data hiển thị đúng | P0 |
| TC-U02 | Filter theo Active | Chọn Status = Active | Chỉ hiển thị user Active | P1 |
| TC-U03 | Filter theo Inactive | Chọn Status = Inactive | Chỉ hiển thị user Inactive | P1 |
| TC-U04 | Tìm kiếm theo email | Nhập email vào search | Kết quả lọc đúng theo email nhập | P1 |
| TC-U05 | Phân trang | Xem trang 1 → chuyển trang 2 | Dữ liệu thay đổi, số trang hiển thị đúng | P1 |

### 7.4 Partner Management

| ID | Tên Test Case | Bước Thực Hiện | Kết Quả Mong Đợi | Priority |
|----|--------------|----------------|------------------|---------|
| TC-P01 | Hiển thị danh sách partner | Vào trang Partner | Bảng đúng cấu trúc, mock data hiển thị | P0 |
| TC-P02 | Xem tab Organization | Click partner → tab Organization | Hiển thị thông tin tổ chức đúng | P0 |
| TC-P03 | Xem tab Tokens | Click partner → tab Tokens | Hiển thị danh sách token mock | P0 |
| TC-P04 | Chuyển qua lại 2 tab | Click Organization ↔ Tokens | Nội dung đổi đúng, không bị lỗi render | P1 |

---
