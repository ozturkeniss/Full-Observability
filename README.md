# Full Observability Microservices

A production-ready microservices architecture built with Go, featuring comprehensive observability, service mesh, and event-driven patterns.

## Architecture Overview

```mermaid
graph TB
    subgraph "External Access"
        Client([Client/Browser])
        Admin([Administrator])
    end
    
    subgraph "Istio Service Mesh"
        IG[Istio Ingress Gateway<br/>mTLS + Load Balancing]
        
        subgraph "API Layer"
            GW[API Gateway<br/>:8000<br/>Rate Limit + Circuit Breaker]
        end
        
        subgraph "Microservices"
            US[User Service<br/>:8080 HTTP<br/>:9090 gRPC]
            PS[Product Service<br/>:8081 HTTP<br/>:9091 gRPC]
            IS[Inventory Service<br/>:8082 HTTP<br/>:9092 gRPC]
            PYS[Payment Service<br/>:8083 HTTP]
        end
        
        subgraph "Infrastructure"
            PG[(PostgreSQL<br/>userdb, productdb<br/>inventorydb, paymentdb)]
            RD[(Redis<br/>Cache + Rate Limit)]
            KF[Kafka<br/>Event Bus]
        end
        
        subgraph "Observability Stack"
            PR[Prometheus<br/>:9090<br/>Metrics]
            GR[Grafana<br/>:3000<br/>Dashboards]
            JG[Jaeger<br/>:16686<br/>Tracing]
        end
    end
    
    Client -->|HTTPS| IG
    IG -->|mTLS| GW
    
    GW -->|mTLS| US
    GW -->|mTLS| PS
    GW -->|mTLS| IS
    GW -->|mTLS| PYS
    
    US --> PG
    PS --> PG
    IS --> PG
    PYS --> PG
    
    GW --> RD
    
    PYS -->|Publish Events| KF
    IS -->|Consume Events| KF
    
    PYS -.->|gRPC Call| US
    PYS -.->|gRPC Call| PS
    PYS -.->|gRPC Call| IS
    PS -.->|gRPC Call| US
    IS -.->|gRPC Call| US
    
    US -.->|Metrics| PR
    PS -.->|Metrics| PR
    IS -.->|Metrics| PR
    PYS -.->|Metrics| PR
    GW -.->|Metrics| PR
    
    US -.->|Traces| JG
    PS -.->|Traces| JG
    IS -.->|Traces| JG
    PYS -.->|Traces| JG
    GW -.->|Traces| JG
    
    PR --> GR
    JG --> GR
    
    Admin -->|View| GR
    Admin -->|View| JG
```

## Microservices Architecture

### Service Responsibilities

```mermaid
graph LR
    subgraph "User Service"
        U1[Authentication<br/>JWT Generation]
        U2[User Management<br/>CRUD Operations]
        U3[Role Management<br/>Admin/User]
        U4[gRPC API<br/>User Validation]
    end
    
    subgraph "Product Service"
        P1[Product Catalog<br/>CRUD Operations]
        P2[Category Management<br/>Filtering]
        P3[Stock Tracking<br/>Inventory Sync]
        P4[gRPC API<br/>Product Info]
    end
    
    subgraph "Inventory Service"
        I1[Stock Management<br/>Quantity Tracking]
        I2[Availability Check<br/>gRPC Service]
        I3[Kafka Consumer<br/>Purchase Events]
        I4[Stock Updates<br/>Real-time Sync]
    end
    
    subgraph "Payment Service"
        PY1[Payment Processing<br/>Transaction Management]
        PY2[Multi-Service Orchestration<br/>User+Product+Inventory]
        PY3[Kafka Producer<br/>Payment Events]
        PY4[Idempotency<br/>Duplicate Prevention]
    end
```

### CQRS Pattern Implementation

```mermaid
graph TB
    subgraph "CQRS - Command Query Responsibility Segregation"
        direction LR
        
        subgraph "Write Path (Commands)"
            C1[HTTP POST/PUT/DELETE] --> CH[Command Handler]
            CH --> CV[Validation]
            CV --> CR[Repository Write]
            CR --> DB1[(Database<br/>Write Operations)]
            CR --> EV[Event Publisher]
            EV --> KF[Kafka]
        end
        
        subgraph "Read Path (Queries)"
            Q1[HTTP GET] --> QH[Query Handler]
            QH --> QR[Repository Read]
            QR --> DB2[(Database<br/>Read Operations)]
        end
    end
    
    style C1 fill:#ff9999
    style Q1 fill:#99ff99
    style DB1 fill:#ffcc99
    style DB2 fill:#99ccff
```

### Event-Driven Architecture

```mermaid
sequenceDiagram
    participant Client
    participant Payment as Payment Service
    participant User as User Service
    participant Product as Product Service
    participant Inventory as Inventory Service
    participant Kafka
    participant DB as PostgreSQL
    
    Client->>Payment: POST /api/payments
    
    Note over Payment: 1. Validate Request
    Payment->>User: gRPC: ValidateUser(userID)
    User-->>Payment: User Valid ✓
    
    Payment->>Product: gRPC: GetProduct(productID)
    Product-->>Payment: Product Details
    
    Payment->>Inventory: gRPC: CheckAvailability(productID, qty)
    Inventory-->>Payment: Available ✓
    
    Note over Payment: 2. Create Payment
    Payment->>DB: INSERT payment
    DB-->>Payment: Payment ID
    
    Note over Payment: 3. Publish Event
    Payment->>Kafka: ProductPurchasedEvent
    Kafka-->>Payment: ACK
    
    Payment-->>Client: 201 Created
    
    Note over Inventory: 4. Consume Event (Async)
    Kafka->>Inventory: ProductPurchasedEvent
    Inventory->>DB: UPDATE inventory SET quantity = quantity - 1
    Inventory->>Kafka: InventoryUpdatedEvent
```

## Observability Architecture

### Metrics Flow

```mermaid
graph LR
    subgraph "Application Layer"
        US[User Service<br/>Metrics Middleware]
        PS[Product Service<br/>Metrics Middleware]
        IS[Inventory Service<br/>Metrics Middleware]
        PYS[Payment Service<br/>Metrics Middleware]
    end
    
    subgraph "Service Mesh Layer"
        E1[Envoy Proxy<br/>User Service]
        E2[Envoy Proxy<br/>Product Service]
        E3[Envoy Proxy<br/>Inventory Service]
        E4[Envoy Proxy<br/>Payment Service]
    end
    
    subgraph "Monitoring"
        PR[Prometheus<br/>Time Series DB]
        GR[Grafana<br/>Visualization]
    end
    
    US --> E1
    PS --> E2
    IS --> E3
    PYS --> E4
    
    E1 -.->|/metrics<br/>every 15s| PR
    E2 -.->|/metrics| PR
    E3 -.->|/metrics| PR
    E4 -.->|/metrics| PR
    
    US -.->|/metrics| PR
    PS -.->|/metrics| PR
    IS -.->|/metrics| PR
    PYS -.->|/metrics| PR
    
    PR --> GR
    
    style E1 fill:#e1f5ff
    style E2 fill:#e1f5ff
    style E3 fill:#e1f5ff
    style E4 fill:#e1f5ff
```

### Distributed Tracing

```mermaid
graph TD
    subgraph "Trace Context Propagation"
        C[Client Request<br/>trace-id: abc123] --> GW[API Gateway]
        GW -->|trace-id: abc123<br/>span-id: 001| PYS[Payment Service]
        PYS -->|trace-id: abc123<br/>span-id: 002| US[User Service]
        PYS -->|trace-id: abc123<br/>span-id: 003| PS[Product Service]
        PYS -->|trace-id: abc123<br/>span-id: 004| IS[Inventory Service]
    end
    
    subgraph "Trace Collection"
        GW -.->|Span Data| JC[Jaeger Collector]
        PYS -.->|Span Data| JC
        US -.->|Span Data| JC
        PS -.->|Span Data| JC
        IS -.->|Span Data| JC
        
        JC --> JDB[(Jaeger Storage)]
        JDB --> JUI[Jaeger UI<br/>Trace Visualization]
    end
    
    style JC fill:#ffe1e1
    style JUI fill:#e1ffe1
```

## Istio Service Mesh

### mTLS Communication

```mermaid
graph LR
    subgraph "Service A Pod"
        SA[Service A<br/>Container]
        EA[Envoy Proxy A<br/>Sidecar]
        SA --> EA
    end
    
    subgraph "Service B Pod"
        EB[Envoy Proxy B<br/>Sidecar]
        SB[Service B<br/>Container]
        EB --> SB
    end
    
    EA -->|1. mTLS Handshake<br/>Certificate Exchange| EB
    EA ==>|2. Encrypted Traffic<br/>TLS 1.3| EB
    
    subgraph "Istio Control Plane"
        Istiod[Istiod<br/>Certificate Authority]
    end
    
    Istiod -.->|Auto-rotate Certs<br/>Every 24h| EA
    Istiod -.->|Auto-rotate Certs| EB
    
    style EA fill:#fff4e1
    style EB fill:#fff4e1
    style Istiod fill:#e1f0ff
```

### Traffic Management

```mermaid
graph TD
    subgraph "Istio Gateway"
        GW[Istio Ingress<br/>Port 80/443]
    end
    
    subgraph "VirtualService - Routing Rules"
        VS[VirtualService<br/>URL Matching<br/>Header Matching<br/>Weight Distribution]
    end
    
    subgraph "DestinationRule - Traffic Policy"
        DR[Load Balancing<br/>Circuit Breaker<br/>Connection Pool<br/>mTLS]
    end
    
    subgraph "Service Versions"
        V1[Version v1<br/>90% Traffic<br/>Stable]
        V2[Version v2<br/>10% Traffic<br/>Canary]
    end
    
    GW --> VS
    VS -->|Route Decision| DR
    DR -->|90% Weight| V1
    DR -->|10% Weight| V2
    
    style VS fill:#ffe1e1
    style DR fill:#e1ffe1
    style V1 fill:#e1e1ff
    style V2 fill:#ffe1ff
```

### Circuit Breaker Pattern

```mermaid
stateDiagram-v2
    [*] --> Closed: Healthy State
    Closed --> Open: 5 Consecutive Errors
    Open --> HalfOpen: Wait 30s
    HalfOpen --> Closed: Request Success
    HalfOpen --> Open: Request Failed
    
    note right of Closed
        All requests allowed
        Monitor error rate
    end note
    
    note right of Open
        All requests rejected
        Fail fast (no wait)
        Protect downstream
    end note
    
    note right of HalfOpen
        Allow 1 test request
        Check if recovered
    end note
```

## Kubernetes Deployment

### Pod Architecture

```mermaid
graph TB
    subgraph "Kubernetes Pod - User Service"
        subgraph "Init Containers"
            I1[wait-for-postgres]
            I2[wait-for-redis]
        end
        
        subgraph "Application Containers"
            direction LR
            AC[User Service<br/>Go Application<br/>Port 8080/9090]
            EP[Istio Envoy Proxy<br/>Sidecar<br/>Port 15001/15006]
        end
        
        subgraph "Volumes"
            CM[ConfigMap<br/>app-config]
            SEC[Secret<br/>db-credentials]
        end
        
        I1 --> AC
        I2 --> AC
        AC <--> EP
        CM --> AC
        SEC --> AC
    end
    
    subgraph "Probes"
        LP[Liveness Probe<br/>/health]
        RP[Readiness Probe<br/>/health]
        SP[Startup Probe<br/>/health]
    end
    
    AC --> LP
    AC --> RP
    AC --> SP
    
    style EP fill:#fff4e1
    style AC fill:#e1f0ff
```

### Horizontal Pod Autoscaling

```mermaid
graph LR
    subgraph "HPA Controller"
        HPA[HorizontalPodAutoscaler<br/>Target: CPU 70%<br/>Memory 80%]
    end
    
    subgraph "Deployment"
        D[Deployment<br/>user-service]
    end
    
    subgraph "Pods"
        P1[Pod 1<br/>CPU: 30%]
        P2[Pod 2<br/>CPU: 85%]
        P3[Pod 3<br/>CPU: 75%]
    end
    
    subgraph "Metrics Server"
        MS[Metrics Server<br/>Collects Resource Usage]
    end
    
    D --> P1
    D --> P2
    D --> P3
    
    P1 -.->|CPU/Memory| MS
    P2 -.->|CPU/Memory| MS
    P3 -.->|CPU/Memory| MS
    
    MS -->|Average: 63%| HPA
    HPA -->|Scale Decision| D
    
    HPA -.->|If > 70%: Scale UP| D
    HPA -.->|If < 30%: Scale DOWN| D
```

### Deployment Strategies

```mermaid
graph TB
    subgraph "Canary Deployment"
        direction TB
        S1[Stage 1: v1 100%] --> S2[Stage 2: v1 90% + v2 10%]
        S2 --> S3[Stage 3: v1 70% + v2 30%]
        S3 --> S4[Stage 4: v1 50% + v2 50%]
        S4 --> S5[Stage 5: v2 100%]
        
        S2 -.->|Errors > 5%| RB1[Rollback to v1]
        S3 -.->|High Latency| RB1
        S4 -.->|Resource Issues| RB1
    end
    
    subgraph "Blue-Green Deployment"
        direction LR
        B1[Blue v1<br/>100% Active] --> SW{Switch}
        G1[Green v2<br/>0% Standby] --> SW
        SW -->|Instant Switch| B2[Blue v1<br/>0% Standby]
        SW --> G2[Green v2<br/>100% Active]
        B2 -.->|Instant Rollback| SW
    end
    
    style S5 fill:#99ff99
    style RB1 fill:#ff9999
    style G2 fill:#99ff99
```

## Security Architecture

### Authentication & Authorization Flow

```mermaid
sequenceDiagram
    participant C as Client
    participant IG as Istio Gateway
    participant GW as API Gateway
    participant US as User Service
    participant PS as Payment Service
    
    Note over C,PS: 1. User Authentication
    C->>IG: POST /auth/login<br/>{username, password}
    IG->>GW: mTLS Encrypted
    GW->>US: Forward Request
    US->>US: Validate Credentials
    US->>US: Generate JWT Token
    US-->>GW: {token: "eyJhbGc..."}
    GW-->>IG: Response
    IG-->>C: JWT Token
    
    Note over C,PS: 2. Authenticated Request
    C->>IG: POST /api/payments<br/>Header: Authorization: Bearer eyJhbGc...
    IG->>GW: mTLS + JWT
    GW->>GW: Validate JWT Signature
    GW->>GW: Check Authorization
    GW->>PS: Forward with User Context
    
    Note over PS: 3. Service-to-Service (mTLS)
    PS->>PS: Extract User ID from Context
    PS->>US: gRPC: ValidateUser()<br/>mTLS Certificate Exchange
    US->>US: Verify mTLS Certificate<br/>Principal: payment-service
    US->>US: Check Authorization Policy
    US-->>PS: User Valid
    
    PS-->>GW: Payment Created
    GW-->>IG: Response
    IG-->>C: 201 Created
```

### mTLS Certificate Lifecycle

```mermaid
sequenceDiagram
    participant Istiod as Istio CA
    participant Pod as Service Pod
    participant Envoy as Envoy Proxy
    participant Remote as Remote Service
    
    Note over Istiod,Envoy: Certificate Issuance
    Pod->>Istiod: Request Certificate<br/>Identity: user-service
    Istiod->>Istiod: Generate Certificate<br/>Valid: 24 hours<br/>SPIFFE ID
    Istiod-->>Pod: Certificate + Private Key
    Pod->>Envoy: Mount Certificate
    
    Note over Envoy,Remote: mTLS Connection
    Envoy->>Remote: TLS Handshake<br/>Present Certificate
    Remote->>Remote: Verify Certificate<br/>Check CA Signature
    Remote-->>Envoy: Certificate OK
    Envoy->>Remote: Encrypted Request
    Remote-->>Envoy: Encrypted Response
    
    Note over Istiod,Envoy: Auto-Rotation (12 hours later)
    Istiod->>Istiod: Check Expiry<br/>< 12h remaining
    Istiod->>Pod: New Certificate
    Pod->>Envoy: Hot Reload<br/>Zero Downtime
```

## Layer Architecture

### Clean Architecture Layers

```mermaid
graph TB
    subgraph "Delivery Layer"
        HTTP[HTTP Handler<br/>REST API + Swagger]
        GRPC[gRPC Server<br/>Service Implementation]
        MW[Middleware<br/>Auth, Logging, Metrics, Tracing]
    end
    
    subgraph "UseCase Layer - CQRS"
        direction LR
        
        subgraph "Commands"
            CMD1[CreateCommand]
            CMD2[UpdateCommand]
            CMD3[DeleteCommand]
        end
        
        subgraph "Queries"
            QRY1[GetQuery]
            QRY2[ListQuery]
            QRY3[StatsQuery]
        end
    end
    
    subgraph "Domain Layer"
        ENT[Entities<br/>Business Logic]
        REPO[Repository Interface<br/>Contracts]
    end
    
    subgraph "Infrastructure Layer"
        GORM[GORM Repository<br/>PostgreSQL]
        KAFKA[Kafka Publisher/Consumer]
        REDIS[Redis Cache]
    end
    
    HTTP --> MW
    GRPC --> MW
    MW --> CMD1
    MW --> CMD2
    MW --> CMD3
    MW --> QRY1
    MW --> QRY2
    MW --> QRY3
    
    CMD1 --> ENT
    CMD2 --> ENT
    CMD3 --> ENT
    QRY1 --> ENT
    QRY2 --> ENT
    QRY3 --> ENT
    
    ENT --> REPO
    REPO --> GORM
    REPO --> KAFKA
    REPO --> REDIS
    
    style MW fill:#ffe1e1
    style ENT fill:#e1ffe1
    style REPO fill:#e1e1ff
```

### Request Lifecycle

```mermaid
sequenceDiagram
    participant C as Client
    participant IG as Istio Gateway
    participant E as Envoy Proxy
    participant MW as Middleware Stack
    participant H as Handler
    participant UC as UseCase (CQRS)
    participant R as Repository
    participant DB as Database
    participant P as Prometheus
    participant J as Jaeger
    
    C->>IG: HTTPS Request
    IG->>E: mTLS (Certificate Validated)
    
    Note over E: Envoy Intercepts
    E->>E: 1. Create Trace Span
    E->>E: 2. Record Request Start
    
    E->>MW: Forward to App
    
    Note over MW: Middleware Chain
    MW->>MW: 1. Logging Middleware
    MW->>MW: 2. Tracing Middleware<br/>Extract/Create Span
    MW->>MW: 3. Metrics Middleware<br/>Start Timer
    MW->>MW: 4. Auth Middleware<br/>Validate JWT
    
    MW->>H: Authenticated Request
    H->>UC: Execute Command/Query
    UC->>R: Repository Call
    R->>DB: SQL Query
    DB-->>R: Result Set
    R-->>UC: Domain Entity
    UC-->>H: Response DTO
    H-->>MW: HTTP Response
    
    Note over MW: Record Metrics
    MW->>MW: duration = time.Since(start)
    MW->>MW: counter.Inc()
    MW->>MW: histogram.Observe(duration)
    
    MW-->>E: Response
    
    Note over E: Envoy Records
    E->>E: 1. Complete Trace Span
    E->>E: 2. Record Response Time
    E->>E: 3. Update Circuit Breaker
    
    E-->>IG: mTLS Response
    IG-->>C: HTTPS Response
    
    E-.->|Trace Data| J
    E-.->|Metrics| P
    MW-.->|App Metrics| P
```

## Resilience Patterns

### Circuit Breaker in Action

```mermaid
graph TB
    subgraph "Normal Operation - Closed State"
        R1[Request 1] --> S1[Success ✓]
        R2[Request 2] --> S2[Success ✓]
        R3[Request 3] --> S3[Success ✓]
    end
    
    subgraph "Failures Detected"
        R4[Request 4] --> F1[Error ❌]
        R5[Request 5] --> F2[Error ❌]
        R6[Request 6] --> F3[Error ❌]
        R7[Request 7] --> F4[Error ❌]
        R8[Request 8] --> F5[Error ❌]
    end
    
    subgraph "Circuit Open - Fast Fail"
        R9[Request 9] --> FF1[Immediate Fail<br/>No Backend Call]
        R10[Request 10] --> FF2[Immediate Fail]
        R11[Request 11] --> FF3[Immediate Fail]
    end
    
    subgraph "Half-Open - Testing"
        R12[Request 12<br/>After 30s] --> T1{Test Request}
        T1 -->|Success| CLOSE[Circuit Closed<br/>Resume Normal]
        T1 -->|Failure| OPEN[Stay Open<br/>Wait 60s]
    end
    
    S3 --> F1
    F5 --> FF1
    FF3 --> R12
    
    style F1 fill:#ffcccc
    style F5 fill:#ffcccc
    style FF1 fill:#ff9999
    style CLOSE fill:#99ff99
    style OPEN fill:#ff9999
```

### Retry Strategy

```mermaid
flowchart TD
    Start[Request Starts] --> Attempt1[Attempt 1<br/>Timeout: 10s]
    
    Attempt1 -->|Success| Success[Return 200 OK]
    Attempt1 -->|5xx Error| Wait1[Wait 100ms]
    Attempt1 -->|Timeout| Wait1
    
    Wait1 --> Attempt2[Attempt 2<br/>Timeout: 10s]
    Attempt2 -->|Success| Success
    Attempt2 -->|5xx Error| Wait2[Wait 200ms<br/>Exponential Backoff]
    
    Wait2 --> Attempt3[Attempt 3<br/>Timeout: 10s]
    Attempt3 -->|Success| Success
    Attempt3 -->|5xx Error| Failed[Return Error<br/>All Attempts Failed]
    
    Attempt1 -->|4xx Error| Failed
    Attempt2 -->|4xx Error| Failed
    Attempt3 -->|Timeout| Failed
    
    style Success fill:#99ff99
    style Failed fill:#ff9999
```

## Helm Deployment

### Helm Chart Structure

```mermaid
graph TB
    subgraph "Helm Chart"
        Chart[Chart.yaml<br/>Metadata]
        Values[values.yaml<br/>Configuration]
        
        subgraph "Templates"
            NS[namespace.yaml]
            SEC[secret.yaml]
            CM[configmap.yaml]
            
            subgraph "Infrastructure"
                PG[postgresql.yaml<br/>StatefulSet + PVC]
                RD[redis.yaml]
                KF[kafka.yaml]
            end
            
            subgraph "Observability"
                PROM[prometheus.yaml]
                GRAF[grafana.yaml]
                JAEG[jaeger.yaml]
            end
            
            subgraph "Microservices"
                USR[user-service.yaml<br/>Deployment + Service + HPA]
                PRD[product-service.yaml]
                INV[inventory-service.yaml]
                PAY[payment-service.yaml]
            end
            
            subgraph "Istio"
                GTW[gateway.yaml]
                VS[virtualservice.yaml]
                DR[destinationrule.yaml]
                PA[peer-authentication.yaml]
                AZ[authorization-policy.yaml]
            end
            
            ING[ingress.yaml]
            NP[network-policy.yaml]
        end
    end
    
    Chart --> Values
    Values --> Templates
    
    style Chart fill:#e1f0ff
    style Values fill:#ffe1e1
```

### Resource Dependencies

```mermaid
graph TD
    subgraph "Phase 1: Foundation"
        NS[Namespace] --> SEC[Secrets]
        SEC --> CM[ConfigMaps]
    end
    
    subgraph "Phase 2: Storage"
        CM --> PVC[PersistentVolumeClaims]
    end
    
    subgraph "Phase 3: Infrastructure"
        PVC --> PG[PostgreSQL<br/>StatefulSet]
        PVC --> RD[Redis<br/>Deployment]
        PVC --> ZK[Zookeeper<br/>StatefulSet]
        ZK --> KF[Kafka<br/>StatefulSet]
    end
    
    subgraph "Phase 4: Observability"
        PVC --> PR[Prometheus]
        PVC --> GR[Grafana]
        PG --> JG[Jaeger]
    end
    
    subgraph "Phase 5: Microservices"
        PG --> US[User Service]
        US --> PS[Product Service]
        PS --> IS[Inventory Service]
        KF --> IS
        IS --> PYS[Payment Service]
        RD --> PYS
    end
    
    subgraph "Phase 6: Gateway"
        US --> GW[API Gateway]
        PS --> GW
        IS --> GW
        PYS --> GW
        RD --> GW
    end
    
    subgraph "Phase 7: Ingress"
        GW --> ING[Ingress<br/>TLS + Rules]
    end
    
    style NS fill:#e1f0ff
    style PG fill:#ffe1e1
    style GW fill:#e1ffe1
    style ING fill:#ffe1ff
```

## Monitoring Metrics

### Application Metrics

```mermaid
graph LR
    subgraph "Counter Metrics"
        C1[http_requests_total<br/>Labels: method, endpoint, status]
        C2[grpc_requests_total<br/>Labels: service, method, code]
        C3[db_queries_total<br/>Labels: operation, table]
    end
    
    subgraph "Histogram Metrics"
        H1[http_request_duration_seconds<br/>Buckets: 0.1, 0.5, 1, 5, 10]
        H2[grpc_request_duration_seconds]
        H3[db_query_duration_seconds]
    end
    
    subgraph "Gauge Metrics"
        G1[active_connections<br/>Current Value]
        G2[goroutines_count]
        G3[memory_usage_bytes]
    end
    
    subgraph "Summary Metrics"
        S1[request_size_bytes<br/>Quantiles: 0.5, 0.9, 0.99]
        S2[response_size_bytes]
    end
```

### Istio Service Mesh Metrics

```mermaid
graph TB
    subgraph "Request Metrics"
        R1[istio_requests_total<br/>source_app, destination_app<br/>response_code, response_flags]
        R2[istio_request_duration_milliseconds<br/>source_app, destination_app<br/>response_code]
        R3[istio_request_bytes<br/>Request Size]
        R4[istio_response_bytes<br/>Response Size]
    end
    
    subgraph "TCP Metrics"
        T1[istio_tcp_sent_bytes_total]
        T2[istio_tcp_received_bytes_total]
        T3[istio_tcp_connections_opened_total]
        T4[istio_tcp_connections_closed_total]
    end
    
    subgraph "Circuit Breaker Metrics"
        CB1[envoy_cluster_upstream_cx_overflow<br/>Connection Pool Overflow]
        CB2[envoy_cluster_outlier_detection_ejections_active<br/>Ejected Hosts]
        CB3[envoy_cluster_upstream_rq_pending_overflow<br/>Request Queue Overflow]
    end
    
    style R1 fill:#e1f0ff
    style CB1 fill:#ffe1e1
```

## Data Flow Patterns

### Write Operation (Command)

```mermaid
flowchart TD
    Start[Client: POST /api/products] --> Gateway{API Gateway<br/>Rate Limit Check}
    
    Gateway -->|Allowed| Auth[JWT Validation]
    Gateway -->|Exceeded| Reject1[429 Too Many Requests]
    
    Auth -->|Valid| PS[Product Service]
    Auth -->|Invalid| Reject2[401 Unauthorized]
    
    PS --> Validate[Input Validation]
    Validate -->|Invalid| Reject3[400 Bad Request]
    Validate -->|Valid| CMD[CreateProductCommand]
    
    CMD --> CheckPerm[Check Permissions<br/>Admin Only]
    CheckPerm -->|Not Admin| Reject4[403 Forbidden]
    CheckPerm -->|Admin| Repo[Repository.Create]
    
    Repo --> TX[Begin Transaction]
    TX --> Insert[INSERT INTO products]
    Insert -->|Error| Rollback[Rollback Transaction]
    Insert -->|Success| Commit[Commit Transaction]
    
    Rollback --> Error[500 Internal Error]
    Commit --> Event[Publish Event<br/>ProductCreated]
    Event --> Cache[Invalidate Cache]
    Cache --> Success[201 Created]
    
    style Success fill:#99ff99
    style Reject1 fill:#ff9999
    style Reject2 fill:#ff9999
    style Reject3 fill:#ff9999
    style Reject4 fill:#ff9999
    style Error fill:#ff9999
```

### Read Operation (Query)

```mermaid
flowchart TD
    Start[Client: GET /api/products] --> Gateway[API Gateway]
    
    Gateway --> Cache{Redis Cache<br/>Check}
    
    Cache -->|Hit| Return1[Return Cached Data<br/>Fast Response]
    Cache -->|Miss| PS[Product Service]
    
    PS --> Auth[JWT Validation<br/>Optional for Public]
    Auth -->|Valid/Public| Query[ListProductsQuery]
    Auth -->|Invalid| Reject[401 Unauthorized]
    
    Query --> Filter[Apply Filters<br/>category, price, stock]
    Filter --> Repo[Repository.List]
    
    Repo --> SQL[SELECT * FROM products<br/>WHERE ... LIMIT ... OFFSET ...]
    SQL --> Result[Result Set]
    
    Result --> Transform[Transform to DTO]
    Transform --> CacheSet[Set Cache<br/>TTL: 5 minutes]
    CacheSet --> Return2[200 OK + Data]
    
    style Return1 fill:#99ff99
    style Return2 fill:#99ff99
    style Reject fill:#ff9999
```

## Technology Stack

```mermaid
mindmap
  root((Full Observability<br/>Microservices))
    Programming
      Go 1.24
      GORM ORM
      Gorilla Mux
      gRPC
    
    Architecture
      CQRS Pattern
      Event-Driven
      Clean Architecture
      Microservices
      Service Mesh
    
    Infrastructure
      Kubernetes
      Helm Charts
      Istio Service Mesh
      Docker
    
    Databases
      PostgreSQL 15
      Redis 7
      Kafka
    
    Observability
      Prometheus
      Grafana
      Jaeger
      OpenTelemetry
    
    Security
      JWT Authentication
      mTLS Encryption
      RBAC Authorization
      Network Policies
    
    Deployment
      Helm
      Kustomize
      Docker Compose
      Istio Gateway
```

## Service Communication Patterns

### Synchronous Communication (gRPC)

```mermaid
graph LR
    subgraph "Payment Service Orchestration"
        PYS[Payment Service<br/>Orchestrator]
    end
    
    subgraph "Parallel gRPC Calls"
        direction TB
        PYS -->|1. ValidateUser| US[User Service<br/>gRPC Server]
        PYS -->|2. GetProduct| PS[Product Service<br/>gRPC Server]
        PYS -->|3. CheckAvailability| IS[Inventory Service<br/>gRPC Server]
    end
    
    subgraph "Results"
        US -.->|User: Valid ✓| R1[Aggregate Results]
        PS -.->|Product: Available ✓| R1
        IS -.->|Stock: 50 units ✓| R1
    end
    
    R1 --> Decision{All Valid?}
    Decision -->|Yes| Create[Create Payment]
    Decision -->|No| Reject[Reject Request]
    
    style Create fill:#99ff99
    style Reject fill:#ff9999
```

### Asynchronous Communication (Kafka)

```mermaid
sequenceDiagram
    participant P as Payment Service
    participant K as Kafka Broker
    participant I as Inventory Service
    
    Note over P,I: Eventual Consistency Pattern
    
    P->>P: Create Payment<br/>Status: Pending
    P->>P: Commit to Database
    
    P->>K: Publish: ProductPurchasedEvent<br/>{paymentID, productID, quantity}
    K-->>P: ACK (async)
    
    P->>P: Return Response<br/>Don't wait for inventory
    
    Note over K,I: Asynchronous Processing
    K->>I: Deliver Event<br/>(Consumer Group)
    
    I->>I: Process Event<br/>Update Stock
    I->>I: quantity -= purchased
    I->>I: Commit to Database
    
    I->>K: Publish: InventoryUpdatedEvent<br/>{productID, newQuantity}
    
    Note over P,I: Eventually Consistent<br/>Payment created immediately<br/>Inventory updated async
```

## Deployment Commands

### Docker Compose
```bash
# Build all services
docker-compose build

# Start all services
docker-compose up -d

# View logs
docker-compose logs -f user-service

# Stop all services
docker-compose down
```

### Kubernetes with Helm
```bash
# Install (development)
cd helm
./install.sh dev

# Install (production)
./install.sh prod

# Upgrade
helm upgrade full-observability ./full-observability -n observability

# Uninstall
./uninstall.sh
```

### Istio Service Mesh
```bash
# Install Istio
cd helm/istio
./install-istio.sh

# Enable sidecar injection
./enable-injection.sh

# Configure mTLS
./configure-mtls.sh strict

# Test
./test-istio.sh
```

### One-Command Deployment
```bash
# Deploy everything
./deploy-all.sh dev
```

## Project Structure

```mermaid
graph TB
    subgraph "Repository Root"
        CMD[cmd/<br/>Service Entry Points]
        INT[internal/<br/>Business Logic]
        PKG[pkg/<br/>Shared Packages]
        API[api/<br/>Protocol Buffers]
        HELM[helm/<br/>Kubernetes Deployment]
        DOCKER[dockerfiles/<br/>Container Images]
    end
    
    subgraph "cmd/"
        U[user/main.go]
        P[product/main.go]
        I[inventory/main.go]
        PY[payment/main.go]
    end
    
    subgraph "internal/"
        direction LR
        SVC1[user/<br/>delivery, domain<br/>usecase, repository]
        SVC2[product/]
        SVC3[inventory/]
        SVC4[payment/]
    end
    
    subgraph "pkg/"
        AUTH[auth/<br/>JWT, Password]
        DB[database/<br/>PostgreSQL, GORM]
        LOG[logger/<br/>Structured Logging]
        TRACE[tracing/<br/>OpenTelemetry]
    end
    
    subgraph "helm/"
        CHART[full-observability/<br/>Chart + Templates]
        ISTIO[istio/<br/>Service Mesh Config]
        SCRIPTS[*.sh<br/>Deployment Scripts]
    end
    
    CMD --> U
    CMD --> P
    CMD --> I
    CMD --> PY
    
    INT --> SVC1
    INT --> SVC2
    INT --> SVC3
    INT --> SVC4
    
    PKG --> AUTH
    PKG --> DB
    PKG --> LOG
    PKG --> TRACE
    
    HELM --> CHART
    HELM --> ISTIO
    HELM --> SCRIPTS
```

## Service Ports

| Service | HTTP | gRPC | Database | Purpose |
|---------|------|------|----------|---------|
| User Service | 8080 | 9090 | userdb | Authentication, User Management |
| Product Service | 8081 | 9091 | productdb | Product Catalog, Stock |
| Inventory Service | 8082 | 9092 | inventorydb | Stock Management, Kafka Consumer |
| Payment Service | 8083 | - | paymentdb | Payments, Transaction Orchestration |
| API Gateway | 8000 | - | - | Entry Point, Rate Limiting |
| Prometheus | 9090 | - | - | Metrics Collection |
| Grafana | 3000 | - | - | Visualization |
| Jaeger | 16686 | - | - | Distributed Tracing |
| PostgreSQL | 5432 | - | - | Relational Database |
| Redis | 6379 | - | - | Cache, Rate Limiting |
| Kafka | 9093 | - | - | Event Streaming |

## Key Features

- **Microservices Architecture**: Independent, scalable services
- **CQRS Pattern**: Separate read and write operations
- **Event-Driven**: Kafka-based asynchronous communication
- **Service Mesh**: Istio for traffic management, security, observability
- **Observability**: Prometheus metrics, Jaeger tracing, Grafana dashboards
- **Security**: JWT authentication, mTLS encryption, RBAC authorization
- **Resilience**: Circuit breakers, retries, timeouts, health checks
- **Scalability**: Horizontal Pod Autoscaling, load balancing
- **Deployment**: Helm charts, canary deployments, blue-green deployments
- **API Documentation**: Swagger/OpenAPI for all services
