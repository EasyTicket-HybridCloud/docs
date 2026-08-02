# Thiết kế API (Endpoint Design)

## Mục lục

1. [Quy ước chung](#1-quy-ước-chung)
2. [User Service - api/v1/users](#2-user-service---apiv1users)
3. [Event Service - api/v1/events](#3-event-service---apiv1events)
4. [Ticket Service - api/v1/tickets](#4-ticket-service---apiv1tickets)
5. [Order Service - api/v1/orders](#5-order-service---apiv1orders)
6. [Payment Service - api/v1/payments](#6-payment-service---apiv1payments)
7. [Notification Service - api/v1/notifications](#7-notification-service---apiv1notifications)

---

## 1. Quy ước chung

### API Response Format

```json
{
  "success": true,
  "errorCode": null,
  "message": "Operation successful",
  "data": { ... },
  "traceId": "abc123"
}
```

```json
{
  "success": false,
  "errorCode": "VALIDATION_ERROR",
  "message": "Invalid input",
  "data": null,
  "traceId": "abc123"
}
```

### Authentication & Authorization

| Symbol | Meaning |
|--------|---------|
| ✗ | Public endpoint (không cần auth) |
| ✓ | Cần JWT Bearer token |
| `ROLE` | Yêu cầu role cụ thể |

### HTTP Status Codes

| Code | Usage |
|------|-------|
| 200 | Thành công |
| 201 | Tạo mới thành công |
| 400 | Validation error |
| 401 | Chưa xác thực |
| 403 | Không có quyền |
| 404 | Resource không tìm thấy |
| 409 | Conflict (ví dụ: hết vé) |
| 500 | Lỗi server |

### Versioning

- Tất cả endpoints có prefix: `api/v1/...`
- Breaking changes → `api/v2/...`, giữ `v1` backward-compatible

---

## 2. User Service - api/v1/users

### Authentication Endpoints

#### POST /api/v1/users/login

Đăng nhập - delegate sang Keycloak.

```json
// Request
{
  "username": "john_doe",
  "password": "SecurePass123"
}

// Response 200
{
  "success": true,
  "data": {
    "accessToken": "eyJhbGciOiJSUzI1NiIs...",
    "refreshToken": "eyJhbGciOiJIUzI1NiIs...",
    "roles": ["BUYER"]
  }
}

// Response 401
{
  "success": false,
  "errorCode": "INVALID_CREDENTIALS",
  "message": "Invalid username or password"
}
```

| Auth | Role | Mô tả |
|------|------|-------|
| ✗ | - | Đăng nhập, nhận JWT tokens |

---

#### POST /api/v1/users/register/buyer

Đăng ký tài khoản Ticket Buyer.

```json
// Request
{
  "username": "john_doe",
  "password": "SecurePass123",
  "email": "john@example.com",
  "fullName": "John Doe"
}

// Response 201
{
  "success": true,
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440000"
  }
}

// Response 409
{
  "success": false,
  "errorCode": "USER_ALREADY_EXISTS",
  "message": "Username or email already exists"
}
```

| Auth | Role | Mô tả |
|------|------|-------|
| ✗ | - | Tạo tài khoản Buyer mới |

---

#### POST /api/v1/users/register/organizer

Đăng ký tài khoản Organizer.

```json
// Request
{
  "username": "event_organizer",
  "password": "SecurePass123",
  "email": "organizer@example.com",
  "fullName": "Event Organizer"
}

// Response 201
{
  "success": true,
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440001"
  }
}
```

| Auth | Role | Mô tả |
|------|------|-------|
| ✗ | - | Tạo tài khoản Organizer mới |

---

#### POST /api/v1/users/register/admin

Admin tạo tài khoản Admin khác.

```json
// Request
{
  "username": "new_admin",
  "password": "SecurePass123",
  "email": "admin@example.com",
  "fullName": "New Admin"
}

// Response 201
{
  "success": true,
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440002"
  }
}
```

| Auth | Role | Mô tả |
|------|------|-------|
| ✓ | `ADMIN` | Tạo tài khoản Admin mới |

---

### Profile Endpoints

#### GET /api/v1/users/me

Xem hồ sơ cá nhân.

```json
// Response 200
{
  "success": true,
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "username": "john_doe",
    "email": "john@example.com",
    "fullName": "John Doe",
    "phone": "0901234567",
    "avatarUrl": "https://example.com/avatar.jpg",
    "address": "123 Main St, Hanoi",
    "roles": ["BUYER"]
  }
}
```

| Auth | Role | Mô tả |
|------|------|-------|
| ✓ | - | Xem hồ sơ của chính mình |

---

#### PUT /api/v1/users/me

Cập nhật hồ sơ cá nhân.

```json
// Request (PATCH semantics - các trường không gửi sẽ giữ nguyên)
{
  "fullName": "John Updated",
  "phone": "0909876543",
  "avatarUrl": "https://example.com/new-avatar.jpg",
  "address": "456 New St, Hanoi"
}

// Response 200
{
  "success": true,
  "data": { ... }
}
```

| Auth | Role | Mô tả |
|------|------|-------|
| ✓ | - | Cập nhật hồ sơ (PATCH semantics) |

---

#### GET /api/v1/users/me/ticket-history

Lịch sử vé đã mua (aggregated từ Order Service).

```json
// Response 200
{
  "success": true,
  "data": [
    {
      "orderId": "order-uuid",
      "eventId": "event-uuid",
      "eventTitle": "Concert 2024",
      "ticketType": "VIP",
      "quantity": 2,
      "totalAmount": 1000.00,
      "status": "PAID",
      "purchasedAt": "2024-01-15T10:30:00Z"
    }
  ]
}
```

| Auth | Role | Mô tả |
|------|------|-------|
| ✓ | - | Lịch sử vé đã mua |

---

#### GET /api/v1/users/me/organizer-history

Thống kê sự kiện đã tổ chức (aggregated từ Event Service).

```json
// Response 200
{
  "success": true,
  "data": {
    "totalEvents": 5,
    "totalTicketsSold": 1200,
    "totalRevenue": 500000.00,
    "events": [
      {
        "eventId": "event-uuid",
        "title": "Concert 2024",
        "ticketsSold": 500,
        "revenue": 250000.00
      }
    ]
  }
}
```

| Auth | Role | Mô tả |
|------|------|-------|
| ✓ | `ORGANIZER` | Thống kê sự kiện |

---

### Admin Endpoints

#### GET /api/v1/users

Danh sách tài khoản (Admin only).

```json
// Query params: ?role=ORGANIZER&status=ACTIVE&page=0&size=20

// Response 200
{
  "success": true,
  "data": {
    "content": [
      {
        "id": "user-uuid",
        "username": "organizer1",
        "email": "org@example.com",
        "fullName": "Organizer One",
        "roles": ["ORGANIZER"],
        "status": "ACTIVE",
        "createdAt": "2024-01-01T00:00:00Z"
      }
    ],
    "totalElements": 50,
    "totalPages": 3,
    "page": 0,
    "size": 20
  }
}
```

| Auth | Role | Mô tả |
|------|------|-------|
| ✓ | `ADMIN` | Danh sách tài khoản |

---

#### PATCH /api/v1/users/{userId}/status

Khóa/mở khóa tài khoản Organizer.

```json
// Request
{
  "enabled": false  // true = mở khóa, false = khóa
}

// Response 200
{
  "success": true
}
```

| Auth | Role | Mô tả |
|------|------|-------|
| ✓ | `ADMIN` | Cập nhật trạng thái tài khoản |

---

## 3. Event Service - api/v1/events

### Event CRUD

#### POST /api/v1/events

Tạo sự kiện mới (draft).

```json
// Request
{
  "title": "Music Festival 2024",
  "description": "The biggest music event of the year",
  "categoryId": "category-uuid",
  "locationId": "location-uuid",
  "location": "10 Mễ Trì Hạ, Nam Từ Liêm, Hà Nội",
  "bannerUrl": "https://example.com/banner.jpg",
  "startTime": "2024-06-01T18:00:00Z",
  "endTime": "2024-06-01T23:00:00Z"
}

// Response 201
{
  "success": true,
  "data": {
    "id": "event-uuid",
    "status": "DRAFT"
  }
}
```

| Auth | Role | Mô tả |
|------|------|-------|
| ✓ | `ORGANIZER` | Tạo sự kiện mới (trạng thái DRAFT) |

---

#### PUT /api/v1/events/{eventId}

Cập nhật thông tin sự kiện.

```json
// Request
{
  "title": "Updated Title",
  "description": "Updated description",
  "bannerUrl": "https://example.com/new-banner.jpg",
  "startTime": "2024-06-15T18:00:00Z",
  "endTime": "2024-06-15T23:00:00Z",
  "status": "PUBLISHED"  // Publish event
}

// Response 200
{
  "success": true,
  "data": { ... }
}
```

| Auth | Role | Mô tả |
|------|------|-------|
| ✓ | `ORGANIZER` (chủ sở hữu) | Cập nhật sự kiện |

> **Lưu ý:** Khi `status=PUBLISHED`, không thể sửa ticket-types và flash-sale.

---

#### DELETE /api/v1/events/{eventId}

Soft delete sự kiện.

```json
// Response 200
{
  "success": true
}
```

| Auth | Role | Mô tả |
|------|------|-------|
| ✓ | `ORGANIZER` (chủ sở hữu) | Xóa mềm sự kiện |

---

#### GET /api/v1/events

Tìm kiếm/lọc sự kiện (cache Redis).

```json
// Query params:
// ?categoryId=uuid&locationId=uuid&startDate=2024-06-01&endDate=2024-06-30&status=PUBLISHED&page=0&size=20

// Response 200
{
  "success": true,
  "data": {
    "content": [
      {
        "id": "event-uuid",
        "title": "Music Festival 2024",
        "description": "...",
        "category": "Nhạc sống",
        "location": "Hà Nội",
        "locationDetail": "10 Mễ Trì Hạ, Nam Từ Liêm",
        "bannerUrl": "...",
        "startTime": "2024-06-01T18:00:00Z",
        "endTime": "2024-06-01T23:00:00Z",
        "status": "PUBLISHED"
      }
    ],
    "totalElements": 100,
    "totalPages": 5
  }
}
```

| Auth | Role | Mô tả |
|------|------|-------|
| ✗ | - | Tìm kiếm sự kiện (public) |

---

#### GET /api/v1/events/{eventId}

Chi tiết sự kiện (chỉ PUBLISHED, cache Redis).

```json
// Response 200
{
  "success": true,
  "data": {
    "id": "event-uuid",
    "title": "Music Festival 2024",
    "description": "...",
    "category": "Nhạc sống",
    "categoryId": "...",
    "location": "Hà Nội",
    "locationDetail": "10 Mễ Trì Hạ, Nam Từ Liêm",
    "bannerUrl": "...",
    "startTime": "2024-06-01T18:00:00Z",
    "endTime": "2024-06-01T23:00:00Z",
    "status": "PUBLISHED",
    "organizerId": "...",
    "ticketTypes": [
      {
        "id": "ticket-type-uuid",
        "name": "VIP",
        "price": 500.00,
        "available": 100
      }
    ]
  }
}
```

| Auth | Role | Mô tả |
|------|------|-------|
| ✗ | - | Chi tiết sự kiện (chỉ PUBLISHED) |

---

#### GET /api/v1/events/mine

Danh sách sự kiện của chính mình (mọi trạng thái).

```json
// Response 200
{
  "success": true,
  "data": {
    "content": [
      {
        "id": "event-uuid",
        "title": "My Event",
        "status": "DRAFT",
        ...
      }
    ]
  }
}
```

| Auth | Role | Mô tả |
|------|------|-------|
| ✓ | `ORGANIZER` | Sự kiện của tôi |

---

#### GET /api/v1/events/{eventId}/manage

Chi tiết sự kiện của chính mình (bất kỳ trạng thái).

```json
// Response 200
{
  "success": true,
  "data": {
    // Full event details including all ticket types
  }
}
```

| Auth | Role | Mô tả |
|------|------|-------|
| ✓ | `ORGANIZER` (chủ sở hữu) | Form chỉnh sửa |

---

### Ticket Types

#### POST /api/v1/events/{eventId}/ticket-types

Thêm loại vé.

```json
// Request
{
  "name": "VIP",
  "price": 500.00,
  "totalQuantity": 100
}

// Response 201
{
  "success": true,
  "data": {
    "id": "ticket-type-uuid"
  }
}
```

| Auth | Role | Mô tả |
|------|------|-------|
| ✓ | `ORGANIZER` (chủ sở hữu) | Thêm loại vé (chỉ DRAFT) |

---

#### PUT /api/v1/events/{eventId}/ticket-types/{ticketTypeId}

Cập nhật loại vé.

```json
// Request
{
  "name": "VIP Plus",
  "price": 600.00,
  "totalQuantity": 150
}

// Response 200
{
  "success": true
}
```

| Auth | Role | Mô tả |
|------|------|-------|
| ✓ | `ORGANIZER` (chủ sở hữu) | Cập nhật loại vé (chỉ DRAFT) |

---

#### GET /api/v1/events/{eventId}/ticket-types

Danh sách loại vé.

```json
// Response 200
{
  "success": true,
  "data": [
    {
      "id": "ticket-type-uuid",
      "name": "VIP",
      "price": 500.00,
      "totalQuantity": 100
    },
    {
      "id": "ticket-type-uuid-2",
      "name": "Standard",
      "price": 250.00,
      "totalQuantity": 500
    }
  ]
}
```

| Auth | Role | Mô tả |
|------|------|-------|
| ✗ | - | Danh sách loại vé |

---

### Flash Sale

#### POST /api/v1/events/{eventId}/flash-sale

Lên lịch flash sale.

```json
// Request
{
  "startAt": "2024-05-01T10:00:00Z",
  "endAt": "2024-05-01T12:00:00Z"
}

// Response 201
{
  "success": true,
  "data": {
    "id": "flash-sale-uuid"
  }
}
```

| Auth | Role | Mô tả |
|------|------|-------|
| ✓ | `ORGANIZER` (chủ sở hữu) | Lên lịch flash sale (chỉ DRAFT) |

> **Lưu ý:** Mỗi event chỉ có một flash sale, không có endpoint sửa/xóa.

---

#### GET /api/v1/events/{eventId}/flash-sale

Xem thông tin flash sale.

```json
// Response 200 (có flash sale)
{
  "success": true,
  "data": {
    "id": "flash-sale-uuid",
    "startAt": "2024-05-01T10:00:00Z",
    "endAt": "2024-05-01T12:00:00Z",
    "status": "SCHEDULED"
  }
}

// Response 200 (chưa cấu hình)
{
  "success": true,
  "data": null
}
```

| Auth | Role | Mô tả |
|------|------|-------|
| ✗ | - | Thông tin flash sale |

---

### Dashboard & Statistics

#### GET /api/v1/events/{eventId}/dashboard

Dashboard doanh thu theo event.

```json
// Response 200
{
  "success": true,
  "data": {
    "eventId": "event-uuid",
    "title": "Music Festival",
    "totalTickets": 1000,
    "soldTickets": 750,
    "revenue": 375000.00,
    "ticketTypes": [
      {
        "name": "VIP",
        "total": 100,
        "sold": 80,
        "revenue": 40000.00
      }
    ]
  }
}
```

| Auth | Role | Mô tả |
|------|------|-------|
| ✓ | `ORGANIZER` (chủ sở hữu) | Dashboard doanh thu |

---

#### GET /api/v1/events/organizer-history

Thống kê toàn bộ sự kiện của organizer.

```json
// Response 200
{
  "success": true,
  "data": {
    "totalEvents": 10,
    "totalTicketsSold": 5000,
    "totalRevenue": 2500000.00,
    "events": [
      {
        "eventId": "...",
        "title": "...",
        "ticketsSold": 500,
        "revenue": 250000.00
      }
    ]
  }
}
```

| Auth | Role | Mô tả |
|------|------|-------|
| ✓ | `ORGANIZER` | Thống kê organizer |

---

### Reference Data

#### GET /api/v1/locations

Danh sách thành phố/tỉnh (cache Redis).

```json
// Response 200
{
  "success": true,
  "data": [
    { "id": "uuid", "name": "Hà Nội" },
    { "id": "uuid", "name": "TP.HCM" }
  ]
}
```

| Auth | Role | Mô tả |
|------|------|-------|
| ✗ | - | Danh sách địa điểm |

---

#### GET /api/v1/categories

Danh sách danh mục sự kiện (cache Redis).

```json
// Response 200
{
  "success": true,
  "data": [
    { "id": "uuid", "name": "Nhạc sống" },
    { "id": "uuid", "name": "Thể thao" }
  ]
}
```

| Auth | Role | Mô tả |
|------|------|-------|
| ✗ | - | Danh sách danh mục |

---

## 4. Ticket Service - api/v1/tickets

### Purchase

#### POST /api/v1/tickets/{eventId}/purchase

Mua vé (thực thi Lua Script).

```json
// Request
{
  "ticketTypeId": "ticket-type-uuid",
  "quantity": 2
}

// Response 200 (thành công)
{
  "success": true,
  "data": {
    "reservationId": "reservation-uuid"
  }
}

// Response 409 (hết vé)
{
  "success": false,
  "errorCode": "SOLD_OUT",
  "message": "Hết vé"
}
```

| Auth | Role | Mô tả |
|------|------|-------|
| ✓ | `BUYER` | Mua vé (Redis Lua Script) |

---

#### GET /api/v1/tickets/{eventId}/availability

Xem số vé còn lại theo loại vé (real-time).

```json
// Response 200
{
  "success": true,
  "data": [
    {
      "ticketTypeId": "uuid",
      "name": "VIP",
      "available": 95
    },
    {
      "ticketTypeId": "uuid",
      "name": "Standard",
      "available": 450
    }
  ]
}
```

| Auth | Role | Mô tả |
|------|------|-------|
| ✗ | - | Số vé còn lại (public) |

---

#### POST /api/v1/tickets/{eventId}/load-inventory

Nạp tồn kho vào Redis (internal, scheduler gọi).

```json
// Response 200
{
  "success": true,
  "data": {
    "loaded": true,
    "ticketTypes": 3
  }
}
```

| Auth | Role | Mô tả |
|------|------|-------|
| ✓ | `SYSTEM` | Load inventory (internal) |

---

## 5. Order Service - api/v1/orders

### Order Endpoints

#### GET /api/v1/orders/{orderId}

Chi tiết đơn hàng.

```json
// Response 200
{
  "success": true,
  "data": {
    "id": "order-uuid",
    "reservationId": "reservation-uuid",
    "userId": "user-uuid",
    "eventId": "event-uuid",
    "ticketTypeId": "ticket-type-uuid",
    "quantity": 2,
    "unitPrice": 500.00,
    "totalAmount": 1000.00,
    "paymentId": "payment-uuid",
    "status": "PENDING_PAYMENT",
    "createdAt": "2024-01-15T10:30:00Z"
  }
}
```

| Auth | Role | Mô tả |
|------|------|-------|
| ✓ | `BUYER` (chủ sở hữu) / `ADMIN` | Chi tiết đơn hàng |

---

#### GET /api/v1/orders/my-tickets

Lịch sử vé đã mua.

```json
// Response 200
{
  "success": true,
  "data": [
    {
      "orderId": "order-uuid",
      "eventId": "event-uuid",
      "eventTitle": "Concert 2024",
      "ticketType": "VIP",
      "quantity": 2,
      "totalAmount": 1000.00,
      "status": "PAID",
      "purchasedAt": "2024-01-15T10:30:00Z"
    }
  ]
}
```

| Auth | Role | Mô tả |
|------|------|-------|
| ✓ | `BUYER` | Lịch sử vé đã mua |

---

#### GET /api/v1/orders

Danh sách toàn bộ đơn hàng (Admin).

```json
// Query params: ?status=PAID&page=0&size=20

// Response 200
{
  "success": true,
  "data": {
    "content": [
      {
        "id": "order-uuid",
        "userId": "user-uuid",
        "eventId": "event-uuid",
        "status": "PAID",
        "totalAmount": 1000.00,
        "createdAt": "2024-01-15T10:30:00Z"
      }
    ],
    "totalElements": 1000,
    "totalPages": 50
  }
}
```

| Auth | Role | Mô tả |
|------|------|-------|
| ✓ | `ADMIN` | Danh sách đơn hàng |

---

## 6. Payment Service - api/v1/payments

### Payment Endpoints

#### POST /api/v1/payments

Khởi tạo thanh toán.

```json
// Request
{
  "orderId": "order-uuid",
  "paymentMethod": "CARD"  // CARD | MOMO | VNPAY | BANK_TRANSFER
}

// Response 201
{
  "success": true,
  "data": {
    "paymentId": "payment-uuid",
    "expiresAt": "2024-01-15T10:32:00Z",  // createdAt + 2 phút
    "checkoutUrl": "https://payment-gateway.com/checkout/xxx"
  }
}

// Response 409 (order đã có payment)
{
  "success": false,
  "errorCode": "PAYMENT_EXISTS",
  "message": "Order already has a payment"
}
```

| Auth | Role | Mô tả |
|------|------|-------|
| ✓ | `BUYER` | Khởi tạo thanh toán (2 phút timeout) |

---

#### GET /api/v1/payments/{paymentId}

Trạng thái giao dịch.

```json
// Response 200
{
  "success": true,
  "data": {
    "id": "payment-uuid",
    "orderId": "order-uuid",
    "amount": 1000.00,
    "paymentMethod": "CARD",
    "status": "PENDING",  // PENDING | SUCCESS | FAILED
    "externalTransactionId": null,
    "failedReason": null,
    "expiresAt": "2024-01-15T10:32:00Z",
    "createdAt": "2024-01-15T10:30:00Z"
  }
}
```

| Auth | Role | Mô tả |
|------|------|-------|
| ✓ | `BUYER` (chủ sở hữu) / `ADMIN` | Trạng thái thanh toán |

---

#### POST /api/v1/payments/{paymentId}/callback

Webhook callback từ cổng thanh toán.

```json
// Request (signature verified)
{
  "status": "SUCCESS",
  "transactionId": "gateway-txn-123",
  "signature": "sha256signature"
}

// Response 200
{
  "success": true
}
```

| Auth | Role | Mô tả |
|------|------|-------|
| Webhook (signed) | - | Callback thanh toán |

---

## 7. Notification Service - api/v1/notifications

### Notification Endpoints

#### GET /api/v1/notifications/me

Lịch sử thông báo/email đã gửi.

```json
// Response 200
{
  "success": true,
  "data": [
    {
      "id": "notification-uuid",
      "orderId": "order-uuid",
      "type": "TICKET_QR",
      "channel": "EMAIL",
      "status": "QUEUED",
      "queuedAt": "2024-01-15T10:35:00Z"
    }
  ]
}
```

| Auth | Role | Mô tả |
|------|------|-------|
| ✓ | `BUYER` | Lịch sử thông báo |

---

## Error Codes Reference

| Error Code | HTTP Status | Mô tả |
|------------|-------------|--------|
| `VALIDATION_ERROR` | 400 | Dữ liệu không hợp lệ |
| `UNAUTHORIZED` | 401 | Chưa đăng nhập |
| `FORBIDDEN` | 403 | Không có quyền |
| `NOT_FOUND` | 404 | Resource không tìm thấy |
| `USER_ALREADY_EXISTS` | 409 | Username/email đã tồn tại |
| `SOLD_OUT` | 409 | Hết vé |
| `EVENT_ALREADY_PUBLISHED` | 409 | Event đã publish |
| `PAYMENT_EXISTS` | 409 | Order đã có payment |
| `PAYMENT_TIMEOUT` | 409 | Payment đã hết hạn |
| `REGISTRATION_FAILED` | 500 | Đăng ký thất bại |
| `ORDER_SERVICE_UNAVAILABLE` | 502 | Order Service không phản hồi |
| `EVENT_SERVICE_UNAVAILABLE` | 502 | Event Service không phản hồi |
| `KEYCLOAK_UNAVAILABLE` | 502 | Keycloak không phản hồi |
