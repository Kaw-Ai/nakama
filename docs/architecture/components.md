# Component Architecture

This document provides a detailed breakdown of Nakama's core components, their responsibilities, and interactions.

## Component Overview

```mermaid
graph TB
    subgraph "Client Interface Layer"
        HTTP[HTTP/REST Handler]
        GRPC[gRPC Handler]
        WS[WebSocket Handler]
        Console[Console Interface]
    end
    
    subgraph "API Layer"
        Auth[Authentication API]
        Account[Account API]
        Social[Social API]
        Storage[Storage API]
        Match[Match API]
        Tournament[Tournament API]
        Group[Group API]
        Friend[Friend API]
        Leaderboard[Leaderboard API]
        Purchase[Purchase API]
        Runtime[Runtime API]
    end
    
    subgraph "Core Services"
        SessionRegistry[Session Registry]
        MatchRegistry[Match Registry]
        Tracker[Presence Tracker]
        StreamManager[Stream Manager]
        Matchmaker[Matchmaker]
        LeaderboardCache[Leaderboard Cache]
        LoginAttemptCache[Login Attempt Cache]
    end
    
    subgraph "Runtime Engine"
        RuntimeGo[Go Runtime]
        RuntimeLua[Lua Runtime]
        RuntimeJS[JavaScript Runtime]
        RuntimeHooks[Runtime Hooks]
    end
    
    subgraph "Data Layer"
        DBLayer[Database Layer]
        StorageIndex[Storage Index]
        Encryption[Encryption Service]
    end
    
    HTTP --> Auth
    GRPC --> Account
    WS --> Social
    Console --> Storage
    
    Auth --> SessionRegistry
    Social --> Tracker
    Match --> MatchRegistry
    Storage --> StorageIndex
    
    SessionRegistry --> RuntimeHooks
    MatchRegistry --> RuntimeGo
    Tracker --> RuntimeLua
    StreamManager --> RuntimeJS
    
    RuntimeHooks --> DBLayer
    RuntimeGo --> Encryption
    StorageIndex --> DBLayer
```

## Core Components Detail

### 1. Session Management

```mermaid
classDiagram
    class SessionRegistry {
        +sessions map[string]*Session
        +Add(session *Session) error
        +Remove(sessionID string) bool
        +Get(sessionID string) *Session
        +Count() int
        +Range(fn func(*Session) bool)
    }
    
    class Session {
        +ID string
        +UserID uuid.UUID
        +Username string
        +Expiry int64
        +Vars map[string]string
        +Format SessionFormat
        +Context context.Context
        +Logger *zap.Logger
    }
    
    class SessionCache {
        +sessions *sync.Map
        +Add(session *Session)
        +Remove(sessionID string)
        +Get(sessionID string) *Session
    }
    
    SessionRegistry --> Session
    SessionRegistry --> SessionCache
```

### 2. Match System

```mermaid
classDiagram
    class MatchRegistry {
        +matches map[string]*Match
        +CreateMatch(ctx context.Context, id string, core RuntimeMatchCore) (*Match, error)
        +GetMatch(ctx context.Context, id string) *Match
        +RemoveMatch(id string, stream PresenceStream)
        +JoinAttempt(ctx context.Context, id string, node string, userID, sessionID uuid.UUID, username string, sessionExpiry int64, vars map[string]string, clientIP, clientPort, fromNode string) (bool, bool, string, error)
        +Leave(id string, presences []*MatchPresence) error
        +SendData(id string, node string, userID uuid.UUID, sessionID uuid.UUID, username string, node string, opCode int64, data []byte, reliable bool, receiveTime int64) error
    }
    
    class Match {
        +ID string
        +Node string
        +IDComponents []string
        +Core RuntimeMatchCore
        +Handler RuntimeMatchHandler
        +CreateTime int64
        +StartTime int64
        +Tick int64
        +Rate int64
        +Label string
        +TickRate int64
    }
    
    class MatchPresence {
        +UserID uuid.UUID
        +SessionID uuid.UUID
        +Username string
        +Node string
        +Reason PresenceReason
    }
    
    MatchRegistry --> Match
    Match --> MatchPresence
```

### 3. Real-time Communication

```mermaid
classDiagram
    class Tracker {
        +presences map[string]*PresenceMeta
        +Track(ctx context.Context, sessionID uuid.UUID, stream PresenceStream, userID uuid.UUID, meta PresenceMeta) (bool, bool)
        +Untrack(sessionID uuid.UUID, stream PresenceStream, userID uuid.UUID)
        +UntrackAll(sessionID uuid.UUID, reason ...PresenceReason)
        +Update(ctx context.Context, sessionID uuid.UUID, stream PresenceStream, userID uuid.UUID, meta PresenceMeta) (bool, bool)
        +GetByStream(stream PresenceStream) []*Presence
        +CountByStream(stream PresenceStream) int
    }
    
    class PresenceStream {
        +Mode uint8
        +Subject uuid.UUID
        +Subcontext uuid.UUID
        +Label string
    }
    
    class StreamManager {
        +streams map[PresenceStream]struct{}
        +RegisterStream(stream PresenceStream, authoritative bool)
        +UnregisterStream(stream PresenceStream)
        +SendToStream(logger *zap.Logger, stream PresenceStream, envelope *rtapi.Envelope, reliable bool)
    }
    
    Tracker --> PresenceStream
    StreamManager --> PresenceStream
```

### 4. Storage Engine

```mermaid
classDiagram
    class StorageEngine {
        +Read(ctx context.Context, logger *zap.Logger, db *sql.DB, userID uuid.UUID, objectIDs []*api.ReadStorageObjectId) ([]*api.StorageObject, error)
        +Write(ctx context.Context, logger *zap.Logger, db *sql.DB, userID uuid.UUID, objects []*api.WriteStorageObject) ([]*api.StorageObjectAck, error)
        +Delete(ctx context.Context, logger *zap.Logger, db *sql.DB, userID uuid.UUID, objectIDs []*api.DeleteStorageObjectId) error
        +List(ctx context.Context, logger *zap.Logger, db *sql.DB, userID uuid.UUID, collection string, limit int, cursor string) ([]*api.StorageObject, string, error)
    }
    
    class StorageIndex {
        +Write(ctx context.Context, objects []*api.StorageObject) error
        +Delete(ctx context.Context, objects []*api.StorageObjectId) error
        +Search(ctx context.Context, indexName, query string, limit int) ([]*api.StorageObject, error)
    }
    
    class StorageObject {
        +Collection string
        +Key string
        +UserID uuid.UUID
        +Value []byte
        +Version string
        +PermissionRead int32
        +PermissionWrite int32
        +CreateTime int64
        +UpdateTime int64
    }
    
    StorageEngine --> StorageObject
    StorageIndex --> StorageObject
```

### 5. Matchmaker System

```mermaid
stateDiagram-v2
    [*] --> Idle
    Idle --> Queued: Add Ticket
    Queued --> Matched: Match Found
    Queued --> Expired: Timeout
    Matched --> [*]: Match Created
    Expired --> [*]: Ticket Removed
    
    state Queued {
        [*] --> Waiting
        Waiting --> Processing: Process Interval
        Processing --> Waiting: No Match
        Processing --> MatchFound: Match Criteria Met
        MatchFound --> [*]
    }
```

```mermaid
classDiagram
    class Matchmaker {
        +tickets map[string]*MatchmakerTicket
        +index *MatchmakerIndex
        +Add(presences []*MatchmakerPresence, sessionID, partyId uuid.UUID, query string, minCount, maxCount int, stringProperties, numericProperties map[string]float64) (string, error)
        +Remove(sessionID uuid.UUID, ticket string) error
        +RemoveAll(sessionID uuid.UUID) error
        +Process() []*MatchmakerMatch
    }
    
    class MatchmakerTicket {
        +Ticket string
        +PresenceIDs []string
        +SessionID uuid.UUID
        +PartyId uuid.UUID
        +Query string
        +MinCount int
        +MaxCount int
        +CountMultiple int
        +StringProperties map[string]string
        +NumericProperties map[string]float64
        +CreatedAt int64
    }
    
    class MatchmakerMatch {
        +MatchId string
        +Token string
        +Users []*MatchmakerUser
        +Self *MatchmakerUser
    }
    
    Matchmaker --> MatchmakerTicket
    Matchmaker --> MatchmakerMatch
```

### 6. Runtime Engine Architecture

```mermaid
graph TB
    subgraph "Runtime Interface"
        RuntimeManager[Runtime Manager]
        RuntimeProvider[Runtime Provider]
    end
    
    subgraph "Language Runtimes"
        GoRuntime[Go Runtime<br/>Native Performance]
        LuaRuntime[Lua Runtime<br/>Embedded VM]
        JSRuntime[JavaScript Runtime<br/>V8 Engine]
    end
    
    subgraph "Runtime Features"
        Hooks[Event Hooks]
        RPC[Custom RPC]
        MatchCore[Match Logic]
        Auth[Custom Auth]
        Timers[Scheduled Tasks]
    end
    
    subgraph "Runtime Context"
        Database[Database Access]
        HTTP[HTTP Client]
        Logger[Logging]
        Metrics[Metrics]
        Storage[Storage Access]
    end
    
    RuntimeManager --> GoRuntime
    RuntimeManager --> LuaRuntime
    RuntimeManager --> JSRuntime
    
    GoRuntime --> Hooks
    LuaRuntime --> RPC
    JSRuntime --> MatchCore
    
    Hooks --> Database
    RPC --> HTTP
    MatchCore --> Logger
```

## Component Interactions

### Authentication Flow
```mermaid
sequenceDiagram
    participant C as Client
    participant API as API Handler
    participant Auth as Auth Service
    participant SR as Session Registry
    participant DB as Database
    participant RT as Runtime
    
    C->>API: Authentication Request
    API->>Auth: Validate Credentials
    Auth->>DB: Lookup User Account
    DB->>Auth: User Data
    
    alt Runtime Hook Exists
        Auth->>RT: Before Authenticate Hook
        RT->>Auth: Hook Response
    end
    
    Auth->>SR: Create Session
    SR->>Auth: Session Created
    
    alt Runtime Hook Exists
        Auth->>RT: After Authenticate Hook
        RT->>Auth: Hook Response
    end
    
    Auth->>API: Authentication Token
    API->>C: JWT Token Response
```

### Real-time Message Flow
```mermaid
sequenceDiagram
    participant C1 as Client 1
    participant WS as WebSocket Handler
    participant SM as Stream Manager
    participant T as Tracker
    participant C2 as Client 2
    
    C1->>WS: Send Message
    WS->>SM: Route Message
    SM->>T: Get Stream Presences
    T->>SM: Return Recipients
    SM->>WS: Broadcast to Recipients
    WS->>C2: Deliver Message
    C2->>WS: Message Acknowledgment
```

### Storage Operation Flow
```mermaid
sequenceDiagram
    participant C as Client
    participant API as Storage API
    participant RT as Runtime
    participant SE as Storage Engine
    participant SI as Storage Index
    participant DB as Database
    
    C->>API: Storage Write Request
    
    alt Before Hook
        API->>RT: Before Storage Write Hook
        RT->>API: Modified Data
    end
    
    API->>SE: Write Storage Object
    SE->>DB: Database Transaction
    DB->>SE: Write Confirmation
    SE->>SI: Index Update
    SI->>SE: Index Updated
    
    alt After Hook
        SE->>RT: After Storage Write Hook
        RT->>SE: Hook Completed
    end
    
    SE->>API: Write Success
    API->>C: Storage Write Response
```

## Performance Characteristics

### Memory Usage Patterns
```mermaid
pie title Memory Usage Distribution
    "Session Registry" : 25
    "Match Registry" : 30
    "Runtime VMs" : 20
    "Cache Layers" : 15
    "Other Components" : 10
```

### CPU Usage Patterns
```mermaid
pie title CPU Usage Distribution
    "Runtime Execution" : 40
    "Database Operations" : 25
    "Network I/O" : 20
    "Match Processing" : 10
    "Other Operations" : 5
```

## Scaling Considerations

### Horizontal Scaling Components
- **Stateless**: API handlers, Runtime engine (mostly)
- **Session Affinity Required**: WebSocket connections, Match instances
- **Shared State**: Database, Cache layers

### Resource Requirements
- **CPU Intensive**: Runtime execution, Match processing
- **Memory Intensive**: Session registry, Match registry, Runtime VMs
- **I/O Intensive**: Database operations, Network communication

### Bottleneck Analysis
```mermaid
graph LR
    subgraph "Potential Bottlenecks"
        DB[Database<br/>Read/Write Heavy]
        Runtime[Runtime Execution<br/>CPU Heavy]
        Memory[Memory Usage<br/>Session/Match Storage]
        Network[Network I/O<br/>Real-time Traffic]
    end
    
    subgraph "Mitigation Strategies"
        DBCache[Database Caching<br/>Redis/In-Memory]
        RuntimeOpt[Runtime Optimization<br/>Code Efficiency]
        MemoryOpt[Memory Management<br/>Session Cleanup]
        NetworkOpt[Network Optimization<br/>Message Batching]
    end
    
    DB --> DBCache
    Runtime --> RuntimeOpt
    Memory --> MemoryOpt
    Network --> NetworkOpt
```

## Component Health Monitoring

Each component exposes health metrics and status information:

- **Response Times**: Per-operation latency tracking
- **Error Rates**: Component-specific error tracking
- **Resource Usage**: Memory, CPU, and connection metrics
- **Business Metrics**: Active sessions, matches, storage operations

For detailed monitoring setup, see the [Deployment Architecture](deployment.md#monitoring) documentation.