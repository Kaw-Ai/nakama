# Deployment Architecture

This document details production deployment patterns, infrastructure requirements, monitoring, and operational considerations for Nakama.

## Deployment Overview

```mermaid
graph TB
    subgraph "Infrastructure Layers"
        CDN[CDN Layer<br/>Content Delivery]
        LoadBalancer[Load Balancer<br/>Traffic Distribution]
        WebTier[Web Tier<br/>Nakama Servers]
        DatabaseTier[Database Tier<br/>CockroachDB/PostgreSQL]
        CacheTier[Cache Tier<br/>Redis/In-Memory]
    end
    
    subgraph "External Services"
        DNS[DNS Service<br/>Route 53/CloudFlare]
        Monitoring[Monitoring<br/>Prometheus/Grafana]
        Logging[Logging<br/>ELK/Fluentd]
        Backup[Backup Service<br/>Automated Backups]
    end
    
    subgraph "Security Layer"
        WAF[Web Application Firewall]
        TLS[TLS Termination]
        VPN[VPN Access]
        IAM[Identity & Access Management]
    end
    
    CDN --> LoadBalancer
    LoadBalancer --> WebTier
    WebTier --> DatabaseTier
    WebTier --> CacheTier
    
    DNS --> CDN
    Monitoring --> WebTier
    Logging --> WebTier
    Backup --> DatabaseTier
    
    WAF --> LoadBalancer
    TLS --> LoadBalancer
    VPN --> WebTier
    IAM --> WebTier
```

## Infrastructure Patterns

### 1. Single Region Deployment

```mermaid
graph TB
    subgraph "Availability Zone A"
        ALB_A[Application Load Balancer]
        Nakama_A1[Nakama Server 1]
        Nakama_A2[Nakama Server 2]
        DB_A[Database Node A]
        Redis_A[Redis Instance A]
    end
    
    subgraph "Availability Zone B"
        Nakama_B1[Nakama Server 3]
        Nakama_B2[Nakama Server 4]
        DB_B[Database Node B]
        Redis_B[Redis Instance B]
    end
    
    subgraph "Availability Zone C"
        DB_C[Database Node C]
        Redis_C[Redis Instance C]
    end
    
    Internet[Internet] --> ALB_A
    ALB_A --> Nakama_A1
    ALB_A --> Nakama_A2
    ALB_A --> Nakama_B1
    ALB_A --> Nakama_B2
    
    Nakama_A1 --> DB_A
    Nakama_A1 --> Redis_A
    Nakama_A2 --> DB_B
    Nakama_A2 --> Redis_B
    Nakama_B1 --> DB_C
    Nakama_B1 --> Redis_C
    Nakama_B2 --> DB_A
    Nakama_B2 --> Redis_A
    
    DB_A -.->|Replication| DB_B
    DB_B -.->|Replication| DB_C
    DB_C -.->|Replication| DB_A
    
    Redis_A -.->|Replication| Redis_B
    Redis_B -.->|Replication| Redis_C
```

### 2. Multi-Region Deployment

```mermaid
graph TB
    subgraph "Region US-East"
        DNS_USE[Route 53<br/>us-east.game.com]
        ALB_USE[Load Balancer US-East]
        Nakama_USE[Nakama Cluster US-East]
        DB_USE[Database Cluster US-East]
        Redis_USE[Redis Cluster US-East]
    end
    
    subgraph "Region EU-West"
        DNS_EUW[Route 53<br/>eu-west.game.com]
        ALB_EUW[Load Balancer EU-West]
        Nakama_EUW[Nakama Cluster EU-West]
        DB_EUW[Database Cluster EU-West]
        Redis_EUW[Redis Cluster EU-West]
    end
    
    subgraph "Region AP-Southeast"
        DNS_APS[Route 53<br/>ap-southeast.game.com]
        ALB_APS[Load Balancer AP-Southeast]
        Nakama_APS[Nakama Cluster AP-Southeast]
        DB_APS[Database Cluster AP-Southeast]
        Redis_APS[Redis Cluster AP-Southeast]
    end
    
    subgraph "Global Services"
        GlobalDNS[Global DNS<br/>game.com]
        CDN[CloudFront CDN]
        Monitoring[Global Monitoring]
    end
    
    GlobalDNS --> DNS_USE
    GlobalDNS --> DNS_EUW
    GlobalDNS --> DNS_APS
    
    CDN --> ALB_USE
    CDN --> ALB_EUW
    CDN --> ALB_APS
    
    DB_USE -.->|Cross-Region Replication| DB_EUW
    DB_EUW -.->|Cross-Region Replication| DB_APS
    DB_APS -.->|Cross-Region Replication| DB_USE
    
    Monitoring --> Nakama_USE
    Monitoring --> Nakama_EUW
    Monitoring --> Nakama_APS
```

## Container Orchestration

### 1. Kubernetes Deployment

```mermaid
graph TB
    subgraph "Kubernetes Cluster"
        Master[Master Nodes<br/>Control Plane]
        Worker1[Worker Node 1]
        Worker2[Worker Node 2]
        Worker3[Worker Node 3]
    end
    
    subgraph "Nakama Deployment"
        Deployment[Nakama Deployment<br/>ReplicaSet]
        Service[Nakama Service<br/>Load Balancer]
        ConfigMap[ConfigMap<br/>Configuration]
        Secret[Secret<br/>Credentials]
    end
    
    subgraph "Database Deployment"
        StatefulSet[CockroachDB StatefulSet]
        PVC[Persistent Volume Claims]
        DBService[Database Service]
    end
    
    subgraph "Monitoring Stack"
        Prometheus[Prometheus<br/>Metrics Collection]
        Grafana[Grafana<br/>Visualization]
        AlertManager[Alert Manager<br/>Notifications]
    end
    
    Master --> Worker1
    Master --> Worker2
    Master --> Worker3
    
    Deployment --> Worker1
    Deployment --> Worker2
    Deployment --> Worker3
    
    Service --> Deployment
    ConfigMap --> Deployment
    Secret --> Deployment
    
    StatefulSet --> PVC
    StatefulSet --> Worker1
    StatefulSet --> Worker2
    StatefulSet --> Worker3
    
    Prometheus --> Deployment
    Prometheus --> StatefulSet
    Grafana --> Prometheus
    AlertManager --> Prometheus
```

### 2. Docker Compose Development

```yaml
# docker-compose.yml example
version: '3.8'

services:
  nakama:
    image: heroiclabs/nakama:3.0.0
    entrypoint:
      - "/bin/sh"
      - "-ecx"
      - >
        /nakama/nakama migrate up --database.address root@cockroachdb:26257 &&
        exec /nakama/nakama --name nakama1 --database.address root@cockroachdb:26257 --logger.level INFO --session.token_expiry_sec 7200 --runtime.path /nakama/data/modules
    restart: "unless-stopped"
    links:
      - "cockroachdb:db"
    depends_on:
      - cockroachdb
      - redis
    volumes:
      - ./data:/nakama/data
    expose:
      - "7349"
      - "7350"
      - "7351"
    ports:
      - "7349:7349"
      - "7350:7350"
      - "7351:7351"
    healthcheck:
      test: ["CMD", "/nakama/nakama", "healthcheck"]
      interval: 30s
      timeout: 10s
      retries: 5

  cockroachdb:
    image: cockroachdb/cockroach:latest-v21.2
    command: start-single-node --insecure --store=attrs=ssd,path=/var/lib/cockroach/
    restart: "unless-stopped"
    volumes:
      - data:/var/lib/cockroach
    expose:
      - "8080"
      - "26257"
    ports:
      - "26257:26257"
      - "8080:8080"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health?ready=1"]
      interval: 30s
      timeout: 10s
      retries: 5

  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes
    restart: "unless-stopped"
    volumes:
      - redis:/data
    expose:
      - "6379"
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 30s
      timeout: 10s
      retries: 5

volumes:
  data:
  redis:
```

## Cloud Provider Deployments

### 1. AWS Deployment

```mermaid
graph TB
    subgraph "AWS Services"
        Route53[Route 53<br/>DNS Management]
        CloudFront[CloudFront<br/>CDN]
        ALB[Application Load Balancer]
        ECS[ECS/EKS<br/>Container Service]
        RDS[RDS/Aurora<br/>Database Service]
        ElastiCache[ElastiCache<br/>Redis Service]
        S3[S3<br/>Object Storage]
        CloudWatch[CloudWatch<br/>Monitoring]
    end
    
    subgraph "Security Services"
        WAF[AWS WAF<br/>Web Application Firewall]
        ACM[AWS Certificate Manager<br/>SSL/TLS Certificates]
        IAM[AWS IAM<br/>Identity Management]
        VPC[VPC<br/>Network Isolation]
    end
    
    subgraph "Operational Services"
        Systems_Manager[Systems Manager<br/>Configuration Management]
        Secrets_Manager[Secrets Manager<br/>Secret Storage]
        CodePipeline[CodePipeline<br/>CI/CD]
        ECR[ECR<br/>Container Registry]
    end
    
    Route53 --> CloudFront
    CloudFront --> WAF
    WAF --> ALB
    ALB --> ECS
    ECS --> RDS
    ECS --> ElastiCache
    ECS --> S3
    
    ACM --> ALB
    IAM --> ECS
    VPC --> RDS
    VPC --> ElastiCache
    
    Systems_Manager --> ECS
    Secrets_Manager --> ECS
    CodePipeline --> ECR
    ECR --> ECS
    
    CloudWatch --> ECS
    CloudWatch --> RDS
    CloudWatch --> ElastiCache
```

### 2. Google Cloud Platform Deployment

```mermaid
graph TB
    subgraph "GCP Services"
        CloudDNS[Cloud DNS<br/>DNS Management]
        CloudCDN[Cloud CDN<br/>Content Delivery]
        LoadBalancer[Load Balancer<br/>HTTP(S) LB]
        GKE[GKE<br/>Kubernetes Engine]
        CloudSQL[Cloud SQL<br/>PostgreSQL]
        Memorystore[Memorystore<br/>Redis Service]
        CloudStorage[Cloud Storage<br/>Object Storage]
        Monitoring[Cloud Monitoring<br/>Stackdriver]
    end
    
    subgraph "Security Services"
        CloudArmor[Cloud Armor<br/>DDoS Protection]
        IAM[Cloud IAM<br/>Identity Management]
        VPC[VPC<br/>Network Security]
        KMS[Cloud KMS<br/>Key Management]
    end
    
    subgraph "DevOps Services"
        CloudBuild[Cloud Build<br/>CI/CD]
        ContainerRegistry[Container Registry<br/>Image Storage]
        ConfigManagement[Config Management<br/>GitOps]
        SecretManager[Secret Manager<br/>Secret Storage]
    end
    
    CloudDNS --> CloudCDN
    CloudCDN --> CloudArmor
    CloudArmor --> LoadBalancer
    LoadBalancer --> GKE
    GKE --> CloudSQL
    GKE --> Memorystore
    GKE --> CloudStorage
    
    IAM --> GKE
    VPC --> CloudSQL
    VPC --> Memorystore
    KMS --> SecretManager
    
    CloudBuild --> ContainerRegistry
    ContainerRegistry --> GKE
    ConfigManagement --> GKE
    SecretManager --> GKE
    
    Monitoring --> GKE
    Monitoring --> CloudSQL
    Monitoring --> Memorystore
```

## Configuration Management

### 1. Environment Configuration

```mermaid
classDiagram
    class EnvironmentConfig {
        +string environment
        +DatabaseConfig database
        +ServerConfig server
        +RuntimeConfig runtime
        +LoggingConfig logging
        +MetricsConfig metrics
        +SecurityConfig security
    }
    
    class DatabaseConfig {
        +[]string addresses
        +string username
        +string password
        +string database
        +int maxOpenConns
        +int maxIdleConns
        +duration connMaxLifetime
        +bool sslMode
    }
    
    class ServerConfig {
        +string address
        +int port
        +int grpcPort
        +duration shutdownGraceSec
        +SessionConfig session
        +SocketConfig socket
    }
    
    class RuntimeConfig {
        +string path
        +string entrypoint
        +bool httpKey
        +duration callStackSize
        +int registrySize
        +duration eventQueueSize
    }
    
    EnvironmentConfig --> DatabaseConfig
    EnvironmentConfig --> ServerConfig
    EnvironmentConfig --> RuntimeConfig
```

### 2. Configuration Sources

```mermaid
graph LR
    subgraph "Configuration Sources"
        ConfigFile[Configuration File<br/>YAML/JSON]
        EnvVars[Environment Variables<br/>12-Factor App]
        CommandLine[Command Line Args<br/>CLI Flags]
        ConfigMap[Kubernetes ConfigMap<br/>K8s Native]
        Vault[HashiCorp Vault<br/>Secret Management]
        ConsulKV[Consul KV<br/>Service Discovery]
    end
    
    subgraph "Configuration Precedence"
        Highest[1. Command Line]
        High[2. Environment Variables]
        Medium[3. Configuration File]
        Low[4. Default Values]
    end
    
    subgraph "Secret Management"
        K8sSecrets[Kubernetes Secrets]
        CloudSecrets[Cloud Secret Manager]
        ExternalVault[External Vault]
    end
    
    CommandLine --> Highest
    EnvVars --> High
    ConfigFile --> Medium
    ConfigMap --> Medium
    
    Vault --> K8sSecrets
    ConsulKV --> CloudSecrets
    K8sSecrets --> ExternalVault
```

## Monitoring and Observability

### 1. Monitoring Stack

```mermaid
graph TB
    subgraph "Metrics Collection"
        Prometheus[Prometheus<br/>Metrics Database]
        NodeExporter[Node Exporter<br/>System Metrics]
        NakamaMetrics[Nakama Metrics<br/>Application Metrics]
        DBMetrics[Database Metrics<br/>PostgreSQL/CockroachDB]
    end
    
    subgraph "Visualization"
        Grafana[Grafana<br/>Dashboards]
        Kibana[Kibana<br/>Log Analysis]
        Jaeger[Jaeger<br/>Distributed Tracing]
    end
    
    subgraph "Alerting"
        AlertManager[Alert Manager<br/>Alert Routing]
        PagerDuty[PagerDuty<br/>Incident Management]
        Slack[Slack<br/>Team Notifications]
        Email[Email<br/>Alert Notifications]
    end
    
    subgraph "Log Management"
        Fluentd[Fluentd<br/>Log Collector]
        Elasticsearch[Elasticsearch<br/>Log Storage]
        Logstash[Logstash<br/>Log Processing]
    end
    
    NodeExporter --> Prometheus
    NakamaMetrics --> Prometheus
    DBMetrics --> Prometheus
    Prometheus --> Grafana
    
    Fluentd --> Elasticsearch
    Logstash --> Elasticsearch
    Elasticsearch --> Kibana
    
    Prometheus --> AlertManager
    AlertManager --> PagerDuty
    AlertManager --> Slack
    AlertManager --> Email
    
    NakamaMetrics --> Jaeger
    Jaeger --> Grafana
```

### 2. Key Metrics Dashboard

```mermaid
graph TB
    subgraph "System Metrics"
        CPU[CPU Usage<br/>Per Node]
        Memory[Memory Usage<br/>Available/Used]
        Disk[Disk I/O<br/>IOPS/Latency]
        Network[Network I/O<br/>Bandwidth/Packets]
    end
    
    subgraph "Application Metrics"
        Connections[Active Connections<br/>WebSocket/gRPC]
        APILatency[API Latency<br/>P50/P95/P99]
        ErrorRate[Error Rate<br/>4xx/5xx Responses]
        Throughput[Request Throughput<br/>RPS]
    end
    
    subgraph "Database Metrics"
        DBConnections[DB Connections<br/>Pool Usage]
        QueryLatency[Query Latency<br/>Slow Queries]
        DBThroughput[DB Throughput<br/>QPS/TPS]
        ReplicationLag[Replication Lag<br/>Master/Replica]
    end
    
    subgraph "Business Metrics"
        ActiveUsers[Active Users<br/>Concurrent Players]
        Matches[Active Matches<br/>Running Games]
        Messages[Message Rate<br/>Chat/Real-time]
        Storage[Storage Operations<br/>Read/Write Rate]
    end
```

## Security Hardening

### 1. Network Security

```mermaid
graph TB
    subgraph "Network Layers"
        Internet[Internet<br/>Public Access]
        WAF[Web Application Firewall<br/>Application Layer]
        LoadBalancer[Load Balancer<br/>Traffic Distribution]
        PrivateNetwork[Private Network<br/>VPC/Subnet]
    end
    
    subgraph "Security Controls"
        DDoSProtection[DDoS Protection<br/>Rate Limiting]
        IPWhitelisting[IP Whitelisting<br/>Admin Access]
        TLSTermination[TLS Termination<br/>Encryption]
        NetworkACLs[Network ACLs<br/>Firewall Rules]
    end
    
    subgraph "Internal Security"
        ServiceMesh[Service Mesh<br/>mTLS Communication]
        NetworkPolicies[Network Policies<br/>K8s Segmentation]
        PrivateEndpoints[Private Endpoints<br/>Database Access]
        VPNAccess[VPN Access<br/>Administrative]
    end
    
    Internet --> WAF
    WAF --> LoadBalancer
    LoadBalancer --> PrivateNetwork
    
    DDoSProtection --> WAF
    IPWhitelisting --> LoadBalancer
    TLSTermination --> LoadBalancer
    NetworkACLs --> PrivateNetwork
    
    ServiceMesh --> PrivateNetwork
    NetworkPolicies --> PrivateNetwork
    PrivateEndpoints --> PrivateNetwork
    VPNAccess --> PrivateNetwork
```

### 2. Application Security

```mermaid
graph TB
    subgraph "Authentication Security"
        MultiFactorAuth[Multi-Factor Authentication<br/>TOTP/SMS]
        TokenSecurity[JWT Token Security<br/>Short Expiry/Rotation]
        SessionManagement[Session Management<br/>Secure Storage]
        LoginProtection[Login Protection<br/>Brute Force Prevention]
    end
    
    subgraph "Authorization Security"
        RBAC[Role-Based Access Control<br/>Least Privilege]
        ResourcePermissions[Resource Permissions<br/>Ownership Validation]
        APIGateway[API Gateway<br/>Rate Limiting]
        InputValidation[Input Validation<br/>Sanitization]
    end
    
    subgraph "Data Security"
        EncryptionAtRest[Encryption at Rest<br/>Database/Files]
        EncryptionInTransit[Encryption in Transit<br/>TLS 1.3]
        SecretManagement[Secret Management<br/>Vault/K8s Secrets]
        DataMasking[Data Masking<br/>PII Protection]
    end
    
    subgraph "Runtime Security"
        RuntimeSandboxing[Runtime Sandboxing<br/>Code Isolation]
        ResourceLimits[Resource Limits<br/>CPU/Memory/Time]
        SecurityScanning[Security Scanning<br/>Vulnerability Detection]
        ComplianceAuditing[Compliance Auditing<br/>SOC2/ISO27001]
    end
```

## Backup and Disaster Recovery

### 1. Backup Strategy

```mermaid
graph TB
    subgraph "Backup Types"
        Full[Full Backup<br/>Complete Database]
        Incremental[Incremental Backup<br/>Changed Data Only]
        Transaction[Transaction Log Backup<br/>Point-in-Time Recovery]
        Config[Configuration Backup<br/>System Settings]
    end
    
    subgraph "Backup Schedule"
        Daily[Daily Full Backup<br/>3 AM UTC]
        Hourly[Hourly Incremental<br/>Every Hour]
        Continuous[Continuous WAL<br/>Real-time]
        Weekly[Weekly Config<br/>Sunday 2 AM]
    end
    
    subgraph "Storage Locations"
        LocalStorage[Local Storage<br/>Fast Recovery]
        CloudStorage[Cloud Storage<br/>S3/GCS Buckets]
        CrossRegion[Cross-Region<br/>Geographic Distribution]
        Tape[Tape Storage<br/>Long-term Archive]
    end
    
    subgraph "Retention Policy"
        Daily30[Daily: 30 Days]
        Weekly12[Weekly: 12 Weeks]
        Monthly12[Monthly: 12 Months]
        Yearly7[Yearly: 7 Years]
    end
    
    Full --> Daily
    Incremental --> Hourly
    Transaction --> Continuous
    Config --> Weekly
    
    Daily --> LocalStorage
    Hourly --> CloudStorage
    Continuous --> CrossRegion
    Weekly --> Tape
    
    LocalStorage --> Daily30
    CloudStorage --> Weekly12
    CrossRegion --> Monthly12
    Tape --> Yearly7
```

### 2. Disaster Recovery Plan

```mermaid
sequenceDiagram
    participant Incident as Disaster Event
    participant Monitoring as Monitoring System
    participant OnCall as On-Call Engineer
    participant Team as Recovery Team
    participant Backup as Backup System
    participant Infrastructure as Infrastructure
    
    Incident->>Monitoring: System Failure Detected
    Monitoring->>OnCall: Alert Triggered
    OnCall->>Team: Escalate to Recovery Team
    
    Team->>Team: Assess Impact & Scope
    Team->>Backup: Identify Recovery Point
    Team->>Infrastructure: Provision Recovery Environment
    
    Backup->>Infrastructure: Restore Data
    Infrastructure->>Team: Environment Ready
    Team->>Team: Validate Data Integrity
    
    Team->>Infrastructure: Update DNS/Traffic
    Infrastructure->>Team: Service Restored
    Team->>Monitoring: Confirm Recovery
    
    Team->>Team: Post-Incident Review
    Team->>Team: Update Runbooks
```

## Performance Optimization

### 1. Application Performance

```mermaid
graph TB
    subgraph "Code Optimization"
        ProfilerAnalysis[Profiler Analysis<br/>CPU/Memory Hotspots]
        GarbageCollection[GC Tuning<br/>Memory Management]
        ConcurrencyOptimization[Concurrency Optimization<br/>Goroutine Efficiency]
        AlgorithmOptimization[Algorithm Optimization<br/>Data Structures]
    end
    
    subgraph "Caching Strategy"
        InMemoryCache[In-Memory Cache<br/>Application Cache]
        RedisCache[Redis Cache<br/>Distributed Cache]
        CDNCache[CDN Cache<br/>Static Content]
        DatabaseCache[Database Cache<br/>Query Results]
    end
    
    subgraph "Database Optimization"
        IndexOptimization[Index Optimization<br/>Query Performance]
        QueryOptimization[Query Optimization<br/>Execution Plans]
        ConnectionPooling[Connection Pooling<br/>Resource Efficiency]
        Partitioning[Table Partitioning<br/>Data Distribution]
    end
    
    subgraph "Network Optimization"
        LoadBalancing[Load Balancing<br/>Traffic Distribution]
        CompressionOptimization[Compression<br/>Bandwidth Reduction]
        KeepAlive[HTTP Keep-Alive<br/>Connection Reuse]
        CDNOptimization[CDN Optimization<br/>Geographic Distribution]
    end
```

### 2. Scaling Strategies

```mermaid
graph LR
    subgraph "Horizontal Scaling"
        LoadBalancer[Load Balancer<br/>Traffic Distribution]
        AutoScaling[Auto Scaling<br/>Dynamic Capacity]
        ServiceMesh[Service Mesh<br/>Microservices]
        Sharding[Data Sharding<br/>Database Partitioning]
    end
    
    subgraph "Vertical Scaling"
        CPUUpgrade[CPU Upgrade<br/>More Cores]
        MemoryUpgrade[Memory Upgrade<br/>More RAM]
        StorageUpgrade[Storage Upgrade<br/>Faster Disks]
        NetworkUpgrade[Network Upgrade<br/>Higher Bandwidth]
    end
    
    subgraph "Caching & CDN"
        ApplicationCache[Application Cache<br/>In-Memory]
        DatabaseCache[Database Cache<br/>Query Results]
        ContentCache[Content Cache<br/>Static Assets]
        EdgeCache[Edge Cache<br/>Geographic]
    end
    
    LoadBalancer --> AutoScaling
    AutoScaling --> ServiceMesh
    ServiceMesh --> Sharding
    
    CPUUpgrade --> MemoryUpgrade
    MemoryUpgrade --> StorageUpgrade
    StorageUpgrade --> NetworkUpgrade
    
    ApplicationCache --> DatabaseCache
    DatabaseCache --> ContentCache
    ContentCache --> EdgeCache
```

## Cost Optimization

### 1. Resource Optimization

```mermaid
pie title Infrastructure Cost Distribution
    "Compute (Servers)" : 45
    "Database" : 25
    "Storage" : 15
    "Network" : 10
    "Monitoring" : 5
```

### 2. Cost Management Strategies

```mermaid
graph TB
    subgraph "Compute Optimization"
        RightSizing[Right-sizing<br/>Appropriate Instance Types]
        SpotInstances[Spot Instances<br/>Cost Savings]
        ReservedInstances[Reserved Instances<br/>Long-term Commitment]
        AutoScaling[Auto Scaling<br/>Dynamic Capacity]
    end
    
    subgraph "Storage Optimization"
        StorageTiering[Storage Tiering<br/>Hot/Warm/Cold]
        DataCompression[Data Compression<br/>Reduce Size]
        LifecyclePolicies[Lifecycle Policies<br/>Automated Cleanup]
        BackupOptimization[Backup Optimization<br/>Efficient Retention]
    end
    
    subgraph "Network Optimization"
        CDNUsage[CDN Usage<br/>Reduce Bandwidth]
        DataTransferOptimization[Data Transfer<br/>Regional Optimization]
        CompressionEnabled[Compression<br/>Reduce Traffic]
        CachingStrategy[Caching Strategy<br/>Reduce Requests]
    end
    
    subgraph "Monitoring & Analysis"
        CostAnalysis[Cost Analysis<br/>Regular Reviews]
        ResourceTagging[Resource Tagging<br/>Cost Attribution]
        BudgetAlerts[Budget Alerts<br/>Overspend Prevention]
        UsageOptimization[Usage Optimization<br/>Eliminate Waste]
    end
```

For more information on related topics:
- [Component Architecture](components.md) - Deployment component details
- [Database Architecture](database.md) - Database deployment patterns
- [Authentication & Authorization](auth.md) - Security deployment considerations

## Fork-Specific Deployment Assets

This repository ships concrete deployment assets that implement the patterns
described above:

- **Release pipeline**: [`.github/workflows/release.yml`](../../.github/workflows/release.yml)
  builds multi-arch (linux/amd64, linux/arm64) container images stamped with
  version and commit metadata, and publishes them to GHCR as
  `ghcr.io/kaw-ai/nakama`.
- **Security gates**: [`.github/workflows/security.yml`](../../.github/workflows/security.yml)
  runs govulncheck and Trivy dependency scanning.
- **Per-environment configuration**: [`deploy/config/`](../../deploy/config/)
  contains dev, staging, and production configuration templates with
  secret-injection guidance.
- **Kubernetes manifests**: [`deploy/kubernetes/`](../../deploy/kubernetes/)
  provides a pre-deploy migration job, the server deployment with readiness/
  liveness probes and connection-drain settings, and internal/external
  services.
- **Runbooks**: [`docs/operations/runbooks.md`](../operations/runbooks.md)
  documents restart, deploy, rollback, config change, scaling, and backup
  procedures, plus monitoring and alerting guidance.

See [`deploy/README.md`](../../deploy/README.md) for the full deployment guide,
including database strategy, runtime module packaging, and network topology.