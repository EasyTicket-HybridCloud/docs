# Công nghệ sử dụng

## Mục lục

1. [Tổng quan](#1-tổng-quan)
2. [Dependency Versions](#2-dependency-versions)
3. [Tiêu chuẩn Logging](#3-tiêu-chuẩn-logging)
4. [Observability (OpenTelemetry)](#4-observability-opentelemetry)
5. [Exception Handling](#5-exception-handling)
6. [API Response chuẩn](#6-api-response-chuẩn)
7. [Database & Liquibase](#7-database--liquibase)
8. [Configuration Management](#8-configuration-management)
9. [Security](#9-security)
10. [Kafka Standards](#10-kafka-standards)
11. [API Standards](#11-api-standards)
12. [Inter-Service Communication](#12-inter-service-communication)

---

## 1. Tổng quan

EasyTicket sử dụng kiến trúc Microservices với Java 21 và Spring Boot 4.0.7. Mỗi service có database riêng (Database per Service) và giao tiếp qua REST (đồng bộ) và Kafka (bất đồng bộ).

## 2. Dependency Versions

| Dependency | Version |
|------------|---------|
| Spring Boot | `4.0.7` |
| Java | `21` |
| Build tool | Maven (multi-module per service) |
| Lombok | `1.18.30` |
| MapStruct | `1.6.3` |
| Liquibase | `4.30.0` |
| Keycloak Admin Client | `26.0.4` |
| Logstash Logback Encoder | `7.4` |
| Spring Cloud OpenFeign | `4.2.1` |
| Spring Security | Managed by Spring Boot BOM |
| MySQL Connector | Managed by Spring Boot BOM |

> **Quy tắc:** Version được quản lý tập trung trong Parent POM `<dependencyManagement>` của mỗi service. Không pin version lẻ tẻ trong module con.

## 3. Tiêu chuẩn Logging

### Công cụ
- **SLF4J + Logback** (mặc định Spring Boot)
- **Không bao giờ** dùng `System.out.println`

### Log Levels

| Level | Khi nào dùng |
|-------|--------------|
| `ERROR` | Exception không recover được, lỗi hệ thống, tích hợp thất bại |
| `WARN` | Bất thường nhưng hệ thống vẫn chạy, retry, timeout |
| `INFO` | Business event quan trọng (register, order created, payment success) |
| `DEBUG` | Chi tiết kỹ thuật, chỉ dùng dev |
| `TRACE` | SQL params, raw HTTP body |

### Quy tắc log

```java
@Slf4j
@Service
public class UsersServiceImpl implements UsersService {
    public void register(String userId, String email) {
        log.info("User registered successfully. userId={}, email={}", userId, email);
    }
}
```

**Phải kèm context** (`userId`, `orderId`, ...) — không log chay.

### Nghiêm cấm log

- Password, client_secret, private key
- Access/refresh token
- Số thẻ tín dụng
- PII đầy đủ (CCCD, SĐT)

## 4. Observability (OpenTelemetry)

### Pipeline

```
Spring Boot → OTel Collector (OTLP) → Traces → APM Server → Elasticsearch → Kibana
                                      → Logs   → Logstash  → Elasticsearch → Kibana
                                      → Metrics → Elasticsearch → Kibana
```

### Yêu cầu bắt buộc

- `traceId`/`spanId` tự động inject vào MDC qua `micrometer-tracing`
- Trace context propagate qua HTTP header W3C (`traceparent`) tự động với Feign
- RestTemplate cần bean `ObservationRestTemplateCustomizer`
- Kafka dùng `TracingKafkaProducerFactory`/`ConsumerFactory`
- `management.tracing.sampling.probability`: `1.0` ở local/dev, `0.1` ở prod
- Mọi service expose `/actuator/prometheus` để OTel Collector scrape
- `logback-spring.xml` gửi log JSON qua `LogstashTcpSocketAppender` kèm `traceId`/`spanId`

## 5. Exception Handling

### Hierarchy bắt buộc

```
RuntimeException
└── BusinessException (base, đặt trong business/exception/)
    ├── ResourceNotFoundException     # 404
    ├── ConflictException             # 409
    ├── ValidationException           # 400 (business validation)
    └── {Domain}Exception             # ví dụ TicketSoldOutException
```

### Quy tắc bắt buộc

- **Không** bắt exception rồi nuốt im lặng – luôn log hoặc rethrow
- **Không** throw `RuntimeException` trần – dùng `BusinessException` hoặc subclass cụ thể
- Infrastructure exception (DB, Kafka, HTTP timeout) → bắt ở Service layer, wrap thành `BusinessException` với message thân thiện, log bản gốc ở `ERROR`
- **Không** để stack trace lộ trong response body – chỉ trả `errorCode` + `message`

### GlobalExceptionHandler

Xử lý tập trung qua `@RestControllerAdvice` đặt trong module `application`.

## 6. API Response chuẩn

### ApiResponse class

```java
@Data @Builder @NoArgsConstructor @AllArgsConstructor
public class ApiResponse<T> {
    private boolean success;
    private String errorCode;   // null khi success
    private String message;
    private T data;             // null khi error
    private String traceId;     // inject từ MDC

    public static <T> ApiResponse<T> ok(T data) { ... }
    public static <T> ApiResponse<T> ok(T data, String message) { ... }
    public static ApiResponse<Void> error(String errorCode, String message) { ... }
}
```

### Quy tắc bắt buộc

- Controller luôn trả `ResponseEntity<ApiResponse<T>>`
- **Không bao giờ** trả Entity hay kiểu thô trực tiếp
- Vị trí: `com.easytickets.common.dto.ApiResponse`

## 7. Database & Liquibase

### Nguyên tắc Database per Service

- Mỗi service sở hữu database riêng
- **Không** có foreign key vật lý giữa hai database khác nhau
- Các cột tham chiếu cross-service chỉ là ID tham chiếu logic

### Migration

- Module `{ServiceName}-migration` chạy **độc lập** với application
- `spring.liquibase.enabled=false` ở application, `=true` ở migration
- File SQL: `V{n}_{YYYYMMDDHHmm}_{description}.sql` đặt tại `db/sources/`
- `changelog.xml` chỉ include, không viết SQL trực tiếp
- ChangeSet ID = tên file (bỏ `.sql`); luôn thêm `onValidationFail="MARK_RAN"`

### Nghiêm cấm

- **Không** sửa changeSet đã commit – chỉ thêm mới
- **Không** `DROP TABLE` trong migration thực tế

### Naming Convention

- Table: `snake_case` số nhiều (`ticket_orders`)
- Column: `snake_case`
- Primary key: `CHAR(36) DEFAULT (UUID())`
- Enum value: `SCREAMING_SNAKE_CASE`

### Audit Fields (bắt buộc cho mọi Entity)

```java
@MappedSuperclass
@Data
@EntityListeners(AuditingEntityListener.class)
public abstract class BaseEntity {
    @Id @GeneratedValue(strategy = GenerationType.UUID)
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

### Soft Delete

- **Không bao giờ** xóa vật lý dữ liệu
- "Xóa" = set `delete_flag = DELETED`
- Query mặc định chỉ lấy `delete_flag = ACTIVE` (dùng `@Where` annotation)
- `createdBy`/`updatedBy` chỉ lưu Keycloak User ID (JWT claim `sub`)

## 8. Configuration Management

### Nguyên tắc

- **Không hardcode** credentials/secrets trong code hay `application.yaml`
- Luôn dùng `${ENV_VAR:default}` (default chỉ dùng local dev)
- Dùng `@ConfigurationProperties` thay vì `@Value` rời rạc

### Profiles

| Profile | Mục đích |
|---------|----------|
| `local` | Dev máy cá nhân |
| `dev` | Staging |
| `prod` | Production |

File cấu hình: `application-{profile}.yaml`

### Prod requirements

- `hibernate.show_sql=false`
- Log level `WARN/INFO`
- Tracing sampling `0.1`

## 9. Security

### OAuth2 Resource Server

- Mọi service (trừ public API) là **OAuth2 Resource Server**
- Xác thực JWT từ Keycloak
- Danh sách endpoint public cấu hình trong `application.yaml` (`url.permit`)

### Role Extraction

```java
// Role extract từ claim: resource_access.{client-id}.roles
// Prefix ROLE_ để tương thích Spring Security
@PreAuthorize("hasRole('BUYER')")
@PreAuthorize("hasRole('ORGANIZER')")
@PreAuthorize("hasRole('ADMIN')")
```

### Internal Calls

- Gọi service nội bộ: dùng Keycloak Client Credentials flow lấy token
- Truyền `Authorization: Bearer {token}`
- **Không** bypass authentication cho internal call

## 10. Kafka Standards

### Message Format

- **JSON** format
- Định nghĩa class trong `business/dto/event/`

### Idempotency (bắt buộc)

- Consumer phải kiểm tra message đã xử lý chưa
- Dùng `orderId`/`eventId`/`reservationId` làm idempotency key lưu DB

### Dead Letter Queue

- Message lỗi sau N lần retry chuyển sang topic `{topic}.DLT`

### Message Key

- Message key = ID entity chính
- Đảm bảo ordering theo partition

### Consumer Rules

- **Không** block consumer bằng long-running operation

## 11. API Standards

### Versioning

- Bắt buộc: `api/v1/...`
- Breaking change → tạo controller `api/v2/...`
- Giữ `v1` backward-compatible

### HTTP Methods

| Method | Mục đích |
|--------|----------|
| `POST` | Tạo mới |
| `GET` | Đọc |
| `PUT` | Cập nhật toàn bộ |
| `PATCH` | Cập nhật một phần |
| `DELETE` | Xóa (soft delete) |

### Health Check

- Mọi service expose `/actuator/health` cho Kubernetes liveness/readiness probe

## 12. Inter-Service Communication

### OpenFeign

```java
// Client interface đặt trong business/client/
@FeignClient(name = "event-service", url = "${services.event-service.url}")
public interface EventServiceClient {
    @GetMapping("/api/v1/events/{id}")
    ApiResponse<EventResponse> getEvent(@PathVariable String id);
}
```

### Nguyên tắc

- URL service khác luôn cấu hình qua env var (`${EVENT_SERVICE_URL:...}`)
- **Không** hardcode URL

---

## Cấu trúc Module (mỗi service)

```
{ServiceName}/
├── {ServiceName}-application      # Entry point, Controller, Security Config
├── {ServiceName}-business         # Business logic, Service, DTO, Port (interface repo)
├── {ServiceName}-common           # Shared utilities, constants, base classes
├── {ServiceName}-infratructures   # JPA Entity, Repository impl, Mapper, JpaConfig
├── {ServiceName}-migration        # Liquibase migration scripts (chạy độc lập)
├── {ServiceName}-worker           # Kafka consumers, scheduled jobs, async workers
└── pom.xml                        # Parent POM – quản lý dependency versions
```

### Dependency Direction

```
application → business ← infratructures
application → infratructures
worker      → business
worker      → infratructures
```

### Package Structure

| Module | Package | Nội dung |
|--------|---------|----------|
| application | `com.easytickets.application` | Controller, SecurityConfig, Application.java |
| business | `com.easytickets.business` | Service interface, DTO, Port interface, Exception |
| common | `com.easytickets.common` | BaseEntity, ApiResponse, constants |
| infratructures | `com.easytickets.infratructures` | JPA Entity, Repository, Mapper, JpaConfig |
| worker | `com.easytickets.worker` | Kafka consumers, producers, schedulers |
