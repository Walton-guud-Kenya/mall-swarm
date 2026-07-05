# mall-swarm Architecture Documentation

> Complete microservices e-commerce platform architecture with comprehensive Mermaid diagrams

## Table of Contents
1. [System Architecture Overview](#system-architecture-overview)
2. [Request Flow & Service Communication](#request-flow--service-communication)
3. [Module Dependencies & Organization](#module-dependencies--organization)
4. [Technology Stack by Layer](#technology-stack-by-layer)
5. [Data Flow Example](#data-flow-example)
6. [Deployment Architecture](#deployment-architecture)
7. [Configuration & Service Discovery](#configuration--service-discovery-flow)
8. [Security Architecture](#security-architecture)
9. [Environment Setup](#environment-setup-checklist)
10. [Port Mapping](#port-mapping)

---

## System Architecture Overview

This diagram illustrates the high-level system architecture showing all major components and their interactions:

```mermaid
graph TB
    subgraph "Client Layer"
        AdminWeb["🖥️ Admin Web<br/>(Vue-based)"]
        MobileApp["📱 Mobile App<br/>(uni-app)"]
        Browser["🌐 Browser Client"]
    end
    
    subgraph "Infrastructure & Gateway"
        Gateway["🚪 API Gateway<br/>Spring Cloud Gateway<br/>Port: 8201"]
        Nginx["⚙️ Nginx Reverse Proxy<br/>Load Balancer"]
    end
    
    subgraph "Service Discovery & Config"
        Nacos["📋 Nacos Registry & Config<br/>Port: 8848"]
    end
    
    subgraph "Microservices"
        Auth["🔐 mall-auth<br/>Authentication Service<br/>Port: 8401<br/>Sa-Token + JWT"]
        Admin["👨‍💼 mall-admin<br/>Backend Admin<br/>Port: 8080<br/>Products, Orders, Users"]
        Search["🔍 mall-search<br/>Search Service<br/>Port: 8081<br/>Elasticsearch Integration"]
        Portal["🛍️ mall-portal<br/>Frontend Commerce<br/>Port: 8082<br/>Cart, Orders, Payments"]
        Demo["🧪 mall-demo<br/>Testing Service<br/>Inter-service Calls"]
    end
    
    subgraph "Data & Cache Layer"
        MySQL["🗄️ MySQL Database<br/>Version: 5.7<br/>Business Data"]
        Redis["💾 Redis Cache<br/>Version: 7.0<br/>Session & Cache"]
        MongoDB["📊 MongoDB<br/>Version: 5.0<br/>User Activity Logs"]
        ES["🔎 Elasticsearch<br/>Version: 7.17.3<br/>Product Search Index"]
    end
    
    subgraph "Messaging & Integration"
        RabbitMQ["📬 RabbitMQ<br/>Version: 3.10.5<br/>Async Events"]
        OSS["☁️ Aliyun OSS<br/>MinIO Storage<br/>File Upload"]
    end
    
    subgraph "Monitoring & Logging"
        Admin_Monitor["📈 Spring Boot Admin<br/>Port: 8101<br/>Service Monitoring"]
        Kibana["📊 Kibana<br/>Port: 5601<br/>Log Visualization"]
        Logstash["📝 Logstash<br/>Log Collection & Processing"]
    end
    
    subgraph "Deployment"
        Docker["🐳 Docker<br/>Container Runtime"]
        K8s["☸️ Kubernetes<br/>Orchestration"]
    end
    
    Browser --> Nginx
    AdminWeb --> Nginx
    MobileApp --> Nginx
    Nginx --> Gateway
    
    Gateway --> Auth
    Gateway --> Admin
    Gateway --> Search
    Gateway --> Portal
    Auth --> Admin
    Auth --> Portal
    
    Auth -.Register.-> Nacos
    Admin -.Register.-> Nacos
    Search -.Register.-> Nacos
    Portal -.Register.-> Nacos
    Demo -.Register.-> Nacos
    
    Auth -.Pull Config.-> Nacos
    Admin -.Pull Config.-> Nacos
    Search -.Pull Config.-> Nacos
    Portal -.Pull Config.-> Nacos
    
    Admin --> MySQL
    Portal --> MySQL
    Auth --> MySQL
    Search --> MySQL
    
    Admin --> Redis
    Portal --> Redis
    Auth --> Redis
    Gateway --> Redis
    
    Search --> ES
    Portal --> MongoDB
    
    Admin --> RabbitMQ
    Portal --> RabbitMQ
    RabbitMQ -.Async Events.-> Portal
    
    Admin --> OSS
    Portal --> OSS
    
    Admin --> Admin_Monitor
    Portal --> Admin_Monitor
    Auth --> Admin_Monitor
    Search --> Admin_Monitor
    Gateway --> Admin_Monitor
    
    Logstash --> Admin
    Logstash --> Portal
    Logstash --> Auth
    Logstash --> Search
    Logstash --> Kibana
    
    Docker -.Containerize.-> Auth
    Docker -.Containerize.-> Admin
    Docker -.Containerize.-> Search
    Docker -.Containerize.-> Portal
    K8s -.Orchestrate.-> Docker
    
    style Gateway fill:#ff9999
    style Auth fill:#99ccff
    style Admin fill:#99ff99
    style Search fill:#ffcc99
    style Portal fill:#ff99cc
    style MySQL fill:#ccccff
    style Redis fill:#ffcccc
    style ES fill:#ccffcc
    style Nacos fill:#ffffcc
```

**Key Components:**
- **Clients**: Admin web portal, mobile app, and browsers
- **Gateway Layer**: Single entry point for all requests with routing and authentication
- **Microservices**: Independent, loosely coupled business services
- **Data Stores**: Multiple specialized databases for different requirements
- **Message Queue**: Asynchronous event processing
- **Monitoring**: Centralized monitoring and logging infrastructure
- **Deployment**: Container and orchestration platforms

---

## Request Flow & Service Communication

This sequence diagram shows how requests flow through the system:

```mermaid
sequenceDiagram
    participant Client as Client / Browser
    participant Gateway as API Gateway
    participant Auth as Auth Service
    participant Admin as Admin Service
    participant Redis as Redis Cache
    participant MySQL as MySQL DB
    participant RabbitMQ as RabbitMQ
    
    Client->>Gateway: 1. HTTP Request
    Note over Gateway: Route request<br/>Check auth headers
    
    Gateway->>Auth: 2. Validate Token<br/>(Sa-Token)
    Auth->>Redis: 3. Check Token<br/>in Cache
    Redis-->>Auth: 4. Token Valid
    Auth-->>Gateway: 5. Return Permissions
    
    Note over Gateway: Add user context<br/>to request
    
    Gateway->>Admin: 6. Route to Service
    Admin->>MySQL: 7. Query Data<br/>with MyBatis
    MySQL-->>Admin: 8. Return Data
    
    Admin->>RabbitMQ: 9. Publish Event<br/>(async)
    Note over RabbitMQ: Event queued
    
    Admin->>Redis: 10. Cache Result
    Admin-->>Gateway: 11. Return Response
    Gateway-->>Client: 12. Send to Client
    
    RabbitMQ->>Admin: 13. Event Processing<br/>(background)
```

**Flow Description:**
1. Incoming HTTP request from client through gateway
2. Gateway validates authentication token with auth service
3. Token verification against Redis cache for performance
4. Gateway adds user context and routes to appropriate service
5. Service processes business logic using MyBatis ORM
6. Optional: Publish async events for background processing
7. Cache layer stores frequently accessed data
8. Response returned through gateway to client

---

## Module Dependencies & Organization

```mermaid
graph LR
    subgraph "Shared Libraries"
        Common["mall-common<br/>─────────<br/>• Utils & Constants<br/>• Exception Handling<br/>• Redis Service<br/>• Global Filters<br/>• Authorization Checks"]
        MBG["mall-mbg<br/>─────────<br/>• Data Models<br/>• MyBatis Generated<br/>• DAO Interfaces<br/>• Entity POJOs"]
    end
    
    subgraph "Core Services"
        Auth["mall-auth<br/>─────────<br/>Login & Token<br/>Sa-Token + JWT<br/>OpenFeign Clients<br/>to Admin & Portal"]
        Admin["mall-admin<br/>─────────<br/>Depends: Common, MBG<br/>Product, Order,<br/>Brand Management<br/>REST Controllers"]
        Search["mall-search<br/>─────────<br/>Depends: Common, MBG<br/>Elasticsearch<br/>Integration<br/>Product Search"]
        Portal["mall-portal<br/>─────────<br/>Depends: Common, MBG<br/>Cart, Orders, SSO<br/>MongoDB, RabbitMQ<br/>Alipay Integration"]
    end
    
    subgraph "Testing & Monitoring"
        Demo["mall-demo<br/>─────────<br/>Depends: Common<br/>Tests OpenFeign calls<br/>Service-to-service<br/>communication demo"]
        Monitor["mall-monitor<br/>─────────<br/>Spring Boot Admin<br/>Monitoring Dashboard<br/>Service Health Check"]
    end
    
    Common -.->|Used by| Admin
    Common -.->|Used by| Search
    Common -.->|Used by| Portal
    Common -.->|Used by| Auth
    Common -.->|Used by| Demo
    
    MBG -.->|Used by| Admin
    MBG -.->|Used by| Search
    MBG -.->|Used by| Portal
    
    Auth -->|Feign Client<br/>UmsAdminService| Admin
    Auth -->|Feign Client<br/>UmsMemberService| Portal
    
    Admin -->|Feign Client| Search
    Admin -->|Feign Client| Portal
    
    Demo -->|Feign Client| Admin
    Demo -->|Feign Client| Search
    Demo -->|Feign Client| Portal
    
    Admin -.->|Monitored by| Monitor
    Portal -.->|Monitored by| Monitor
    Auth -.->|Monitored by| Monitor
    Search -.->|Monitored by| Monitor
    
    style Common fill:#e1f5ff
    style MBG fill:#e1f5ff
    style Auth fill:#c8e6c9
    style Admin fill:#fff9c4
    style Search fill:#ffe0b2
    style Portal fill:#f8bbd0
    style Demo fill:#e0bee7
    style Monitor fill:#b2dfdb
```

**Module Responsibilities:**

| Module | Purpose |
|--------|---------|
| **mall-common** | Shared utilities, exception handling, Redis operations, authentication constants |
| **mall-mbg** | Auto-generated MyBatis code, entity models, DAOs |
| **mall-auth** | Token generation, validation, and authorization |
| **mall-admin** | Backend admin operations: products, brands, orders |
| **mall-search** | Elasticsearch integration for product search |
| **mall-portal** | Frontend e-commerce: cart, orders, payments |
| **mall-demo** | Testing service-to-service communication |
| **mall-monitor** | Spring Boot Admin for centralized monitoring |

---

## Technology Stack by Layer

```mermaid
graph TB
    subgraph "Presentation Layer"
        PL["<b>Frontend</b><br/>Vue.js + Element UI<br/>Vue Router + Vuex<br/>Axios HTTP Client<br/>Responsive Design"]
    end
    
    subgraph "API Gateway Layer"
        GL["<b>Spring Cloud Gateway</b><br/>WebFlux Reactive<br/>Request Routing<br/>Rate Limiting<br/>Sa-Token Filter"]
    end
    
    subgraph "Service Layer"
        SL1["<b>Spring Boot 3.5</b><br/>Latest stable version"]
        SL2["<b>Spring Cloud 2025</b><br/>Latest release train"]
        SL3["<b>Spring Cloud Alibaba</b><br/>Nacos Discovery<br/>Dynamic Configuration"]
        SL4["<b>Sa-Token</b><br/>Auth & Authorization<br/>JWT + Redis"]
        SL5["<b>OpenFeign</b><br/>HTTP Client<br/>Service Calls"]
    end
    
    subgraph "Business Logic Layer"
        BL1["💼 mall-admin"]
        BL2["🛍️ mall-portal"]
        BL3["🔍 mall-search"]
        BL4["🔐 mall-auth"]
    end
    
    subgraph "Data Access Layer"
        DAL1["<b>MyBatis 3.5</b><br/>ORM Framework"]
        DAL2["<b>MyBatisGenerator</b><br/>Code Gen"]
        DAL3["<b>Druid</b><br/>Connection Pool"]
        DAL4["<b>PageHelper</b><br/>Pagination"]
    end
    
    subgraph "Data Storage Layer"
        DSL1["🗄️ MySQL 5.7<br/>Relational Data"]
        DSL2["🔎 Elasticsearch 7.17<br/>Search Index"]
        DSL3["📊 MongoDB 5.0<br/>NoSQL Logs"]
        DSL4["💾 Redis 7.0<br/>Cache & Sessions"]
    end
    
    subgraph "Cross-Cutting Concerns"
        CCC1["📬 RabbitMQ 3.10<br/>Async Messaging"]
        CCC2["📝 Logstash 7.17<br/>Log Collection"]
        CCC3["📊 Kibana 7.17<br/>Log Analysis"]
        CCC4["📈 Spring Boot Admin<br/>Monitoring"]
        CCC5["☁️ Aliyun OSS / MinIO<br/>File Storage"]
    end
    
    subgraph "Infrastructure"
        INF1["🐳 Docker<br/>Containerization"]
        INF2["☸️ Kubernetes<br/>Orchestration"]
        INF3["🖇️ Nginx<br/>Reverse Proxy"]
    end
    
    PL --> GL
    GL --> SL1
    SL1 --> SL2
    SL2 --> SL3
    SL1 --> SL4
    SL1 --> SL5
    
    SL5 --> BL1
    SL5 --> BL2
    SL5 --> BL3
    SL4 --> BL4
    
    BL1 --> DAL1
    BL2 --> DAL1
    BL3 --> DAL1
    BL4 --> DAL1
    DAL1 --> DAL2
    DAL1 --> DAL3
    DAL1 --> DAL4
    
    DAL1 --> DSL1
    DAL1 --> DSL2
    DAL1 --> DSL3
    DAL1 --> DSL4
    
    BL1 -.-> CCC1
    BL2 -.-> CCC1
    BL1 -.-> CCC2
    BL2 -.-> CCC2
    BL3 -.-> CCC2
    CCC2 --> CCC3
    BL1 -.-> CCC4
    BL2 -.-> CCC4
    BL1 -.-> CCC5
    BL2 -.-> CCC5
    
    SL1 --> INF1
    INF1 --> INF2
    PL --> INF3
    
    style PL fill:#ffe0b2
    style GL fill:#ff9999
    style SL1 fill:#99ccff
    style SL2 fill:#99ccff
    style SL3 fill:#99ccff
    style SL4 fill:#99ccff
    style SL5 fill:#99ccff
    style BL1 fill:#99ff99
    style BL2 fill:#99ff99
    style BL3 fill:#99ff99
    style BL4 fill:#99ff99
    style DAL1 fill:#ccccff
    style DAL2 fill:#ccccff
    style DAL3 fill:#ccccff
    style DAL4 fill:#ccccff
    style DSL1 fill:#ffcccc
    style DSL2 fill:#ffcccc
    style DSL3 fill:#ffcccc
    style DSL4 fill:#ffcccc
    style CCC1 fill:#ffffcc
    style CCC2 fill:#ffffcc
    style CCC3 fill:#ffffcc
    style CCC4 fill:#ffffcc
    style CCC5 fill:#ffffcc
    style INF1 fill:#e0bee7
    style INF2 fill:#e0bee7
    style INF3 fill:#e0bee7
```

---

## Data Flow Example

This diagram shows a complete product search flow through multiple services:

```mermaid
graph TD
    A["👤 User searches for product"] -->|Keywords| B["🌐 Browser request<br/>GET /esProduct/search"]
    B -->|HTTPS| C["🚪 API Gateway<br/>Port 8201"]
    C -->|Route request| D["🔍 mall-search Service<br/>Port 8081"]
    
    D -->|Query metadata| E["🗄️ MySQL<br/>Product catalog"]
    E -->|Product IDs, names| D
    
    D -->|Full-text search| F["🔎 Elasticsearch<br/>Inverted index"]
    F -->|Matching docs| D
    
    D -->|Store result| G["💾 Redis Cache<br/>TTL: 1 hour"]
    G -->|Cached| D
    
    D -->|Format response| H["JSON Results<br/>Products with filters"]
    H -->|Return response| C
    C -->|HTTPS| B
    B -->|Display results| A
    
    D -->|Async log| I["📝 Logstash<br/>Collect logs"]
    I -->|Index| J["📊 Kibana<br/>Visualize"]
    
    D -.->|Health check| K["📈 Spring Boot Admin<br/>Monitor metrics"]
    
    style A fill:#ffe0b2
    style B fill:#ffe0b2
    style C fill:#ff9999
    style D fill:#ffcc99
    style E fill:#ccccff
    style F fill:#ccffcc
    style G fill:#ffcccc
    style H fill:#e0f2f1
    style I fill:#ffffcc
    style J fill:#e0bee7
    style K fill:#c8e6c9
```

---

## Deployment Architecture

```mermaid
graph TB
    subgraph "Source Control"
        Git["📝 GitHub Repository<br/>Source Code"]
    end
    
    subgraph "CI/CD Pipeline"
        Trigger["🔔 Webhook Trigger<br/>On Push/PR"]
        Jenkins["🔄 Jenkins<br/>Automation Server"]
        Build["🔨 Maven Build<br/>Compile & Test"]
        Docker["🐳 Docker Build<br/>Create Image"]
        Registry["📦 Registry Push<br/>Docker Hub"]
    end
    
    subgraph "Kubernetes Cluster"
        Ingress["🚪 Ingress Controller<br/>External Access"]
        
        subgraph "Services"
            Svc1["Service: mall-auth"]
            Svc2["Service: mall-admin"]
            Svc3["Service: mall-search"]
            Svc4["Service: mall-portal"]
            Svc5["Service: mall-gateway"]
        end
        
        subgraph "Pods"
            Pod1["Pod: mall-auth<br/>Replica: 3"]
            Pod2["Pod: mall-admin<br/>Replica: 2"]
            Pod3["Pod: mall-search<br/>Replica: 2"]
            Pod4["Pod: mall-portal<br/>Replica: 3"]
            Pod5["Pod: mall-gateway<br/>Replica: 2"]
        end
        
        subgraph "Persistent Storage"
            PVC["📦 Persistent Volumes<br/>MySQL, MongoDB, ES"]
        end
    end
    
    subgraph "Infrastructure Services"
        LB["⚖️ Load Balancer<br/>Distribute Traffic"]
        Monitor["📊 Prometheus<br/>Metrics Collection"]
        Logs["📝 ELK Stack<br/>Centralized Logs"]
    end
    
    Git --> Trigger
    Trigger --> Jenkins
    Jenkins --> Build
    Build --> Docker
    Docker --> Registry
    
    Registry -->|Pull Image| Pod1
    Registry -->|Pull Image| Pod2
    Registry -->|Pull Image| Pod3
    Registry -->|Pull Image| Pod4
    Registry -->|Pull Image| Pod5
    
    Pod1 --> Svc1
    Pod2 --> Svc2
    Pod3 --> Svc3
    Pod4 --> Svc4
    Pod5 --> Svc5
    
    Svc1 --> PVC
    Svc2 --> PVC
    Svc3 --> PVC
    Svc4 --> PVC
    
    LB --> Ingress
    Ingress -->|Route| Svc5
    
    Svc1 -.->|Metrics| Monitor
    Svc2 -.->|Metrics| Monitor
    Svc3 -.->|Metrics| Monitor
    Svc4 -.->|Metrics| Monitor
    Svc5 -.->|Metrics| Monitor
    
    Svc1 -.->|Logs| Logs
    Svc2 -.->|Logs| Logs
    Svc3 -.->|Logs| Logs
    Svc4 -.->|Logs| Logs
    Svc5 -.->|Logs| Logs
    
    style Git fill:#99ff99
    style Jenkins fill:#ffcc99
    style Build fill:#ffcc99
    style Docker fill:#99ccff
    style Registry fill:#ff99cc
    style Ingress fill:#ff9999
    style Pod1 fill:#e0bee7
    style Pod2 fill:#e0bee7
    style Pod3 fill:#e0bee7
    style Pod4 fill:#e0bee7
    style Pod5 fill:#e0bee7
    style Monitor fill:#c8e6c9
    style Logs fill:#ffffcc
```

---

## Configuration & Service Discovery Flow

```mermaid
graph LR
    subgraph "Service Startup"
        Start["🚀 Service Start"]
        Init["⚙️ Load application.yml"]
        Connect["🔗 Connect to Nacos:8848"]
    end
    
    subgraph "Nacos Server"
        NS["Namespace:<br/>default"]
        GRP["Group:<br/>DEFAULT_GROUP"]
        Reg["📋 Registry<br/>Mall Services"]
        Cfg["⚙️ Config Center<br/>Dynamic Config"]
    end
    
    subgraph "Service Runtime"
        Heart["❤️ Heartbeat<br/>Every 5s"]
        Listen["👂 Config Listener<br/>Watch Changes"]
        Cache["💾 Local Cache<br/>Offline Resilience"]
        Use["✅ Use Config<br/>DataSource, Redis, etc"]
    end
    
    Start --> Init
    Init --> Connect
    Connect --> Reg
    Connect --> Cfg
    
    Reg -->|Register<br/>Instance| NS
    Cfg -->|Pull Config| GRP
    
    Heart -.->|Periodic| Reg
    Listen -.->|Subscribe| Reg
    Listen -.->|Subscribe| Cfg
    
    Reg -->|Service List| Cache
    Cfg -->|Configuration| Cache
    Cache --> Use
    
    style Start fill:#c8e6c9
    style Init fill:#c8e6c9
    style Connect fill:#c8e6c9
    style NS fill:#ffffcc
    style GRP fill:#ffffcc
    style Reg fill:#ffffcc
    style Cfg fill:#ffffcc
    style Heart fill:#99ccff
    style Listen fill:#99ccff
    style Cache fill:#99ccff
    style Use fill:#99ccff
```

---

## Security Architecture

```mermaid
graph TB
    Client["👤 Client Request"]
    
    Client -->|HTTPS| GW["🚪 API Gateway<br/>TLS/SSL"]
    
    GW -->|Validate<br/>Headers| RateLimit["⚖️ Rate Limiting<br/>Prevent Abuse"]
    
    RateLimit -->|Extract<br/>Token| Auth["🔐 Sa-Token Auth<br/>JWT Validation"]
    
    Auth -->|Lookup| Cache["💾 Redis<br/>Token Cache<br/>TTL: 30 min"]
    
    Cache -->|Valid<br/>Token| Authz["✅ Authorization<br/>Role-Based<br/>Permission Check"]
    
    Authz -->|Add<br/>Context| Service["🔄 Route to Service<br/>User Info Header"]
    
    Service -->|Log<br/>Activity| Audit["📝 Audit Log<br/>Who, What, When"]
    
    Audit -->|Store| ELK["📊 ELK Stack<br/>Compliance<br/>Monitoring"]
    
    style Client fill:#ffe0b2
    style GW fill:#ff9999
    style RateLimit fill:#99ccff
    style Auth fill:#c8e6c9
    style Cache fill:#ffcccc
    style Authz fill:#c8e6c9
    style Service fill:#99ff99
    style Audit fill:#ffffcc
    style ELK fill:#e0bee7
```

---

## Environment Setup Checklist

| Component | Version | Port | Status |
|-----------|---------|------|--------|
| **JDK** | 17 | - | ✅ Required |
| **Maven** | 3.8+ | - | ✅ Required |
| **MySQL** | 5.7 | 3306 | ✅ Primary DB |
| **Redis** | 7.0 | 6379 | ✅ Cache & Session |
| **Elasticsearch** | 7.17.3 | 9200 | ✅ Search Engine |
| **Kibana** | 7.17.3 | 5601 | ✅ Log Visualization |
| **Logstash** | 7.17.3 | 5000 | ✅ Log Collection |
| **MongoDB** | 5.0 | 27017 | ✅ Activity Logs |
| **RabbitMQ** | 3.10.5 | 5672 | ✅ Message Broker |
| **Nacos** | Latest | 8848 | ✅ Registry & Config |
| **Nginx** | 1.22 | 80, 443 | ✅ Reverse Proxy |
| **Docker** | Latest | - | ✅ Containerization |
| **Kubernetes** | 1.24+ | - | ⚙️ Optional |

---

## Port Mapping

| Service | Port | Protocol | Type | Purpose |
|---------|------|----------|------|---------|
| **mall-gateway** | 8201 | HTTP/HTTPS | Public | API Entry Point |
| **mall-admin** | 8080 | HTTP | Internal | Admin Backend |
| **mall-portal** | 8082 | HTTP | Internal | Commerce Backend |
| **mall-search** | 8081 | HTTP | Internal | Search Service |
| **mall-auth** | 8401 | HTTP | Internal | Auth Service |
| **mall-monitor** | 8101 | HTTP | Internal | Monitoring Dashboard |
| **Nacos Console** | 8848 | HTTP | Internal | Service Registry |
| **MySQL** | 3306 | TCP | Internal | Database |
| **Redis** | 6379 | TCP | Internal | Cache Store |
| **Elasticsearch** | 9200 | HTTP | Internal | Search Engine |
| **Kibana** | 5601 | HTTP | Internal | Log Dashboard |
| **RabbitMQ** | 5672 | AMQP | Internal | Message Queue |
| **RabbitMQ Admin** | 15672 | HTTP | Internal | Queue Management |

---

## Key Architecture Principles

### 1. **Microservices Pattern**
- Independent, loosely coupled services
- Each service owns its database
- Service-to-service communication via OpenFeign

### 2. **Service Discovery**
- Dynamic registration with Nacos
- Load balancing across instances
- Automatic health checks

### 3. **Authentication & Authorization**
- Centralized auth service (mall-auth)
- Token-based (JWT) with Redis caching
- Sa-Token framework for fine-grained permissions

### 4. **Data Consistency**
- MySQL for transactional data
- Redis for cache-aside pattern
- MongoDB for activity logs
- Elasticsearch for search indices

### 5. **Asynchronous Processing**
- RabbitMQ for event-driven architecture
- Publish-subscribe messaging patterns
- Background job processing

### 6. **Observability**
- Centralized logging (Logstash + Kibana)
- Application monitoring (Spring Boot Admin)
- Distributed tracing capabilities

### 7. **Deployment & Scaling**
- Docker containerization
- Kubernetes orchestration
- Horizontal scaling support
- CI/CD automation with Jenkins

---

## Quick Start

### Local Development Setup

```bash
# 1. Clone repository
git clone https://github.com/Walton-guud-Kenya/mall-swarm.git
cd mall-swarm

# 2. Build project
mvn clean install -DskipTests

# 3. Start services (requires infrastructure setup)
# See CONTRIBUTING.md for detailed setup instructions

# 4. Access services
# Admin: http://localhost:8080
# Gateway: http://localhost:8201
# Nacos: http://localhost:8848
# Kibana: http://localhost:5601
```

### Production Deployment

```bash
# 1. Build Docker images
mvn clean package -DskipTests -Ddocker.build=true

# 2. Push to registry
docker push your-registry/mall-admin:1.0
# ... push other services

# 3. Deploy to Kubernetes
kubectl apply -f document/k8s/

# 4. Verify deployment
kubectl get pods -n default
kubectl get services -n default
```

---

## References

- **Official Documentation**: [https://cloud.macrozheng.com](https://cloud.macrozheng.com)
- **GitHub Repository**: [https://github.com/macrozheng/mall-swarm](https://github.com/macrozheng/mall-swarm)
- **Learning Tutorials**: [https://cloud.macrozheng.com/video/](https://cloud.macrozheng.com/video/)
- **Spring Cloud Guide**: [https://spring.io/projects/spring-cloud](https://spring.io/projects/spring-cloud)
- **Kubernetes Docs**: [https://kubernetes.io/docs/](https://kubernetes.io/docs/)

---

**Document Version**: 1.0  
**Last Updated**: 2025-07-05  
**Maintainer**: mall-swarm community
