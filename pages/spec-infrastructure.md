# Yêu cầu triển khai (Infrastructure Specification)

## Mục lục

1. [Tổng quan](#1-tổng-quan)
2. [Kiến trúc hệ thống](#2-kiến-trúc-hệ-thống)
3. [Docker Infrastructure](#3-docker-infrastructure)
4. [Kubernetes Deployment](#4-kubernetes-deployment)
5. [Observability Stack](#5-observability-stack)
6. [Network & Security](#6-network--security)
7. [CI/CD Pipeline](#7-cicd-pipeline)

---

## 1. Tổng quan

### Môi trường triển khai

| Môi trường | Mục đích | Ghi chú |
|-------------|----------|---------|
| `local` | Dev máy cá nhân | Docker compose |
| `dev` | Staging/QA | Kubernetes |
| `prod` | Production | Kubernetes, high availability |

### Infrastructure Components

| Component | Công nghệ | Port | Mục đích |
|-----------|-----------|------|----------|
| API Gateway | AWS API Gateway (prod), NGINX (local) | 8000 | Rate limiting, routing |
| Database | MySQL 8.0 | 3306 | Database per service |
| Cache | Redis 7 | 6379 | Session cache, inventory |
| Message Broker | Apache Kafka 2.13 | 9092 | Async messaging |
| Identity | Keycloak 26.0.4 | 8080 | OAuth2/JWT |
| Email | AWS SES | - | Gửi email |
| Logs | Elasticsearch 8.10.2 | 9200 | Centralized logs |
| APM | Elastic APM Server | 8200 | Distributed tracing |
| Metrics | Prometheus + OTel | 8889 | Metrics collection |

---

## 2. Kiến trúc hệ thống

### Sơ đồ kiến trúc Production

```
                              ┌─────────────────────┐
                              │   AWS API Gateway    │
                              │  Rate Limiting       │
                              │  WAF Protection      │
                              └──────────┬──────────┘
                                         │
                    ┌────────────────────┼────────────────────┐
                    │                    │                    │
                    ▼                    ▼                    ▼
            ┌───────────────┐    ┌───────────────┐    ┌───────────────┐
            │Event Service  │    │Ticket Service │    │ Order Service │
            │  (EKS)        │    │  (EKS)        │    │  (EKS)        │
            │  2+ replicas   │    │  3+ replicas   │    │  2+ replicas   │
            └───────┬───────┘    └───────┬───────┘    └───────┬───────┘
                    │                    │                    │
                    ▼                    ▼                    ▼
            ┌───────────────┐    ┌───────────────┐    ┌───────────────┐
            │  MySQL        │    │   Redis       │    │  MySQL        │
            │  (RDS)        │    │   (ElastiCache)│   │  (RDS)        │
            └───────────────┘    └───────────────┘    └───────────────┘
                                        │
                                        ▼
                                ┌───────────────┐
                                │   Kafka       │
                                │   (MSK)       │
                                └───────┬───────┘
                                        │
                    ┌───────────────────┼───────────────────┐
                    │                   │                   │
                    ▼                   ▼                   ▼
            ┌───────────────┐    ┌───────────────┐    ┌───────────────┐
            │Payment Service│    │Notification   │    │  User Service │
            │  (EKS)        │    │  Service (EKS)│    │  (EKS)        │
            └───────┬───────┘    └───────┬───────┘    └───────────────┘
                    │                    │
                    ▼                    ▼
            ┌───────────────┐    ┌───────────────┐
            │  MySQL        │    │ AWS SQS       │
            │  (RDS)        │    │      ↓        │
            └───────────────┘    │   AWS SES     │
                                └───────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                         OTel Collector                               │
│    ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐          │
│    │ Traces   │→ │APM Server│→ │Elasticsearch│→ │ Kibana   │          │
│    │ Metrics  │→ │          │  │           │→ │          │          │
│    │ Logs     │→ │ Logstash │→ │           │→ │          │          │
│    └──────────┘  └──────────┘  └──────────┘  └──────────┘          │
└─────────────────────────────────────────────────────────────────────┘
```

### Local Development (Docker Compose)

```
                    ┌─────────────────┐
                    │  NGINX Gateway   │  :8000
                    │ (API Gateway)    │
                    └────────┬─────────┘
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
        ▼                    ▼                    ▼
┌───────────────┐    ┌───────────────┐    ┌───────────────┐
│Event Service  │    │Ticket Service │    │ Order Service │
│  (localhost)  │    │  (localhost) │    │  (localhost)  │
└───────┬───────┘    └───────┬───────┘    └───────┬───────┘
        │                    │                    │
        ▼                    ▼                    ▼
┌───────────────┐    ┌───────────────┐    ┌───────────────┐
│     MySQL     │    │     Redis     │    │     MySQL     │
│    :3306      │    │    :6379      │    │    :3306      │
└───────────────┘    └───────────────┘    └───────────────┘

┌─────────────────────────────────────────────────────────────┐
│                    Docker Network                           │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐         │
│  │  Kafka  │ │Keycloak │ │ Kibana  │ │  OTel   │         │
│  │  :9092   │ │  :8080   │ │ :5601   │ │Collector│         │
│  └─────────┘ └─────────┘ └─────────┘ └─────────┘         │
└─────────────────────────────────────────────────────────────┘
```

---

## 3. Docker Infrastructure

### Local Infrastructure Stack

Xem `docker-compose.yml` tại root project để biết chi tiết.

### Services trong Docker Compose

| Service | Image | Port | Health Check |
|---------|-------|------|--------------|
| mysql | mysql:8.0 | 3306 | `mysqladmin ping` |
| redis | redis:7-alpine | 6379 | `redis-cli ping` |
| zookeeper | bitnami/zookeeper:3.9.3 | 2181 | `zkServer.sh status` |
| kafka | wurstmeister/kafka:2.13-2.8.1 | 9092 | `kafka-topics.sh list` |
| kafka-ui | provectuslabs/kafka-ui | 8086 | - |
| keycloak | quay.io/keycloak/keycloak:26.0.4 | 8080 | TCP socket check |
| elasticsearch | elasticsearch:8.10.2 | 9200 | `curl /_cluster/health` |
| logstash | logstash:8.10.0 | 5000, 9600 | TCP socket check |
| kibana | kibana:8.10.2 | 5601 | `curl /api/status` |
| apm-server | apm-server:8.10.2 | 8200 | TCP socket check |
| otel-collector | otel/opentelemetry-collector-contrib:0.119.0 | 4317, 4318 | - |
| api-gateway | nginx:1.27-alpine | 8000 | `wget /healthz` |
| swagger-ui | swaggerapi/swagger-ui:v5.17.14 | - | - |

### Kafka Topics (Pre-created)

```yaml
KAFKA_CREATE_TOPICS: >-
  ticket-reserved:3:1,
  payment-success:3:1,
  payment-failed:3:1
```

### Environment Variables

| Variable | Default | Mô tả |
|----------|---------|-------|
| `MYSQL_ROOT_PASSWORD` | password | MySQL root password |
| `MYSQL_USER` | easyticket | MySQL username |
| `MYSQL_PASSWORD` | easyticket_password | MySQL password |
| `ENVIRONMENT` | local | Environment profile |
| `KC_BOOTSTRAP_ADMIN_USERNAME` | admin | Keycloak admin username |
| `KC_BOOTSTRAP_ADMIN_PASSWORD` | admin | Keycloak admin password |

---

## 4. Kubernetes Deployment

### Namespace Structure

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: easyticket
  labels:
    name: easyticket
---
apiVersion: v1
kind: Namespace
metadata:
  name: easyticket-infra
  labels:
    name: easyticket-infra
```

### Service Deployment Pattern

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: event-service
  namespace: easyticket
  labels:
    app: event-service
spec:
  replicas: 2
  selector:
    matchLabels:
      app: event-service
  template:
    metadata:
      labels:
        app: event-service
    spec:
      containers:
        - name: event-service
          image: easyticket/event-service:latest
          ports:
            - containerPort: 8080
          env:
            - name: SPRING_PROFILES_ACTIVE
              value: "prod"
            - name: SPRING_DATASOURCE_URL
              valueFrom:
                secretKeyRef:
                  name: database-credentials
                  key: event-db-url
          resources:
            requests:
              memory: "512Mi"
              cpu: "250m"
            limits:
              memory: "1Gi"
              cpu: "500m"
          readinessProbe:
            httpGet:
              path: /actuator/health
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /actuator/health
              port: 8080
            initialDelaySeconds: 60
            periodSeconds: 15
```

### Service Type

```yaml
apiVersion: v1
kind: Service
metadata:
  name: event-service
  namespace: easyticket
spec:
  type: ClusterIP
  selector:
    app: event-service
  ports:
    - port: 80
      targetPort: 8080
```

### Horizontal Pod Autoscaler

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: ticket-service-hpa
  namespace: easyticket
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: ticket-service
  minReplicas: 3
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

### Resource Requirements

| Service | CPU Request | CPU Limit | Memory Request | Memory Limit | Replicas |
|---------|-------------|-----------|----------------|--------------|----------|
| Event Service | 250m | 500m | 512Mi | 1Gi | 2+ |
| Ticket Service | 500m | 1 core | 1Gi | 2Gi | 3+ |
| Order Service | 250m | 500m | 512Mi | 1Gi | 2+ |
| Payment Service | 250m | 500m | 512Mi | 1Gi | 2+ |
| Notification Service | 100m | 200m | 256Mi | 512Mi | 1+ |
| User Service | 250m | 500m | 512Mi | 1Gi | 2+ |

### Ingress Configuration (AWS ALB)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: easyticket-ingress
  namespace: easyticket
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:...
    alb.ingress.kubernetes.io/rules-id: easyticket-rules
spec:
  rules:
    - http:
        paths:
          - path: /api/v1/events
            backend:
              service:
                name: event-service
                port:
                  number: 80
          - path: /api/v1/tickets
            backend:
              service:
                name: ticket-service
                port:
                  number: 80
          - path: /api/v1/orders
            backend:
              service:
                name: order-service
                port:
                  number: 80
          - path: /api/v1/payments
            backend:
              service:
                name: payment-service
                port:
                  number: 80
          - path: /api/v1/users
            backend:
              service:
                name: user-service
                port:
                  number: 80
```

---

## 5. Observability Stack

### OTel Collector Configuration

```yaml
# infra/otel-collector/otel-collector.yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch:
    timeout: 5s
    send_batch_size: 1024
  memory_limiter:
    check_interval: 1s
    limit_mib: 512

exporters:
  otlp/traces:
    endpoint: http://apm-server:8200
    tls:
      insecure: true
  logstash:
    endpoint: logstash:5000
  prometheus:
    endpoint: 0.0.0.0:8889

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [otlp/traces]
    logs:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [logstash]
    metrics:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [prometheus]
```

### Logback Configuration

```xml
<!-- logback-spring.xml -->
<appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
    <encoder class="net.logstash.logback.encoder.LogstashEncoder">
        <includeMdcKeyName>traceId</includeMdcKeyName>
        <includeMdcKeyName>spanId</includeMdcKeyName>
    </encoder>
</appender>

<appender name="LOGSTASH" class="net.logstash.logback.appender.LogstashTcpSocketAppender">
    <destination>otel-collector:5000</destination>
    <encoder class="net.logstash.logback.encoder.LogstashEncoder">
        <includeMdcKeyName>traceId</includeMdcKeyName>
        <includeMdcKeyName>spanId</includeMdcKeyName>
    </encoder>
</appender>

<springProfile name="local,dev">
    <root level="DEBUG">
        <appender-ref ref="CONSOLE" />
    </root>
</springProfile>

<springProfile name="prod">
    <root level="INFO">
        <appender-ref ref="LOGSTASH" />
    </root>
</springProfile>
```

### Tracing Sampling

```yaml
# application-prod.yaml
management:
  tracing:
    sampling:
      probability: 0.1  # 10% in prod
```

### Kibana Dashboards

| Dashboard | Data Source | Metrics |
|-----------|-------------|---------|
| Service Map | APM | Request flow between services |
| Transaction | APM | Latency breakdown |
| Logs Explorer | Elasticsearch | easyticket-logs-* |
| Metrics | Prometheus | CPU, Memory, Request rate |

---

## 6. Network & Security

### Network Policies (Production)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: ticket-service-network-policy
  namespace: easyticket
spec:
  podSelector:
    matchLabels:
      app: ticket-service
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              name: ingress-nginx
      ports:
        - protocol: TCP
          port: 8080
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              name: easyticket-infra
      ports:
        - protocol: TCP
          port: 6379  # Redis
        - protocol: TCP
          port: 9092  # Kafka
```

### Secrets Management

```yaml
# Kubernetes Secret
apiVersion: v1
kind: Secret
metadata:
  name: database-credentials
  namespace: easyticket
type: Opaque
stringData:
  event-db-url: jdbc:mysql://event-db.internal:3306/event_db
  event-db-username: easyticket
  event-db-password: <encrypted>
  redis-password: <encrypted>
```

### AWS Security Groups

| Security Group | Inbound | Outbound |
|----------------|---------|----------|
| EKS Node Group | 443 (API Server), 10250 (Kubelet) | 0.0.0.0/0 |
| RDS | 3306 (from EKS) | 0.0.0.0/0 |
| ElastiCache | 6379 (from EKS) | 0.0.0.0/0 |
| MSK (Kafka) | 9092 (from EKS) | 0.0.0.0/0 |

---

## 7. CI/CD Pipeline

### GitHub Actions Workflow

```yaml
# .github/workflows/deploy.yml
name: Deploy to EKS

on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up JDK 21
        uses: actions/setup-java@v4
        with:
          distribution: 'temurin'
          java-version: '21'
          
      - name: Build with Maven
        run: ./mvnw clean package -DskipTests
        
      - name: Build Docker image
        run: |
          docker build -t ${{ env.REGISTRY }}/event-service:${{ github.sha }} .
          
      - name: Push to ECR
        run: |
          aws ecr get-login-password | docker login --username AWS --password-stdin ${{ env.REGISTRY }}
          docker push ${{ env.REGISTRY }}/event-service:${{ github.sha }}
          
      - name: Update deployment
        run: |
          kubectl set image deployment/event-service event-service=${{ env.REGISTRY }}/event-service:${{ github.sha }}
```

### Helm Charts Structure

```
helm/
├── Chart.yaml
├── values.yaml
├── values-local.yaml
├── values-dev.yaml
├── values-prod.yaml
└── templates/
    ├── deployment.yaml
    ├── service.yaml
    ├── hpa.yaml
    ├── ingress.yaml
    └── configmap.yaml
```

### Deployment Strategy

| Strategy | Description | Use Case |
|----------|-------------|----------|
| Rolling Update | Replace pods gradually | Default for all services |
| Blue/Green | Deploy new version alongside | Major releases |
| Canary | Route small % to new version | Risky changes |

### Health Checks

```yaml
livenessProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8080
  initialDelaySeconds: 60
  periodSeconds: 15
  failureThreshold: 3

readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
  failureThreshold: 1
```

---

## Database Connection Pool

### Recommended Configuration

| Service | HikariCP Pool Size | Connection Timeout |
|---------|-------------------|-------------------|
| Event Service | 10 | 30s |
| Ticket Service | 20 | 30s |
| Order Service | 10 | 30s |
| Payment Service | 10 | 30s |
| User Service | 10 | 30s |

```yaml
# application.yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 10
      minimum-idle: 5
      connection-timeout: 30000
      idle-timeout: 600000
      max-lifetime: 1800000
```

---

## Redis Configuration

### Cluster Mode (Production)

```yaml
# ElastiCache Redis Cluster
cluster-mode: enabled
num-node-groups: 3
replicas-per-node-group: 2

# Connection
spring:
  data:
    redis:
      cluster:
        nodes: redis-001:6379,redis-002:6379,redis-003:6379
      password: ${REDIS_PASSWORD}
      timeout: 2000ms
      lettuce:
        pool:
          max-active: 50
          max-idle: 20
          min-idle: 5
```

### Lua Script Management

```yaml
# application.yaml
spring:
  data:
    redis:
      scripts:
        ticket-inventory: classpath:scripts/ticket-inventory.lua
        ticket-release: classpath:scripts/ticket-release.lua
```
