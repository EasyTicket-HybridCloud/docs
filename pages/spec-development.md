# Luồng nghiệp vụ (Development Specification)

## Mục lục

1. [Tổng quan](#1-tổng-quan)
2. [Luồng 1 - Đăng ký tài khoản](#luồng-1---đăng-ký-tài-khoản)
3. [Luồng 2 - Đăng nhập](#luồng-2---đăng-nhập)
4. [Luồng 3 - Tạo &amp; cấu hình sự kiện](#luồng-3---tạo--cấu-hình-sự-kiện)
5. [Luồng 4 - Flash Sale: Mua vé (luồng lõi)](#luồng-4---flash-sale-mua-vé-luồng-lõi)
6. [Luồng 5 - Xử lý thanh toán](#luồng-5---xử-lý-thanh-toán)
7. [Luồng 6 - Hủy đơn &amp; hoàn vé](#luồng-6---hủy-đơn--hoàn-vé)
8. [Luồng 7 - Gửi vé qua email](#luồng-7---gửi-vé-qua-email)
9. [Luồng 8 - Xem lịch sử vé](#luồng-8---xem-lịch-sử-vé)
10. [Luồng 9 - Quản trị hệ thống](#luồng-9---quản-trị-hệ-thống)

---

## 1. Tổng quan

EasyTicket có 9 luồng nghiệp vụ chính, trong đó **Luồng 4 (Flash Sale)** là quan trọng nhất - phải đảm bảo:

- Không sập web dưới tải cao
- Không bán vượt số lượng vé (over-selling)
- Phản hồi tín bằng mili-giây

### Các Kafka Topics

| Topic               | Producer        | Consumer                            | Nội dung                                                                                 |
| ------------------- | --------------- | ----------------------------------- | ----------------------------------------------------------------------------------------- |
| `ticket-reserved` | Ticket Service  | Order Service                       | `reservationId`, `userId`, `eventId`, `ticketTypeId`, `quantity`, `unitPrice` |
| `payment-success` | Payment Service | Order Service, Notification Service | `orderId`, `paymentId`, `paidAt`                                                    |
| `payment-failed`  | Payment Service | Order Service, Ticket Service       | `orderId`, `reservationId`, `reason` (`DECLINED` \| `TIMEOUT`)                  |

---

## Luồng 1 - Đăng ký tài khoản

### Mô tả

Người dùng tự đăng ký (Buyer, Organizer) hoặc được Admin tạo tài khoản (Admin).

### Các bước xử lý

1. **Client gửi request**

   ```
   POST /api/v1/users/register/{buyer|organizer}
   POST /api/v1/users/register/admin  (cần JWT Admin)
   ```

   Body: `{ username, password, email, fullName }`
2. **Validation**

   - Username: 3-50 ký tự
   - Password: 8-128 ký tự
   - Email: hợp lệ
   - FullName: 1-100 ký tự
   - Sai → trả `400 VALIDATION_ERROR`, **không gọi Keycloak**
3. **Kiểm tra quyền (Admin)**

   - `/register/admin` cần JWT với role `ADMIN`
   - Thiếu token → `401`
   - Sai role → `403 FORBIDDEN`
4. **Tạo user trên Keycloak**

   ```
   UserService → Keycloak Admin Client → realm.users().create(UserRepresentation)
   ```

   - Username/email đã tồn tại → `409 USER_ALREADY_EXISTS`
   - Thành công → parse `keycloakUserId` từ header `Location`
5. **Gán role**

   - Gán role tương ứng: `BUYER`/`ORGANIZER`/`ADMIN`
   - Qua: `clientLevel(clientId).add(...)`
6. **Tạo UserProfile**

   - `id = keycloakUserId`
   - Đăng ký tự phục vụ: `createdBy = keycloakUserId` (AuditorAware trả rỗng khi không có JWT)
   - Đăng ký admin: `createdBy` từ JWT `sub` của Admin
7. **Rollback nếu thất bại**

   - Lưu DB thất bại → xóa user trên Keycloak
   - Trả `500 REGISTRATION_FAILED`
8. **Response**

   ```json
   201 ApiResponse.ok({ id: keycloakUserId })
   ```

### Sơ đồ luồng

```
Client → UserController → UserService → Keycloak Admin Client
                                    ↓
                              Tạo UserProfile → user_db
                                    ↓
                              Response: 201
```

---

## Luồng 2 - Đăng nhập

### Mô tả

UserService làm cầu nối, chuyển tiếp request sang Keycloak.

### Các bước xử lý

1. **Client gửi request**

   ```
   POST /api/v1/users/login
   ```

   Body: `{ username, password }`
2. **Validation**

   - 1-255 ký tự mỗi trường
   - Sai → `400 VALIDATION_ERROR`, không gọi Keycloak
3. **Gọi Keycloak token endpoint**

   ```
   POST {serverUrl}/realms/{realm}/protocol/openid-connect/token
   grant_type=password&client_id={clientId}&client_secret={secret}
   &username={username}&password={password}
   ```

   - Timeout: 5s
4. **Xử lý response Keycloak**

   - Thành công (200): parse JWT, trích role từ `resource_access.{clientId}.roles`
   - Sai mật khẩu (401): trả `401 INVALID_CREDENTIALS`
   - Timeout/Lỗi 5xx: trả `502 KEYCLOAK_UNAVAILABLE`
5. **Response**

   ```json
   200 ApiResponse.ok({ 
     accessToken, 
     refreshToken, 
     roles: ["BUYER"] 
   })
   ```

   **Không log** access/refresh token

### Sơ đồ luồng

```
Client → POST /api/v1/users/login
              ↓
         UserController
              ↓
         Keycloak Token Endpoint
              ↓
         Parse JWT → Extract roles
              ↓
         Response: 200 { accessToken, refreshToken, roles }
```

---

## Luồng 3 - Tạo & cấu hình sự kiện

### Mô tả

Organizer thiết lập thông tin sự kiện trước khi mở bán vé. Đây là bước chuẩn bị dữ liệu cho Luồng 4.

### Các bước xử lý

1. **Đăng nhập**

   - Organizer đăng nhập (Luồng 2) → JWT có role `ORGANIZER`
2. **Tạo sự kiện (draft)**

   ```
   POST /api/v1/events
   ```

   - Tên, mô tả, địa điểm, thời gian, danh mục
   - Trạng thái: `DRAFT`
3. **Thêm loại vé**

   ```
   POST /api/v1/events/{eventId}/ticket-types
   ```

   - Mỗi lần một hạng vé
   - Tên loại vé, giá, số lượng phát hành
   - **Chỉ khi event còn `DRAFT`**
4. **Lên lịch Flash Sale**

   ```
   POST /api/v1/events/{eventId}/flash-sale
   ```

   - Cấu hình `startAt`/`endAt`
   - **Chỉ khi event còn `DRAFT`**
   - Mỗi event chỉ có một flash sale
5. **Cập nhật Event Service**

   - Ghi vào `event_db`
   - Invalidate/refresh Redis cache
6. **Nạp tồn kho vào Redis**

   - Tại thời điểm `startAt`
   - `TicketService-worker` scheduler gọi `GET /api/v1/events/{eventId}/ticket-types`
   - Ghi Redis key: `ticket:inventory:{eventId}:{ticketTypeId} = quantity`
   - Từ thời điểm này, **Redis là nguồn sự thật duy nhất**
7. **Theo dõi tiến độ**

   ```
   GET /api/v1/events/organizer-history
   ```

### Ràng buộc quan trọng

> Loại vé và flash sale chỉ có thể tạo/sửa khi event còn `DRAFT`. Publish event sẽ khóa cấu hình vĩnh viễn.

### Sơ đồ luồng

```
Organizer → POST /api/v1/events (DRAFT)
              ↓
         POST /api/v1/events/{id}/ticket-types
              ↓
         POST /api/v1/events/{id}/flash-sale
              ↓
         TicketService-worker → startAt → Load inventory to Redis
```

---

## Luồng 4 - Flash Sale: Mua vé (luồng lõi)

### Mô tả

Đây là luồng quan trọng nhất - phải đảm bảo không sập web, không over-selling, phản hồi mili-giây.

### Sơ đồ tổng quan

```
User click "Mua vé"
        │
        ▼
  API Gateway ──── Rate Limiting
        │
        ▼
  Ticket Service ──▶ Redis Lua Script: CHECK & DECREMENT
        │
  ┌────┴────┐
Hết vé    Còn vé
  │          │
  ▼          ▼
Response  Kafka(ticket-reserved) → Order Service → PENDING_PAYMENT
"Hết vé"                                 │
                                          ▼
                                   Payment Service (timeout 2 phút)
                                    ┌────┴────┐
                                 Success   Failed/Timeout
                                    │          │
                                    ▼          ▼
                                  PAID    CANCELLED + release vé
                                    │
                                    ▼
                          Notification → Email vé QR
```

### Các bước xử lý chi tiết

1. **Rate Limiting (API Gateway)**

   - Request đi qua AWS API Gateway (dev/prod) hoặc NGINX (local)
   - Rate limiting theo IP/user để chặn bot
2. **Xác thực & gọi Ticket Service**

   ```
   POST /api/v1/tickets/{eventId}/purchase
   Authorization: Bearer {JWT} (role: BUYER)
   Body: { ticketTypeId, quantity }
   ```
3. **Thực thi Lua Script (Redis)**

   ```lua
   -- CHECK: Kiểm tra tồn kho >= quantity
   -- DECREMENT: Trừ ngay trong cùng lệnh
   -- Tất cả nguyên tử - không race condition
   local current = redis.call('GET', key)
   if tonumber(current) >= tonumber(quantity) then
       redis.call('DECRBY', key, quantity)
       return current - quantity
   else
       return -1  -- Hết vé
   end
   ```
4. **Xử lý kết quả**

   **Hết vé:**

   - Trả ngay `409 ApiResponse.error("SOLD_OUT", "Hết vé")`
   - Không tạo order, không publish Kafka
   - Giữ độ trễ tối thiểu

   **Còn vé:**

   - Publish Kafka `ticket-reserved` (key = `eventId` để ordering)
   - Payload: `{ reservationId, userId, eventId, ticketTypeId, quantity, unitPrice }`
   - Trả ngay `200 ApiResponse.ok({ reservationId })`
   - **Không chờ** Order Service xử lý (Async Order Processing)
5. **Order Service nhận Kafka**

   - Consumer nhận `ticket-reserved`
   - Idempotent theo `reservationId`
   - Tạo Order: `status = PENDING_PAYMENT`
6. **Client tiến hành thanh toán**

   - Xem **Luồng 5**
7. **Nếu thanh toán thất bại**

   - Xem **Luồng 6** (hoàn vé)

### Cơ chế chống Over-Selling

1. **Redis Lua Script** - CHECK & DECREMENT nguyên tử

   - Kiểm tra tồn kho và trừ trong một lệnh duy nhất
   - Không dùng lock ở tầng DB
   - Loại bỏ hoàn toàn race condition
2. **Ticket Service là nguồn sự thật duy nhất**

   - Event Service không quyết định số vé còn lại
   - Mọi thao tác trừ/hoàn vé đều qua Lua Script

### Đảm bảo Performance

| Yếu tố            | Giải pháp                              |
| ------------------- | ---------------------------------------- |
| Không sập web     | Rate limiting ở API Gateway             |
| Phản hồi nhanh    | Async processing - không block request  |
| Milisecond response | Redis Lua Script thực thi O(1)          |
| Scalability         | Stateless service, horizontally scalable |

---

## Luồng 5 - Xử lý thanh toán

### Mô tả

Payment Service xử lý giao dịch cho order `PENDING_PAYMENT`, giới hạn 2 phút.

### Các bước xử lý

1. **Client khởi tạo thanh toán**

   ```
   POST /api/v1/payments
   Body: { orderId, paymentMethod }
   ```

   - `paymentMethod`: `CARD`, `MOMO`, `VNPAY`, `BANK_TRANSFER`
2. **Tạo Payment record**

   - Trạng thái: `PENDING`
   - `expires_at = created_at + 2 phút`
   - Khởi động timeout scheduler
3. **Client thanh toán qua cổng bên thứ ba**

   ```
   POST /api/v1/payments/{paymentId}/callback
   ```

   - Webhook có chữ ký số
4. **Xử lý kết quả**

   **Thành công:**

   - Cập nhật `status = SUCCESS`
   - Publish Kafka `payment-success`: `{ orderId, paymentId, paidAt }`
   - Order Service: cập nhật `Order.status = PAID`
   - Notification Service: gửi vé QR (Luồng 7)

   **Thất bại/Hết hạn:**

   - Cập nhật `status = FAILED`
   - `failed_reason`: `DECLINED` | `TIMEOUT`
   - Publish Kafka `payment-failed`: `{ orderId, reservationId, reason }`
   - Order Service: cập nhật `Order.status = CANCELLED`
   - Ticket Service: hoàn vé về Redis (Luồng 6)
5. **Timeout Scheduler**

   - Chạy định kỳ kiểm tra payment quá hạn
   - Tự động đánh dấu `FAILED` + publish `payment-failed`

### Sơ đồ luồng

```
Client → POST /api/v1/payments
              ↓
         PaymentService → Tạo Payment (PENDING)
              ↓
         Callback từ cổng thanh toán
              ↓
    ┌────────┴────────┐
    │                 │
Success           Failed/Timeout
    │                 │
    ▼                 ▼
payment-success   payment-failed
    │                 │
    ▼                 ▼
Order: PAID      Order: CANCELLED
Notification     Ticket: Hoàn vé
```

---

## Luồng 6 - Hủy đơn & hoàn vé

### Mô tả

Đảm bảo vé không bị "khóa chết" khi buyer không thanh toán.

### Các bước xử lý

1. **TicketService nhận Kafka**

   ```
   Consumer: payment-failed
   Idempotent theo: reservationId
   ```
2. **Thực thi Lua Script INCREMENT**

   ```lua
   -- Hoàn vé: cộng trả quantity đã trừ
   redis.call('INCRBY', key, quantity)
   ```
3. **Cập nhật availability**

   - Từ thời điểm này, vé xuất hiện trong:

   ```
   GET /api/v1/tickets/{eventId}/availability
   ```
4. **Vé sẵn sàng cho buyer khác**

   - Lượt mua vé tiếp theo có thể thành công

### Đảm bảo Idempotent

- Kafka message có thể deliver lại (re-delivery)
- Kiểm tra `reservationId` đã xử lý chưa trước khi hoàn
- Tránh hoàn vé hai lần

---

## Luồng 7 - Gửi vé qua email

### Mô tả

Sau khi thanh toán thành công, buyer nhận vé có mã QR qua email.

### Kiến trúc (app → SQS → SES)

```
NotificationService → AWS SQS (ticket-email queue)
                                ↓
                    Lambda/Consumer riêng
                                ↓
                          AWS SES → Email
```

### Các bước xử lý (phase 1 - app → SQS)

1. **Nhận Kafka message**

   ```
   Consumer: payment-success
   Idempotent theo: orderId + type
   ```
2. **Publish sang SQS**

   - Queue: `ticket-email`
   - Payload: `{ orderId, paymentId, paidAt, type: "TICKET_QR" }`
3. **Ghi log vào DB**

   - Bảng `notifications`
   - Trạng thái: `QUEUED` hoặc `QUEUE_FAILED`
4. **Retry nếu thất bại**

   - Kafka retry theo cấu hình Spring Boot

### Các bước xử lý (phase 2 - SQS → SES) [Chưa implement]

1. **Consumer đọc queue SQS**
2. **Lấy thêm thông tin**
   - Order/Event: gọi REST API
   - Email user: Keycloak/UserProfile
3. **Render vé điện tử**
   - Mã QR encode `orderId`/`ticketId`
4. **Gửi email qua AWS SES**

---

## Luồng 8 - Xem lịch sử vé

### Mô tả

UserService đóng vai trò tổng hợp dữ liệu từ các service khác.

### Luồng Buyer xem lịch sử vé

```
Client → GET /api/v1/users/me/ticket-history
              ↓
         UserService → Extract userId từ JWT
              ↓
         Feign → OrderService: GET /api/v1/orders/my-tickets
              ↓
         Response: Danh sách vé đã mua
```

### Luồng Organizer xem thống kê

```
Client → GET /api/v1/users/me/organizer-history
              ↓
         UserService → Feign → EventService
                                   GET /api/v1/events/organizer-history
              ↓
         Response: Thống kê sự kiện đã tổ chức
```

### Xử lý lỗi

| Lỗi                       | Response                          |
| -------------------------- | --------------------------------- |
| Order Service timeout (5s) | `502 ORDER_SERVICE_UNAVAILABLE` |
| Event Service timeout (5s) | `502 EVENT_SERVICE_UNAVAILABLE` |
| Thiếu role ORGANIZER      | `403 FORBIDDEN`                 |

---

## Luồng 9 - Quản trị hệ thống

### Mô tả

System Admin giám sát và quản lý tài khoản.

### Các bước xử lý

1. **Đăng nhập Admin**

   - JWT có role `ADMIN`
2. **Quản lý tài khoản**

   ```
   GET /api/v1/users                    # Danh sách tài khoản
   PATCH /api/v1/users/{userId}/status  # Khóa/mở khóa
   ```
   - Đồng bộ trạng thái `enabled` trên Keycloak
3. **Tạo tài khoản Admin khác**

   ```
   POST /api/v1/users/register/admin
   ```
4. **Giám sát hệ thống**

   - Kibana: traffic, CPU/RAM, độ trễ
   - Pipeline: `Spring Boot → OTel → Elasticsearch → Kibana`
   - Trace: `traceId`/`spanId` xuyên suốt 5 service
5. **Theo dõi sự kiện**

   ```
   GET /api/v1/events  # Danh sách event đang publish
   ```
