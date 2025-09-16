# Nakama Architecture Overview

Nakama is a distributed, scalable server for social and realtime games and apps. This document provides a comprehensive overview of the system architecture, core components, and design principles.

## High-Level Architecture

```mermaid
graph TB
    Client[Game Clients<br/>Unity, Unreal, Web, Mobile] --> LoadBalancer[Load Balancer]
    LoadBalancer --> Nakama1[Nakama Server 1]
    LoadBalancer --> Nakama2[Nakama Server 2]
    LoadBalancer --> NakamaN[Nakama Server N]
    
    Nakama1 --> Database[(CockroachDB<br/>PostgreSQL)]
    Nakama2 --> Database
    NakamaN --> Database
    
    Nakama1 --> Redis[(Redis<br/>Optional Cache)]
    Nakama2 --> Redis
    NakamaN --> Redis
    
    subgraph "Nakama Server Components"
        API[REST/gRPC API]
        RT[Realtime Engine<br/>WebSockets]
        Matchmaker[Matchmaker]
        Runtime[Runtime Engine<br/>Go/Lua/JS]
        Console[Admin Console]
        Metrics[Metrics & Monitoring]
    end
    
    subgraph "External Services"
        Social[Social Providers<br/>Apple, Google, Facebook]
        IAP[In-App Purchase<br/>Apple, Google, Huawei]
        Push[Push Notifications<br/>FCM, APNS]
    end
    
    Nakama1 -.-> Social
    Nakama1 -.-> IAP
    Nakama1 -.-> Push
```

## Core Design Principles

### 1. Horizontal Scalability
- **Stateless Servers**: Nakama servers are designed to be stateless, allowing horizontal scaling
- **Shared Database**: All servers share a common database for persistent state
- **Session Affinity**: Real-time connections maintain session affinity for optimal performance

### 2. Multi-Protocol Support
- **REST API**: HTTP/1.1 with JSON for web clients and traditional request/response patterns
- **gRPC**: High-performance binary protocol for mobile and native clients
- **WebSockets**: Real-time bidirectional communication for live features
- **rUDP**: Optimized UDP protocol for low-latency gaming scenarios

### 3. Extensibility
- **Runtime Hooks**: Extend server logic with custom code in Go, Lua, or JavaScript
- **Plugin Architecture**: Modular design allows custom integrations
- **Event System**: React to game events with custom logic

## System Layers

```mermaid
graph TB
    subgraph "Client Layer"
        Unity[Unity SDK]
        Unreal[Unreal SDK]
        Web[JavaScript SDK]
        Mobile[Mobile SDKs<br/>iOS, Android]
    end
    
    subgraph "Protocol Layer"
        REST[REST API<br/>HTTP/JSON]
        GRPC[gRPC API<br/>Protocol Buffers]
        WS[WebSocket<br/>Real-time]
        UDP[rUDP<br/>Low Latency]
    end
    
    subgraph "Service Layer"
        Auth[Authentication]
        Social[Social Features]
        Storage[Data Storage]
        Matchmaking[Matchmaking]
        Realtime[Real-time Engine]
        Leaderboards[Leaderboards]
        Runtime[Runtime Engine]
    end
    
    subgraph "Data Layer"
        DB[(Database<br/>CockroachDB/PostgreSQL)]
        Cache[(Cache<br/>Redis)]
        Files[File Storage<br/>Local/Cloud]
    end
    
    Unity --> REST
    Unreal --> GRPC
    Web --> WS
    Mobile --> UDP
    
    REST --> Auth
    GRPC --> Social
    WS --> Storage
    UDP --> Matchmaking
    
    Auth --> DB
    Social --> Cache
    Storage --> Files
```

## Request Flow Architecture

```mermaid
sequenceDiagram
    participant C as Game Client
    participant LB as Load Balancer
    participant N as Nakama Server
    participant RT as Runtime Engine
    participant DB as Database
    
    C->>LB: API Request (REST/gRPC)
    LB->>N: Route to Available Server
    N->>N: Authentication & Validation
    
    alt Runtime Hook Available
        N->>RT: Execute Before Hook
        RT->>N: Hook Result
    end
    
    N->>DB: Database Operation
    DB->>N: Data Response
    
    alt Runtime Hook Available
        N->>RT: Execute After Hook
        RT->>N: Hook Result
    end
    
    N->>LB: API Response
    LB->>C: Return to Client
```

## Data Flow Architecture

```mermaid
graph LR
    subgraph "Data Sources"
        Client[Client Data]
        Runtime[Runtime Generated]
        External[External APIs]
        System[System Events]
    end
    
    subgraph "Processing Layer"
        Validation[Validation & Auth]
        Hooks[Runtime Hooks]
        Business[Business Logic]
        Persistence[Persistence Layer]
    end
    
    subgraph "Storage Layer"
        Users[Users & Accounts]
        Storage[Storage Objects]
        Leaderboards[Leaderboards]
        Matches[Match Data]
        Tournaments[Tournament Data]
        Groups[Groups & Social]
    end
    
    Client --> Validation
    Runtime --> Hooks
    External --> Business
    System --> Persistence
    
    Validation --> Users
    Hooks --> Storage
    Business --> Leaderboards
    Persistence --> Matches
    Business --> Tournaments
    Hooks --> Groups
```

## Key Components Overview

### Authentication & Authorization
- JWT-based session management
- Multi-provider social authentication
- Device-based authentication
- Role-based access control

### Real-time Engine
- WebSocket connection management
- Match-based communication
- Party and group chat
- Live event broadcasting

### Storage Engine
- User-scoped data storage
- Global shared storage
- Conditional writes and transactions
- Storage indexing and queries

### Matchmaking System
- Skill-based matching
- Custom matching criteria
- Real-time and turn-based support
- Authoritative server architecture

### Social Features
- Friend relationships
- Groups and guilds
- Chat systems
- Activity feeds

### Runtime Engine
- Pluggable server-side logic
- Multi-language support (Go, Lua, JavaScript)
- Event-driven architecture
- Custom RPC endpoints

## Scalability Patterns

### Horizontal Scaling
```mermaid
graph TB
    subgraph "Load Balancing"
        Internet[Internet] --> ALB[Application Load Balancer]
        ALB --> N1[Nakama 1]
        ALB --> N2[Nakama 2]
        ALB --> N3[Nakama 3]
    end
    
    subgraph "Database Cluster"
        N1 --> DB1[(CockroachDB Node 1)]
        N2 --> DB2[(CockroachDB Node 2)]
        N3 --> DB3[(CockroachDB Node 3)]
        DB1 -.-> DB2
        DB2 -.-> DB3
        DB3 -.-> DB1
    end
    
    subgraph "Session Management"
        N1 --> Redis1[(Redis Cluster)]
        N2 --> Redis1
        N3 --> Redis1
    end
```

### Geographic Distribution
```mermaid
graph TB
    subgraph "US East"
        USE_LB[Load Balancer] --> USE_N1[Nakama 1]
        USE_LB --> USE_N2[Nakama 2]
        USE_N1 --> USE_DB[(Database)]
        USE_N2 --> USE_DB
    end
    
    subgraph "EU West"
        EUW_LB[Load Balancer] --> EUW_N1[Nakama 1]
        EUW_LB --> EUW_N2[Nakama 2]
        EUW_N1 --> EUW_DB[(Database)]
        EUW_N2 --> EUW_DB
    end
    
    subgraph "Asia Pacific"
        AP_LB[Load Balancer] --> AP_N1[Nakama 1]
        AP_LB --> AP_N2[Nakama 2]
        AP_N1 --> AP_DB[(Database)]
        AP_N2 --> AP_DB
    end
    
    USE_DB -.->|Replication| EUW_DB
    EUW_DB -.->|Replication| AP_DB
    AP_DB -.->|Replication| USE_DB
```

## Performance Characteristics

- **Throughput**: Handles thousands of concurrent connections per server
- **Latency**: Sub-100ms response times for real-time operations
- **Availability**: 99.9%+ uptime with proper clustering
- **Consistency**: Strong consistency for critical game state
- **Partition Tolerance**: Graceful degradation during network issues

## Security Architecture

- **Transport Security**: TLS encryption for all communications
- **Authentication**: Multi-factor authentication support
- **Authorization**: Fine-grained permission system
- **Data Protection**: Encryption at rest and in transit
- **Rate Limiting**: Protection against abuse and DDoS
- **Audit Logging**: Comprehensive security event logging

## Technology Stack

- **Language**: Go (primary server), Lua/JavaScript (runtime)
- **Database**: CockroachDB (primary), PostgreSQL (compatible)
- **Cache**: Redis (optional)
- **Protocols**: HTTP/1.1, HTTP/2, WebSockets, gRPC
- **Serialization**: JSON, Protocol Buffers
- **Monitoring**: Prometheus metrics, structured logging

## Next Steps

For detailed information about specific components:

- [Component Architecture](components.md) - Deep dive into each component
- [Database Architecture](database.md) - Schema design and data modeling
- [Authentication & Authorization](auth.md) - Security implementation details
- [Real-time Communication](realtime.md) - WebSocket and real-time features
- [Runtime Extensions](runtime.md) - Custom server-side logic
- [Deployment Architecture](deployment.md) - Production deployment patterns
- [API Architecture](api.md) - Protocol design and implementation