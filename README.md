# EasyTicket - Tài liệu Dự án

> Nền tảng đặt vé sự kiện trực tuyến, kiến trúc Microservices, chuyên xử lý **Flash Sale** lưu lượng cực cao.

## 🎯 Cam kết cốt lõi

1. **Không sập web** dưới tải cao
2. **Không over-selling vé** - tồn kho được quản lý nguyên tử trên Redis
3. **Phản hồi tính bằng mili-giây** - tối ưu cho trải nghiệm "săn vé"

## 📚 Mục lục tài liệu

| Tài liệu | Mô tả |
|-----------|-------|
| [Công nghệ sử dụng](./pages/tech-stack.md) | Chi tiết tech stack, dependency versions, tiêu chuẩn kỹ thuật |
| [Luồng nghiệp vụ](./pages/spec-development.md) | 9 luồng nghiệp vụ chính của hệ thống |
| [Yêu cầu triển khai](./pages/spec-infrastructure.md) | Kiến trúc hệ thống, Docker, Kubernetes, observability |
| [Thiết kế Database](./pages/database-design.md) | Database schema, table design, migration standards |
| [Thiết kế API](./pages/endpoint-design.md) | Tất cả REST API endpoints |
| [Hướng dẫn triển khai](./pages/deployment.md) | Cài đặt, chạy local, deploy lên cloud |

## 🏗️ Kiến trúc tổng quan

```
                                  ┌─────────────────┐
                                  │   API Gateway    │
                                  │ (AWS API Gateway │
                                  │  dev/prod; NGINX │
                                  │  giả lập ở local)│
                                  └────────┬─────────┘
                                           │
        ┌───────────────┬─────────────────┼───────────────┬───────────────┐
        ▼               ▼                 ▼               ▼               ▼
 ┌─────────────┐  ┌──────────┐    ┌─────────────┐  ┌──────────┐   ┌──────────────┐
 │Event Service│  │  Ticket  │    │Order Service│  │ Payment  │   │ User Service │
 │  (Catalog)  │  │ Service  │    │             │  │ Service  │   │  (Profile)   │
 └──────┬──────┘  └────┬─────┘    └──────┬──────┘  └────┬─────┘   └──────┬───────┘
        │               │                │              │                │
        ▼               ▼                ▼              ▼                ▼
 ┌─────────────┐  ┌──────────┐    ┌─────────────┐  ┌──────────┐   ┌──────────────┐
 │    MySQL    │  │  Redis   │    │    MySQL    │  │  MySQL   │   │    MySQL     │
 │  + Redis    │  │  + Kafka │    │  + Kafka    │  │ + Kafka  │   │  (user_db)   │
 │  (event_db) │  │(tồn kho) │    │ (order_db)  │  │(payment_ │   └──────┬───────┘
 └─────────────┘  └──────────┘    └─────────────┘  │   db)    │          │
                          │                        └──────────┘          ▼
                          ▼                              │        ┌────────────┐
                   ┌────────────────┐              (Kafka: payment│  Keycloak  │
                   │  Notification  │◄─────────────  -success /   │(OAuth2/JWT,│
                   │    Service     │                 -failed)     │ Admin API) │
                   │  (AWS SES)     │                          └────────────┘
                   └────────────────┘
```

## 👥 Đối tượng người dùng

| Role | Mô tả |
|------|-------|
| **Organizer** | Tạo/quản lý event, số lượng vé, mức giá, lên lịch flash sale, xem dashboard doanh thu |
| **Ticket Buyer** | Duyệt/tìm event, tham gia flash sale, thanh toán, nhận vé QR qua email |
| **System Admin** | Duyệt/khóa tài khoản organizer, giám sát hệ thống |

## 📊 Các Microservices

| Service | Vai trò | Database |
|---------|---------|----------|
| **Event Service** | CRUD sự kiện/loại vé/giá, lên lịch flash sale, cache Redis | MySQL (`event_db`) |
| **Ticket Service** | Xử lý mua vé qua Redis Lua Script, quản lý tồn kho Redis | Redis (nguồn sự thật) |
| **Order Service** | Tạo & quản lý order, consume Kafka events | MySQL (`order_db`) |
| **Payment Service** | Xử lý thanh toán, timeout 2 phút | MySQL (`payment_db`) |
| **Notification Service** | Render vé QR, gửi email qua AWS SES | MySQL (`notification_db`) |
| **User Service** | Cầu nối Keycloak, quản lý user profiles | MySQL (`user_db`) |

## 🔧 Cách sử dụng tài liệu

1. **Nhà phát triển mới**: Bắt đầu với [Công nghệ sử dụng](./pages/tech-stack.md) → [./pages/Luồng nghiệp vụ](spec-development.md)
2. **DevOps/Deployment**: Xem [Yêu cầu triển khai](./pages/spec-infrastructure.md) → [Hướng dẫn triển khai](./pages/deployment.md)
3. **Backend Developer**: [Thiết kế Database](./pages/database-design.md) → [Thiết kế API](./pages/endpoint-design.md)
4. **Đóng góp code**: Tham khảo [Công nghệ sử dụng](./pages/tech-stack.md) để nắm coding standards

## 📁 Cấu trúc thư mục dự án

```
EasyTicket/
├── docs/                          # Tài liệu dự án
│   ├── README.md                  # Trang tổng quan (bạn đang xem)
│   ├── tech-stack.md              # Công nghệ sử dụng
│   ├── spec-development.md        # Luồng nghiệp vụ
│   ├── spec-infrastructure.md     # Yêu cầu triển khai
│   ├── database-design.md         # Thiết kế database
│   ├── endpoint-design.md         # Thiết kế API
│   └── deployment.md              # Hướng dẫn triển khai
├── EventService/                  # Event microservice (6-module Maven)
├── TicketService/                 # Ticket microservice
├── OrderService/                  # Order microservice
├── PaymentService/                # Payment microservice
├── NotificationService/           # Notification microservice
├── UserService/                   # User microservice
├── infra/                        # Infrastructure configs (Docker, NGINX, OTel)
└── docker-compose.yml            # Local infrastructure stack
```

## 🚀 Bắt đầu nhanh

```bash
# Clone repository
git clone <repository-url>
cd EasyTicket

# Khởi động hạ tầng local
docker compose up -d

# Build một service
cd EventService && ./mvnw clean package

# Chạy service
./mvnw spring-boot:run -pl EventService-application
```

Truy cập API Gateway tại `http://localhost:8000` để xem Swagger UI tổng hợp.

## 📝 Ghi chú

> **Trạng thái triển khai hiện tại:** Phần lớn service đang ở dạng scaffold (khung module Maven, chưa có business logic hoàn chỉnh). Các endpoint liệt kê trong tài liệu là **thiết kế API mục tiêu** của hệ thống.
