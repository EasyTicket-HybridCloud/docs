# Thiết kế Database

## Mục lục

1. [Tổng quan](#1-tổng-quan)
2. [Nguyên tắc thiết kế](#2-nguyên-tắc-thiết-kế)
3. [User Service - user_db](#3-user-service---user_db)
4. [Event Service - event_db](#4-event-service---event_db)
5. [Ticket Service - Redis](#5-ticket-service---redis)
6. [Order Service - order_db](#6-order-service---order_db)
7. [Payment Service - payment_db](#7-payment-service---payment_db)
8. [Notification Service - notification_db](#8-notification-service---notification_db)
9. [Migration Standards](#9-migration-standards)

---

## 1. Tổng quan

EasyTicket áp dụng mô hình **Database per Service** - mỗi microservice sở hữu database riêng hoàn toàn độc lập.

### Database Summary

| Service              | Database            | Engine    | Host        |
| -------------------- | ------------------- | --------- | ----------- |
| User Service         | `user_db`         | MySQL 8.0 | RDS         |
| Event Service        | `event_db`        | MySQL 8.0 | RDS         |
| Ticket Service       | -                   | Redis 7   | ElastiCache |
| Order Service        | `order_db`        | MySQL 8.0 | RDS         |
| Payment Service      | `payment_db`      | MySQL 8.0 | RDS         |
| Notification Service | `notification_db` | MySQL 8.0 | RDS         |

### Database Relationships

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   user_db    │     │   event_db   │     │   order_db   │
│──────────────│     │──────────────│     │──────────────│
│user_profiles │────▶│   events     │     │   orders     │
└──────────────┘     │──────────────│     │──────────────│
                     │ticket_types  │     └──────┬───────┘
                     │  locations   │            │
                     │ categories   │            ▼
                     │ flash_sales  │     ┌──────────────┐
                     └──────────────┘     │  payment_db  │
                                         │──────────────│
┌──────────────┐     ┌──────────────┐    │  payments    │
│ ticket_svc   │     │notification_ │    └──────────────┘
│──────────────│     │     db       │
│    Redis     │     │──────────────│
│ (Inventory)  │     │notifications │
└──────────────┘     └──────────────┘

Cross-service references are LOGICAL only (no FK constraints)
```

---

## 2. Nguyên tắc thiết kế

### Audit Fields (Bắt buộc)

Mọi bảng MySQL đều có 6 audit columns:

| Column          | Type                         | Constraints                | Description        |
| --------------- | ---------------------------- | -------------------------- | ------------------ |
| `id`          | `CHAR(36)`                 | PK                         | UUID v4            |
| `delete_flag` | `ENUM('ACTIVE','DELETED')` | NOT NULL, DEFAULT 'ACTIVE' | Soft delete        |
| `created_by`  | `VARCHAR(255)`             | NULL                       | Keycloak User UUID |
| `created_at`  | `TIMESTAMP`                | NULL                       | Auto-set by JPA    |
| `updated_by`  | `VARCHAR(255)`             | NULL                       | Keycloak User UUID |
| `updated_at`  | `TIMESTAMP`                | NULL                       | Auto-set by JPA    |

### BaseEntity (JPA)

```java
@MappedSuperclass
@Data
@EntityListeners(AuditingEntityListener.class)
public abstract class BaseEntity {
    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private String id;

    @Enumerated(EnumType.STRING)
    @Column(name = "delete_flag", nullable = false)
    private RecordStatus deleteFlag = RecordStatus.ACTIVE;

    @CreatedBy
    @Column(name = "created_by", updatable = false)
    private String createdBy;

    @CreatedDate
    @Column(name = "created_at", updatable = false)
    private LocalDateTime createdAt;

    @LastModifiedBy
    @Column(name = "updated_by")
    private String updatedBy;

    @LastModifiedDate
    @Column(name = "updated_at")
    private LocalDateTime updatedAt;
}
```

### Naming Conventions

| Object      | Convention               | Example                              |
| ----------- | ------------------------ | ------------------------------------ |
| Table       | `snake_case`, plural   | `user_profiles`, `ticket_orders` |
| Column      | `snake_case`           | `created_at`, `user_id`          |
| Primary Key | `id`                   | `id CHAR(36)`                      |
| Foreign Key | `{table_singular}_id`  | `event_id`, `order_id`           |
| Enum Value  | `SCREAMING_SNAKE_CASE` | `ACTIVE`, `PENDING_PAYMENT`      |

### Soft Delete Rules

- **Không bao giờ** DELETE vật lý
- Query mặc định: `WHERE delete_flag = 'ACTIVE'`
- Dùng `@Where(clause = "delete_flag = 'ACTIVE'")` annotation

### Cross-Service References

- **Không có** foreign key constraint giữa các database
- Reference columns chỉ mang tính logic
- Đồng bộ qua Kafka events hoặc REST/Feign calls

---

## 3. User Service - user_db

### Schema: user_profiles

```sql
CREATE TABLE user_profiles (
    id CHAR(36) PRIMARY KEY,  -- Set = Keycloak User UUID (not auto-generated)
    full_name VARCHAR(100) NOT NULL,
    phone VARCHAR(20) NULL,
    avatar_url VARCHAR(255) NULL,
    address VARCHAR(255) NULL,
    delete_flag ENUM('ACTIVE','DELETED') NOT NULL DEFAULT 'ACTIVE',
    created_by VARCHAR(255) NULL,
    created_at TIMESTAMP NULL,
    updated_by VARCHAR(255) NULL,
    updated_at TIMESTAMP NULL,
    INDEX idx_user_profiles_email (email)
);
```

| Column         | Type             | Constraints | Notes                               |
| -------------- | ---------------- | ----------- | ----------------------------------- |
| `id`         | `CHAR(36)`     | PK          | Set thủ công = Keycloak User UUID |
| `full_name`  | `VARCHAR(100)` | NOT NULL    |                                     |
| `phone`      | `VARCHAR(20)`  | NULL        |                                     |
| `avatar_url` | `VARCHAR(255)` | NULL        |                                     |
| `address`    | `VARCHAR(255)` | NULL        |                                     |

> **Lưu ý:** Username, password, email, role được quản lý bởi Keycloak, không lưu trong `user_db`.

---

## 4. Event Service - event_db

### Schema: locations

```sql
CREATE TABLE locations (
    id CHAR(36) PRIMARY KEY DEFAULT (UUID()),
    name VARCHAR(100) NOT NULL UNIQUE,
    delete_flag ENUM('ACTIVE','DELETED') NOT NULL DEFAULT 'ACTIVE',
    created_by VARCHAR(255) NULL,
    created_at TIMESTAMP NULL,
    updated_by VARCHAR(255) NULL,
    updated_at TIMESTAMP NULL
);
```

| Column   | Type             | Constraints      | Notes                    |
| -------- | ---------------- | ---------------- | ------------------------ |
| `name` | `VARCHAR(100)` | NOT NULL, UNIQUE | Tên thành phố/tỉnh   |
|          |                  |                  | VD: "Hà Nội", "TP.HCM" |

**Seed Data:**

- "Hà Nội"
- "TP.HCM"
- "Đà Nẵng"
- "Hải Phòng"
- "Cần Thơ"

---

### Schema: categories

```sql
CREATE TABLE categories (
    id CHAR(36) PRIMARY KEY DEFAULT (UUID()),
    name VARCHAR(100) NOT NULL UNIQUE,
    delete_flag ENUM('ACTIVE','DELETED') NOT NULL DEFAULT 'ACTIVE',
    created_by VARCHAR(255) NULL,
    created_at TIMESTAMP NULL,
    updated_by VARCHAR(255) NULL,
    updated_at TIMESTAMP NULL
);
```

| Column   | Type             | Constraints      | Notes          |
| -------- | ---------------- | ---------------- | -------------- |
| `name` | `VARCHAR(100)` | NOT NULL, UNIQUE | Tên danh mục |

**Seed Data:**

- "Nhạc sống"
- "Sân khấu & Nghệ thuật"
- "Thể thao"
- "Hội thảo"
- "Hội nghị"
- "Khác"

---

### Schema: events

```sql
CREATE TABLE events (
    id CHAR(36) PRIMARY KEY DEFAULT (UUID()),
    organizer_id VARCHAR(255) NOT NULL,  -- Keycloak User UUID
    title VARCHAR(255) NOT NULL,
    description TEXT NULL,
    category_id CHAR(36) NOT NULL,
    category VARCHAR(100) NOT NULL,  -- Denormalized for display
    location_id CHAR(36) NOT NULL,
    location VARCHAR(255) NOT NULL,  -- Full address (e.g., "23 Mễ Trì Hạ, Hà Nội")
    banner_url VARCHAR(255) NULL,
    start_time DATETIME NOT NULL,
    end_time DATETIME NOT NULL,
    status ENUM('DRAFT','PUBLISHED','CANCELLED') NOT NULL DEFAULT 'DRAFT',
    delete_flag ENUM('ACTIVE','DELETED') NOT NULL DEFAULT 'ACTIVE',
    created_by VARCHAR(255) NULL,
    created_at TIMESTAMP NULL,
    updated_by VARCHAR(255) NULL,
    updated_at TIMESTAMP NULL,
    FOREIGN KEY (category_id) REFERENCES categories(id),
    FOREIGN KEY (location_id) REFERENCES locations(id),
    INDEX idx_events_organizer (organizer_id),
    INDEX idx_events_category (category_id),
    INDEX idx_events_location (location_id),
    INDEX idx_events_status (status),
    INDEX idx_events_start_time (start_time)
);
```

| Column           | Type             | Constraints               | Notes                             |
| ---------------- | ---------------- | ------------------------- | --------------------------------- |
| `organizer_id` | `VARCHAR(255)` | NOT NULL                  | Keycloak User UUID                |
| `title`        | `VARCHAR(255)` | NOT NULL                  | Tên sự kiện                    |
| `description`  | `TEXT`         | NULL                      | Mô tả chi tiết                 |
| `category_id`  | `CHAR(36)`     | NOT NULL, FK              | Tham chiếu`categories.id`      |
| `category`     | `VARCHAR(100)` | NOT NULL                  | Tên denormalized, server resolve |
| `location_id`  | `CHAR(36)`     | NOT NULL, FK              | Thành phố/tỉnh                 |
| `location`     | `VARCHAR(255)` | NOT NULL                  | Địa chỉ chi tiết              |
| `banner_url`   | `VARCHAR(255)` | NULL                      | URL banner                        |
| `start_time`   | `DATETIME`     | NOT NULL                  | Thời gian bắt đầu             |
| `end_time`     | `DATETIME`     | NOT NULL                  | Thời gian kết thúc             |
| `status`       | `ENUM`         | NOT NULL, DEFAULT 'DRAFT' | Trạng thái sự kiện            |

---

### Schema: ticket_types

```sql
CREATE TABLE ticket_types (
    id CHAR(36) PRIMARY KEY DEFAULT (UUID()),
    event_id CHAR(36) NOT NULL,
    name VARCHAR(100) NOT NULL,
    price DECIMAL(12,2) NOT NULL,
    total_quantity INT NOT NULL,
    delete_flag ENUM('ACTIVE','DELETED') NOT NULL DEFAULT 'ACTIVE',
    created_by VARCHAR(255) NULL,
    created_at TIMESTAMP NULL,
    updated_by VARCHAR(255) NULL,
    updated_at TIMESTAMP NULL,
    FOREIGN KEY (event_id) REFERENCES events(id),
    INDEX idx_ticket_types_event (event_id)
);
```

| Column             | Type              | Constraints  | Notes                     |
| ------------------ | ----------------- | ------------ | ------------------------- |
| `event_id`       | `CHAR(36)`      | NOT NULL, FK | Tham chiếu`events.id`  |
| `name`           | `VARCHAR(100)`  | NOT NULL     | VD: "VIP", "Standard"     |
| `price`          | `DECIMAL(12,2)` | NOT NULL     | Giá vé                  |
| `total_quantity` | `INT`           | NOT NULL     | Tổng số vé phát hành |

> **Lưu ý:** Event Service **không** lưu `sold_quantity` real-time. Tồn kho tức thời chỉ tồn tại trên Redis.

---

### Schema: flash_sales

```sql
CREATE TABLE flash_sales (
    id CHAR(36) PRIMARY KEY DEFAULT (UUID()),
    event_id CHAR(36) NOT NULL UNIQUE,
    start_at DATETIME NOT NULL,
    end_at DATETIME NOT NULL,
    status ENUM('SCHEDULED','ACTIVE','ENDED') NOT NULL DEFAULT 'SCHEDULED',
    delete_flag ENUM('ACTIVE','DELETED') NOT NULL DEFAULT 'ACTIVE',
    created_by VARCHAR(255) NULL,
    created_at TIMESTAMP NULL,
    updated_by VARCHAR(255) NULL,
    updated_at TIMESTAMP NULL,
    FOREIGN KEY (event_id) REFERENCES events(id),
    INDEX idx_flash_sales_event (event_id),
    INDEX idx_flash_sales_status (status),
    INDEX idx_flash_sales_start (start_at)
);
```

| Column       | Type         | Constraints                   | Notes                         |
| ------------ | ------------ | ----------------------------- | ----------------------------- |
| `event_id` | `CHAR(36)` | NOT NULL, UNIQUE, FK          | 1 event chỉ có 1 flash sale |
| `start_at` | `DATETIME` | NOT NULL                      | Thời gian bắt đầu bán    |
| `end_at`   | `DATETIME` | NOT NULL                      | Thời gian kết thúc bán    |
| `status`   | `ENUM`     | NOT NULL, DEFAULT 'SCHEDULED' | Trạng thái flash sale       |

---

## 5. Ticket Service - Redis

Ticket Service **không có database MySQL**. Tồn kho vé hoàn toàn trên Redis.

### Redis Key Patterns

| Key Pattern                                   | Type             | TTL                  | Description                 |
| --------------------------------------------- | ---------------- | -------------------- | --------------------------- |
| `ticket:inventory:{eventId}:{ticketTypeId}` | STRING (integer) | No TTL               | Số vé còn lại           |
| `ticket:reservation:{reservationId}`        | HASH             | 180s (2min + buffer) | Reservation details         |
| `ticket:event:{eventId}:loaded`             | STRING (flag)    | No TTL               | Event inventory loaded flag |

### Key: ticket:inventory::

```lua
-- Lua Script: CHECK & DECREMENT (Atomic)
-- Input: KEYS[1] = inventory key, ARGV[1] = quantity
local current = redis.call('GET', KEYS[1])
if current == false then
    return -2  -- Key not found
end
if tonumber(current) >= tonumber(ARGV[1]) then
    redis.call('DECRBY', KEYS[1], ARGV[1])
    return tonumber(current) - tonumber(ARGV[1])
else
    return -1  -- Insufficient inventory
end
```

```lua
-- Lua Script: INCREMENT (Release tickets)
-- Input: KEYS[1] = inventory key, ARGV[1] = quantity
redis.call('INCRBY', KEYS[1], ARGV[1])
return redis.call('GET', KEYS[1])
```

### Key: ticket:reservation:

```json
{
  "userId": "keycloak-uuid",
  "eventId": "event-uuid",
  "ticketTypeId": "ticket-type-uuid",
  "quantity": 2,
  "unitPrice": 500.00,
  "reservedAt": "2024-01-15T10:30:00Z"
}
```

### Key: ticket:event::loaded

```
Value: "1" (flag)
```

> Đánh dấu đã load inventory cho event, tránh load trùng nếu scheduler chạy lại.

---

## 6. Order Service - order_db

### Schema: orders

```sql
CREATE TABLE orders (
    id CHAR(36) PRIMARY KEY DEFAULT (UUID()),
    reservation_id VARCHAR(255) NOT NULL UNIQUE,
    user_id VARCHAR(255) NOT NULL,
    event_id CHAR(36) NOT NULL,
    ticket_type_id CHAR(36) NOT NULL,
    quantity INT NOT NULL,
    unit_price DECIMAL(12,2) NOT NULL,
    total_amount DECIMAL(12,2) NOT NULL,
    payment_id CHAR(36) NULL,
    status ENUM('PENDING_PAYMENT','PAID','CANCELLED') NOT NULL DEFAULT 'PENDING_PAYMENT',
    delete_flag ENUM('ACTIVE','DELETED') NOT NULL DEFAULT 'ACTIVE',
    created_by VARCHAR(255) NULL,
    created_at TIMESTAMP NULL,
    updated_by VARCHAR(255) NULL,
    updated_at TIMESTAMP NULL,
    INDEX idx_orders_user (user_id),
    INDEX idx_orders_event (event_id),
    INDEX idx_orders_status (status),
    INDEX idx_orders_payment (payment_id),
    INDEX idx_orders_reservation (reservation_id)
);
```

| Column             | Type              | Constraints                         | Notes                           |
| ------------------ | ----------------- | ----------------------------------- | ------------------------------- |
| `reservation_id` | `VARCHAR(255)`  | NOT NULL, UNIQUE                    | Idempotency key                 |
| `user_id`        | `VARCHAR(255)`  | NOT NULL                            | Keycloak User UUID              |
| `event_id`       | `CHAR(36)`      | NOT NULL                            | Tham chiếu logic (khác DB)    |
| `ticket_type_id` | `CHAR(36)`      | NOT NULL                            | Tham chiếu logic (khác DB)    |
| `quantity`       | `INT`           | NOT NULL                            | Số lượng vé                 |
| `unit_price`     | `DECIMAL(12,2)` | NOT NULL                            | Giá từng vé                  |
| `total_amount`   | `DECIMAL(12,2)` | NOT NULL                            | = quantity × unit_price        |
| `payment_id`     | `CHAR(36)`      | NULL                                | Gán sau khi payment khởi tạo |
| `status`         | `ENUM`          | NOT NULL, DEFAULT 'PENDING_PAYMENT' |                                 |

### Order Status Flow

```
PENDING_PAYMENT ──(payment-success)──▶ PAID
       │
       └──(payment-failed)──▶ CANCELLED
```

---

## 7. Payment Service - payment_db

### Schema: payments

```sql
CREATE TABLE payments (
    id CHAR(36) PRIMARY KEY DEFAULT (UUID()),
    order_id CHAR(36) NOT NULL UNIQUE,
    user_id VARCHAR(255) NOT NULL,
    amount DECIMAL(12,2) NOT NULL,
    payment_method ENUM('CARD','MOMO','VNPAY','BANK_TRANSFER') NOT NULL,
    status ENUM('PENDING','SUCCESS','FAILED') NOT NULL DEFAULT 'PENDING',
    external_transaction_id VARCHAR(255) NULL,
    expires_at DATETIME NOT NULL,
    failed_reason VARCHAR(255) NULL,
    delete_flag ENUM('ACTIVE','DELETED') NOT NULL DEFAULT 'ACTIVE',
    created_by VARCHAR(255) NULL,
    created_at TIMESTAMP NULL,
    updated_by VARCHAR(255) NULL,
    updated_at TIMESTAMP NULL,
    INDEX idx_payments_order (order_id),
    INDEX idx_payments_user (user_id),
    INDEX idx_payments_status (status),
    INDEX idx_payments_expires (expires_at)
);
```

| Column                      | Type              | Constraints                 | Notes                       |
| --------------------------- | ----------------- | --------------------------- | --------------------------- |
| `order_id`                | `CHAR(36)`      | NOT NULL, UNIQUE            | 1 order = 1 payment         |
| `user_id`                 | `VARCHAR(255)`  | NOT NULL                    | Keycloak User UUID          |
| `amount`                  | `DECIMAL(12,2)` | NOT NULL                    | Số tiền thanh toán       |
| `payment_method`          | `ENUM`          | NOT NULL                    | Phương thức thanh toán  |
| `status`                  | `ENUM`          | NOT NULL, DEFAULT 'PENDING' |                             |
| `external_transaction_id` | `VARCHAR(255)`  | NULL                        | Mã giao dịch bên thứ 3  |
| `expires_at`              | `DATETIME`      | NOT NULL                    | created_at + 2 phút        |
| `failed_reason`           | `VARCHAR(255)`  | NULL                        | `DECLINED` \| `TIMEOUT` |

### Payment Status Flow

```
PENDING ──(callback: success)──▶ SUCCESS
   │
   └──(callback: fail / timeout)──▶ FAILED
```

---

## 8. Notification Service - notification_db

### Schema: notifications

```sql
CREATE TABLE notifications (
    id CHAR(36) PRIMARY KEY DEFAULT (UUID()),
    order_id CHAR(36) NOT NULL,
    type ENUM('TICKET_QR','ORDER_CANCELLED') NOT NULL,
    channel ENUM('EMAIL') NOT NULL DEFAULT 'EMAIL',
    status ENUM('QUEUED','QUEUE_FAILED') NOT NULL,
    error_message VARCHAR(500) NULL,
    queued_at DATETIME NULL,
    delete_flag ENUM('ACTIVE','DELETED') NOT NULL DEFAULT 'ACTIVE',
    created_by VARCHAR(255) NULL,
    created_at TIMESTAMP NULL,
    updated_by VARCHAR(255) NULL,
    updated_at TIMESTAMP NULL,
    UNIQUE KEY uk_notifications_order_type (order_id, type),
    INDEX idx_notifications_order (order_id),
    INDEX idx_notifications_status (status)
);
```

| Column            | Type             | Constraints               | Notes                           |
| ----------------- | ---------------- | ------------------------- | ------------------------------- |
| `order_id`      | `CHAR(36)`     | NOT NULL                  | Idempotency key (với`type`)  |
| `type`          | `ENUM`         | NOT NULL                  | Loại thông báo               |
| `channel`       | `ENUM`         | NOT NULL, DEFAULT 'EMAIL' | Dự phòng SMS/Push             |
| `status`        | `ENUM`         | NOT NULL                  | Kết quả publish SQS           |
| `error_message` | `VARCHAR(500)` | NULL                      | Lỗi SQS                        |
| `queued_at`     | `DATETIME`     | NULL                      | Thời điểm queue thành công |

### Idempotency

- Constraint `UNIQUE (order_id, type)` đảm bảo không publish trùng
- Dùng làm idempotency key khi Kafka message được re-deliver

---

## 9. Migration Standards

### File Naming

```
V{n}_{YYYYMMDDHHmm}_{description}.sql
```

Ví dụ:

- `V1_202401150000_create_user_profiles_table.sql`
- `V2_202401151200_add_index_to_events.sql`

### changelog.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<databaseChangeLog
    xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
        http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.30.xsd">

    <include file="db/sources/V1_202401150000_create_user_profiles_table.sql"
             relativeToChangelogFile="false"/>

</databaseChangeLog>
```

### ChangeSet Pattern

```sql
-- liquibase formatted sql
-- changeset author:id
-- onValidationFail: MARK_RAN

CREATE TABLE user_profiles (
    id CHAR(36) PRIMARY KEY,
    full_name VARCHAR(100) NOT NULL,
    ...
);

-- rollback: DROP TABLE user_profiles;
```

### Rollback

```sql
-- rollback: DROP TABLE user_profiles;
-- rollback: DROP INDEX idx_user_profiles_email;
```

### Nghiêm cấm

- **Không** sửa changeset đã commit
- **Không** `DROP TABLE` trong migration thực tế
- Chỉ thêm changeset mới
