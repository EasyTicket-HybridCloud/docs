# Hướng dẫn triển khai

## Mục lục

1. [Yêu cầu hệ thống](#1-yêu-cầu-hệ-thống)
2. [Cài đặt local (Development)](#2-cài-đặt-local-development)
3. [Cấu hình từng service](#3-cấu-hình-từng-service)
4. [Chạy services](#4-chạy-services)
5. [Triển khai Kubernetes](#5-triển-khai-kubernetes)
6. [Triển khai Production](#6-triển-khai-production)
7. [Monitoring & Troubleshooting](#7-monitoring--troubleshooting)

---

## 1. Yêu cầu hệ thống

### Software Requirements

| Software | Version | Mục đích |
|----------|---------|----------|
| Java | 21+ | Runtime |
| Maven | 3.9+ | Build tool |
| Docker | 24+ | Container runtime |
| Docker Compose | 2.20+ | Local infrastructure |
| Git | 2.40+ | Version control |

### Hardware Requirements (Local)

| Resource | Minimum | Recommended |
|----------|---------|-------------|
| RAM | 8 GB | 16 GB |
| CPU | 4 cores | 8 cores |
| Disk | 20 GB free | 50 GB SSD |

### Infrastructure Requirements (Production)

| Component | Specification |
|-----------|---------------|
| API Gateway | AWS API Gateway |
| Database | AWS RDS MySQL 8.0 (multi-AZ) |
| Cache | AWS ElastiCache Redis 7 (cluster mode) |
| Message Broker | AWS MSK (Kafka) |
| Identity | AWS Keycloak hoặc AWS Cognito |
| Email | AWS SES |
| Container | AWS EKS (Kubernetes) |
| Monitoring | CloudWatch, X-Ray, ELK Stack |

---

## 2. Cài đặt local (Development)

### 2.1 Clone Repository

```bash
git clone <repository-url>
cd EasyTicket
```

### 2.2 Khởi động Infrastructure

```bash
# Khởi động toàn bộ infrastructure (MySQL, Redis, Kafka, Keycloak, ELK, OTel, NGINX)
docker compose up -d

# Kiểm tra trạng thái các container
docker compose ps

# Xem logs
docker compose logs -f
```

### 2.3 Kiểm tra Services

```bash
# MySQL
mysql -h localhost -u root -ppassword -e "SHOW DATABASES;"

# Redis
redis-cli -h localhost -a easyticket_redis ping

# Kafka
docker compose exec kafka kafka-topics.sh --bootstrap-server localhost:9092 --list

# Keycloak
curl http://localhost:8080/health/ready

# Kibana
curl http://localhost:5601/api/status

# NGINX Gateway
curl http://localhost:8000/healthz
```

### 2.4 Truy cập UI

| Service | URL |
|---------|-----|
| API Gateway | http://localhost:8000 |
| Swagger UI | http://localhost:8000/ |
| Keycloak Admin | http://localhost:8080 (admin/admin) |
| Kibana | http://localhost:5601 |
| Kafka UI | http://localhost:8086 |

---

## 3. Cấu hình từng service

### 3.1 User Service

```yaml
# UserService/UserService-application/src/main/resources/application.yaml
spring:
  application:
    name: user-service
  datasource:
    url: jdbc:mysql://localhost:3306/user_db?useSSL=false&allowPublicKeyRetrieval=true&serverTimezone=UTC
    username: easyticket
    password: easyticket_password
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: http://localhost:8080/realms/SonNS_realm
  liquibase:
    enabled: false  # Migration chạy riêng

server:
  port: 8081

keycloak:
  server-url: http://localhost:8080
  realm: SonNS_realm
  client-id: quan_ly_ke_toan
  client-secret: ${KEYCLOAK_CLIENT_SECRET:your-client-secret}

services:
  user-service:
    url: http://localhost:8081
```

### 3.2 Event Service

```yaml
# EventService/EventService-application/src/main/resources/application.yaml
spring:
  application:
    name: event-service
  datasource:
    url: jdbc:mysql://localhost:3306/event_db?useSSL=false&allowPublicKeyRetrieval=true&serverTimezone=UTC
    username: easyticket
    password: easyticket_password
  data:
    redis:
      host: localhost
      port: 6379
      password: easyticket_redis
  liquibase:
    enabled: false

server:
  port: 8082

keycloak:
  server-url: http://localhost:8080
  realm: SonNS_realm

services:
  event-service:
    url: http://localhost:8082
```

### 3.3 Ticket Service

```yaml
# TicketService/TicketService-application/src/main/resources/application.yaml
spring:
  application:
    name: ticket-service
  data:
    redis:
      host: localhost
      port: 6379
      password: easyticket_redis
  kafka:
    bootstrap-servers: localhost:9092
    consumer:
      group-id: ticket-service-group
      auto-offset-reset: earliest

server:
  port: 8083

keycloak:
  server-url: http://localhost:8080
  realm: SonNS_realm

services:
  event-service:
    url: http://localhost:8082
  ticket-service:
    url: http://localhost:8083
```

### 3.4 Order Service

```yaml
# OrderService/OrderService-application/src/main/resources/application.yaml
spring:
  application:
    name: order-service
  datasource:
    url: jdbc:mysql://localhost:3306/order_db?useSSL=false&allowPublicKeyRetrieval=true&serverTimezone=UTC
    username: easyticket
    password: easyticket_password
  kafka:
    bootstrap-servers: localhost:9092
    consumer:
      group-id: order-service-group
      auto-offset-reset: earliest
  liquibase:
    enabled: false

server:
  port: 8084

keycloak:
  server-url: http://localhost:8080
  realm: SonNS_realm

services:
  order-service:
    url: http://localhost:8084
```

### 3.5 Payment Service

```yaml
# PaymentService/PaymentService-application/src/main/resources/application.yaml
spring:
  application:
    name: payment-service
  datasource:
    url: jdbc:mysql://localhost:3306/payment_db?useSSL=false&allowPublicKeyRetrieval=true&serverTimezone=UTC
    username: easyticket
    password: easyticket_password
  kafka:
    bootstrap-servers: localhost:9092
    consumer:
      group-id: payment-service-group
      auto-offset-reset: earliest
  liquibase:
    enabled: false

server:
  port: 8085

keycloak:
  server-url: http://localhost:8080
  realm: SonNS_realm

payment:
  gateway:
    callback-secret: ${PAYMENT_GATEWAY_SECRET:your-secret}
  timeout-minutes: 2

services:
  payment-service:
    url: http://localhost:8085
```

### 3.6 Notification Service

```yaml
# NotificationService/NotificationService-application/src/main/resources/application.yaml
spring:
  application:
    name: notification-service
  datasource:
    url: jdbc:mysql://localhost:3306/notification_db?useSSL=false&allowPublicKeyRetrieval=true&serverTimezone=UTC
    username: easyticket
    password: easyticket_password
  kafka:
    bootstrap-servers: localhost:9092
    consumer:
      group-id: notification-service-group
      auto-offset-reset: earliest
  liquibase:
    enabled: false
  aws:
    sqs:
      ticket-email-queue-url: ${TICKET_EMAIL_QUEUE_URL:}

server:
  port: 8087

services:
  notification-service:
    url: http://localhost:8087
```

---

## 4. Chạy Services

### 4.1 Build Services

```bash
# Build tất cả services
for service in UserService EventService TicketService OrderService PaymentService NotificationService; do
  cd $service
  ./mvnw clean package -DskipTests
  cd ..
done

# Hoặc build từng service
cd EventService
./mvnw clean package -DskipTests
```

### 4.2 Chạy Migration (trước khi chạy application)

```bash
# Chạy migration cho từng service
cd UserService
./mvnw spring-boot:run -pl UserService-migration

cd EventService
./mvnw spring-boot:run -pl EventService-migration

cd OrderService
./mvnw spring-boot:run -pl OrderService-migration

cd PaymentService
./mvnw spring-boot:run -pl PaymentService-migration

cd NotificationService
./mvnw spring-boot:run -pl NotificationService-migration
```

### 4.3 Chạy Applications

```bash
# Chạy từng service (mỗi terminal một service)
cd UserService
./mvnw spring-boot:run -pl UserService-application

cd EventService
./mvnw spring-boot:run -pl EventService-application

cd TicketService
./mvnw spring-boot:run -pl TicketService-application

cd OrderService
./mvnw spring-boot:run -pl OrderService-application

cd PaymentService
./mvnw spring-boot:run -pl PaymentService-application

cd NotificationService
./mvnw spring-boot:run -pl NotificationService-application
```

### 4.4 Test API qua Gateway

```bash
# Đăng ký buyer
curl -X POST http://localhost:8000/api/v1/users/register/buyer \
  -H "Content-Type: application/json" \
  -d '{
    "username": "testbuyer",
    "password": "TestPass123",
    "email": "buyer@example.com",
    "fullName": "Test Buyer"
  }'

# Đăng nhập
curl -X POST http://localhost:8000/api/v1/users/login \
  -H "Content-Type: application/json" \
  -d '{
    "username": "testbuyer",
    "password": "TestPass123"
  }'

# Lấy danh sách sự kiện (public)
curl http://localhost:8000/api/v1/events
```

---

## 5. Triển khai Kubernetes

### 5.1 Chuẩn bị EKS Cluster

```bash
# Tạo EKS cluster
eksctl create cluster \
  --name easyticket-cluster \
  --region ap-southeast-1 \
  --version 1.29 \
  --nodegroup-name linux-nodes \
  --node-type t3.medium \
  --nodes 3 \
  --nodes-min 1 \
  --nodes-max 10

# Cấu hình kubectl
aws eks --region ap-southeast-1 update-kubeconfig --name easyticket-cluster
```

### 5.2 Build Docker Images

```bash
# Login ECR
aws ecr get-login-password --region ap-southeast-1 | \
  docker login --username AWS --password-stdin <account-id>.dkr.ecr.ap-southeast-1.amazonaws.com

# Build và push images
export REGISTRY=<account-id>.dkr.ecr.ap-southeast-1.amazonaws.com

docker build -t $REGISTRY/event-service:latest ./EventService
docker push $REGISTRY/event-service:latest

docker build -t $REGISTRY/ticket-service:latest ./TicketService
docker push $REGISTRY/ticket-service:latest

# ... các service khác tương tự
```

### 5.3 Deploy Services

```bash
# Tạo namespace
kubectl create namespace easyticket

# Deploy Event Service
kubectl apply -f k8s/event-service-deployment.yaml -n easyticket

# Deploy Ticket Service
kubectl apply -f k8s/ticket-service-deployment.yaml -n easyticket

# Deploy Order Service
kubectl apply -f k8s/order-service-deployment.yaml -n easyticket

# Deploy Payment Service
kubectl apply -f k8s/payment-service-deployment.yaml -n easyticket

# Deploy Notification Service
kubectl apply -f k8s/notification-service-deployment.yaml -n easyticket

# Deploy User Service
kubectl apply -f k8s/user-service-deployment.yaml -n easyticket

# Deploy Ingress
kubectl apply -f k8s/ingress.yaml -n easyticket
```

### 5.4 Verify Deployment

```bash
# Kiểm tra pods
kubectl get pods -n easyticket

# Kiểm tra services
kubectl get svc -n easyticket

# Kiểm tra logs
kubectl logs -f deployment/event-service -n easyticket

# Kiểm tra health
kubectl exec -it deployment/event-service -n easyticket -- curl localhost:8080/actuator/health
```

### 5.5 Scale Services

```bash
# Manual scale
kubectl scale deployment event-service --replicas=5 -n easyticket

# Horizontal Pod Autoscaler (tự động)
kubectl autoscale deployment ticket-service \
  --cpu-percent=70 \
  --min=3 \
  --max=10 \
  -n easyticket
```

---

## 6. Triển khai Production

### 6.1 AWS Infrastructure

```bash
# Tạo RDS instances cho mỗi service
aws rds create-db-instance \
  --db-instance-identifier easyticket-user-db \
  --db-instance-class db.t3.medium \
  --engine mysql \
  --engine-version 8.0.35 \
  --allocated-storage 100 \
  --master-username easyticket \
  --master-user-password $DB_PASSWORD

# Tạo ElastiCache Redis cluster
aws elasticache create-replication-group \
  --replication-group-id easyticket-redis \
  --engine redis \
  --engine-version 7.0 \
  --num-node-groups 3 \
  --replicas-per-node-group 2 \
  --node-group-configuration \
    "ReplicaCount=2,PrimaryAvailabilityZone=ap-southeast-1a,ReplicaAvailabilityZones=[ap-southeast-1b,ap-southeast-1c]"

# Tạo MSK Kafka cluster
aws kafka create-cluster \
  --cluster-name easyticket-kafka \
  --broker-node-group-configuration \
    "InstanceType=kafka.m5.large,ClientSubnets=subnet-xxx,SecurityGroups=sg-xxx,StorageInfo={FilerSizeInGB=100}}" \
  --kafka-version 3.6.0 \
  --number-of-broker-nodes 3
```

### 6.2 Environment Variables (Production)

```bash
# Database
export SPRING_DATASOURCE_URL=jdbc:mysql://easyticket-db.cluster-xxx.ap-southeast-1.rds.amazonaws.com:3306
export SPRING_DATASOURCE_USERNAME=easyticket
export SPRING_DATASOURCE_PASSWORD=xxx

# Redis
export SPRING_DATA_REDIS_HOST=easyticket-redis.xxx.cache.amazonaws.com
export SPRING_DATA_REDIS_PORT=6379
export SPRING_DATA_REDIS_PASSWORD=xxx

# Kafka
export SPRING_KAFKA_BOOTSTRAP_SERVERS=easyticket-kafka.xxx.kafka.ap-southeast-1.amazonaws.com:9092

# Keycloak
export KEYCLOAK_SERVER_URL=https://auth.easyticket.com
export KEYCLOAK_REALM=easyticket
export KEYCLOAK_CLIENT_SECRET=xxx

# AWS
export AWS_REGION=ap-southeast-1
export AWS_SES_FROM_EMAIL=noreply@easyticket.com
export TICKET_EMAIL_QUEUE_URL=https://sqs.ap-southeast-1.amazonaws.com/xxx/ticket-email

# Tracing
export OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4317
```

### 6.3 Production Checklist

- [ ] Database credentials rotated
- [ ] Redis password rotated
- [ ] Keycloak client secret rotated
- [ ] Payment gateway secrets configured
- [ ] AWS SES production access requested
- [ ] SSL certificates configured
- [ ] CloudWatch alarms set up
- [ ] Backup strategy configured
- [ ] Runbook documented
- [ ] Rollback procedure tested

### 6.4 Backup & Recovery

```bash
# RDS Automated Backup
aws rds modify-db-instance \
  --db-instance-identifier easyticket-user-db \
  --backup-retention-period 7 \
  --preferred-backup-window "03:00-04:00" \
  --preferred-maintenance-window "Mon:04:00-Mon:05:00"

# Point-in-time recovery
aws rds restore-db-instance-to-point-in-time \
  --source-db-instance-identifier easyticket-user-db \
  --target-db-instance-identifier easyticket-user-db-restored \
  --restore-time 2024-01-15T10:00:00Z
```

---

## 7. Monitoring & Troubleshooting

### 7.1 Kiểm tra Health

```bash
# Tất cả services
for port in 8081 8082 8083 8084 8085 8087; do
  echo "Service on port $port:"
  curl -s http://localhost:$port/actuator/health | jq
done
```

### 7.2 Xem Logs

```bash
# Docker Compose logs
docker compose logs -f mysql
docker compose logs -f redis
docker compose logs -f kafka
docker compose logs -f keycloak

# Application logs
docker compose logs -f event-service
docker compose logs -f ticket-service

# Kibana (local)
open http://localhost:5601/app/discover
```

### 7.3 Common Issues

#### Issue: MySQL Connection Failed

```bash
# Kiểm tra MySQL
docker compose ps mysql
docker compose logs mysql

# Restart MySQL
docker compose restart mysql

# Kiểm tra credentials
mysql -h localhost -u root -ppassword -e "SELECT 1"
```

#### Issue: Redis Connection Failed

```bash
# Kiểm tra Redis
docker compose ps redis
docker compose logs redis

# Restart Redis
docker compose restart redis

# Test connection
redis-cli -h localhost -a easyticket_redis ping
```

#### Issue: Kafka Not Starting

```bash
# Kiểm tra Zookeeper
docker compose ps zookeeper
docker compose logs zookeeper

# Restart Kafka
docker compose restart kafka

# Xóa data và restart (last resort)
docker compose down -v
docker compose up -d
```

#### Issue: Keycloak Realm Not Found

```bash
# Truy cập Keycloak Admin
# http://localhost:8080/admin

# Tạo Realm: SonNS_realm
# Tạo Client: quan_ly_ke_toan
# Cấu hình client:
#   - Access Type: confidential
#   - Valid Redirect URIs: *
#   - Web Origins: *
```

#### Issue: Ticket Service - Redis Lua Script Error

```bash
# Kiểm tra Lua script loaded
redis-cli -h localhost -a easyticket_redis --eval /path/to/script.lua,0

# Xem script source
redis-cli -h localhost -a easyticket_redis SCRIPT LIST
```

### 7.4 Performance Tuning

#### JVM Settings

```bash
# Recommended JVM settings for production
export JAVA_OPTS="-Xms512m -Xmx2g -XX:+UseG1GC -XX:MaxGCPauseMillis=200 -XX:+UseStringDeduplication"
```

#### Database Connection Pool

```yaml
# HikariCP settings
spring:
  datasource:
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      connection-timeout: 30000
      idle-timeout: 600000
      max-lifetime: 1800000
```

#### Redis Connection Pool

```yaml
spring:
  data:
    redis:
      lettuce:
        pool:
          max-active: 50
          max-idle: 20
          min-idle: 5
          max-wait: 2000ms
```

### 7.5 Security Checklist

- [ ] All default passwords changed
- [ ] SSL/TLS enabled for all endpoints
- [ ] Keycloak realm properly configured
- [ ] CORS settings configured for frontend domain
- [ ] Rate limiting enabled at API Gateway
- [ ] WAF rules configured
- [ ] Secrets stored in AWS Secrets Manager
- [ ] IAM roles follow least privilege principle
- [ ] Security groups restrict access appropriately

### 7.6 Useful Commands

```bash
# Rebuild infrastructure
docker compose down -v
docker compose up -d

# Reset Keycloak
docker compose exec keycloak /opt/keycloak/bin/kc.sh bootstrap admin

# Check Kafka consumer lag
docker compose exec kafka kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --group ticket-service-group \
  --describe

# Monitor Redis memory
redis-cli -h localhost -a easyticket_redis info memory

# Trace request through services
curl -H "X-B3-TraceId: test-trace" http://localhost:8082/api/v1/events
```

---

## Quick Reference

### Start Everything

```bash
# 1. Infrastructure
docker compose up -d

# 2. Run migrations (terminal 1-6)
cd EventService && ./mvnw spring-boot:run -pl EventService-migration

# 3. Run applications (terminal 1-6)
cd EventService && ./mvnw spring-boot:run -pl EventService-application
# ... các service khác tương tự
```

### Stop Everything

```bash
# Stop services
pkill -f "spring-boot:run"

# Stop infrastructure
docker compose down
```

### URLs

| Service | URL |
|---------|-----|
| API Gateway | http://localhost:8000 |
| Swagger UI | http://localhost:8000/ |
| Keycloak | http://localhost:8080 |
| Kibana | http://localhost:5601 |
| Kafka UI | http://localhost:8086 |
| MySQL | localhost:3306 |
| Redis | localhost:6379 |
