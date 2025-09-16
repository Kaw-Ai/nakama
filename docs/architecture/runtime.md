# Runtime Extensions Architecture

This document details Nakama's runtime extension system, including Go, Lua, and JavaScript runtime environments, custom logic execution, and extensibility patterns.

## Runtime System Overview

```mermaid
graph TB
    subgraph "Runtime Languages"
        GoRuntime[Go Runtime<br/>Native Performance]
        LuaRuntime[Lua Runtime<br/>Embedded VM]
        JSRuntime[JavaScript Runtime<br/>V8 Engine]
    end
    
    subgraph "Runtime Features"
        Hooks[Event Hooks<br/>Lifecycle Events]
        RPC[Custom RPC<br/>API Extensions]
        MatchLogic[Match Logic<br/>Game Server Logic]
        Authentication[Custom Auth<br/>Authentication Providers]
        Schedulers[Scheduled Tasks<br/>Background Jobs]
    end
    
    subgraph "Runtime Context"
        Database[Database Access<br/>SQL Operations]
        HTTPClient[HTTP Client<br/>External APIs]
        Storage[Storage Access<br/>Game Data]
        Logger[Structured Logging<br/>Debug & Monitoring]
        Metrics[Metrics Collection<br/>Performance Tracking]
    end
    
    subgraph "Security & Isolation"
        Sandboxing[Code Sandboxing<br/>Execution Limits]
        Permissions[Permission System<br/>Access Control]
        ResourceLimits[Resource Limits<br/>CPU/Memory/Time]
    end
    
    GoRuntime --> Hooks
    LuaRuntime --> RPC
    JSRuntime --> MatchLogic
    
    Hooks --> Database
    RPC --> HTTPClient
    MatchLogic --> Storage
    Authentication --> Logger
    Schedulers --> Metrics
    
    Database --> Sandboxing
    HTTPClient --> Permissions
    Storage --> ResourceLimits
```

## Go Runtime Architecture

### 1. Go Plugin System

```mermaid
classDiagram
    class RuntimeProvider {
        +NewRuntimeGoContext() RuntimeGoContext
        +NewRuntimeGoLogger() RuntimeGoLogger
        +NewRuntimeGoNakamaModule() RuntimeGoNakamaModule
        +ProcessMatchmakingMatch() interface{}
        +ProcessTournamentEnd() interface{}
        +ProcessTournamentReset() interface{}
        +ProcessLeaderboardEnd() interface{}
        +ProcessLeaderboardReset() interface{}
    }
    
    class RuntimeGoContext {
        +Ctx context.Context
        +Logger *zap.Logger
        +DB *sql.DB
        +UserID uuid.UUID
        +Username string
        +Vars map[string]string
        +UserSessionExp int64
        +SessionID uuid.UUID
        +ClientIP string
        +ClientPort string
        +Lang string
    }
    
    class RuntimeGoNakamaModule {
        +AuthenticateApple() (*api.Session, error)
        +AuthenticateCustom() (*api.Session, error)
        +AuthenticateDevice() (*api.Session, error)
        +AuthenticateEmail() (*api.Session, error)
        +AuthenticateFacebook() (*api.Session, error)
        +AuthenticateGameCenter() (*api.Session, error)
        +AuthenticateGoogle() (*api.Session, error)
        +AuthenticateStream() (*api.Session, error)
        +StorageRead() ([]*api.StorageObject, error)
        +StorageWrite() ([]*api.StorageObjectAck, error)
        +StorageDelete() error
        +StorageList() ([]*api.StorageObjectList, error)
    }
    
    RuntimeProvider --> RuntimeGoContext : "creates"
    RuntimeProvider --> RuntimeGoNakamaModule : "provides"
```

### 2. Go Runtime Hooks

```mermaid
graph TB
    subgraph "Authentication Hooks"
        BeforeAuth[BeforeAuthenticate<br/>Pre-auth validation]
        AfterAuth[AfterAuthenticate<br/>Post-auth processing]
        BeforeLink[BeforeLinkCustom<br/>Account linking]
        AfterLink[AfterLinkCustom<br/>Link confirmation]
    end
    
    subgraph "Storage Hooks"
        BeforeRead[BeforeStorageRead<br/>Access control]
        AfterRead[AfterStorageRead<br/>Data transformation]
        BeforeWrite[BeforeStorageWrite<br/>Validation]
        AfterWrite[AfterStorageWrite<br/>Side effects]
    end
    
    subgraph "Social Hooks"
        BeforeFriend[BeforeAddFriends<br/>Friendship validation]
        AfterFriend[AfterAddFriends<br/>Friendship notifications]
        BeforeGroup[BeforeJoinGroup<br/>Group access control]
        AfterGroup[AfterJoinGroup<br/>Welcome messages]
    end
    
    subgraph "Match Hooks"
        BeforeMatch[BeforeMatchCreate<br/>Match validation]
        AfterMatch[AfterMatchCreate<br/>Match setup]
        MatchJoin[MatchJoinAttempt<br/>Player validation]
        MatchLoop[MatchLoop<br/>Game tick]
    end
    
    BeforeAuth --> BeforeRead
    AfterAuth --> AfterRead
    BeforeLink --> BeforeWrite
    AfterLink --> AfterWrite
    
    BeforeRead --> BeforeFriend
    AfterRead --> AfterFriend
    BeforeWrite --> BeforeGroup
    AfterWrite --> AfterGroup
    
    BeforeFriend --> BeforeMatch
    AfterFriend --> AfterMatch
    BeforeGroup --> MatchJoin
    AfterGroup --> MatchLoop
```

### 3. Go Runtime Execution Flow

```mermaid
sequenceDiagram
    participant Client as Game Client
    participant API as API Handler
    participant Runtime as Go Runtime
    participant Hook as Runtime Hook
    participant DB as Database
    participant Ext as External Service
    
    Client->>API: API Request
    API->>Runtime: Execute Before Hook
    Runtime->>Hook: Call Hook Function
    Hook->>DB: Database Query
    DB->>Hook: Query Result
    Hook->>Ext: External API Call
    Ext->>Hook: API Response
    Hook->>Runtime: Hook Result
    Runtime->>API: Continue Processing
    API->>DB: Main Operation
    DB->>API: Operation Result
    API->>Runtime: Execute After Hook
    Runtime->>Hook: Call After Hook
    Hook->>Runtime: Hook Complete
    Runtime->>API: Final Result
    API->>Client: API Response
```

## Lua Runtime Architecture

### 1. Lua Virtual Machine

```mermaid
classDiagram
    class LuaRuntime {
        +vm *lua.LState
        +modules map[string]lua.LGFunction
        +InitModule() error
        +LoadModule() error
        +Call() (lua.LValue, error)
        +ExecuteHook() (interface{}, error)
        +InvokeFunction() (interface{}, error)
    }
    
    class LuaState {
        +*lua.LState
        +userdata map[string]interface{}
        +Get(key string) lua.LValue
        +Set(key string, value lua.LValue)
        +CallByParam(params lua.P) error
        +GetGlobal(name string) lua.LValue
        +SetGlobal(name string, value lua.LValue)
    }
    
    class NakamaModule {
        +authenticate_apple() userdata
        +authenticate_custom() userdata
        +authenticate_device() userdata
        +storage_read() table
        +storage_write() table
        +storage_delete() nil
        +http_request() table
        +json_encode() string
        +json_decode() table
        +logger_info() nil
        +logger_error() nil
    }
    
    LuaRuntime --> LuaState : "manages"
    LuaRuntime --> NakamaModule : "provides"
```

### 2. Lua Module System

```mermaid
graph TB
    subgraph "Core Modules"
        NakamaCore[nakama<br/>Core API Functions]
        JSON[json<br/>JSON Encoding/Decoding]
        HTTP[http<br/>HTTP Client]
        Base64[base64<br/>Encoding Utilities]
        UUID[uuid<br/>UUID Generation]
    end
    
    subgraph "Game Modules"
        Match[match<br/>Match Logic]
        Tournament[tournament<br/>Tournament Logic]
        Leaderboard[leaderboard<br/>Scoring Logic]
        Notification[notification<br/>Push Notifications]
    end
    
    subgraph "Utility Modules"
        Crypto[crypto<br/>Cryptographic Functions]
        Time[time<br/>Time Utilities]
        Math[math<br/>Mathematical Functions]
        String[string<br/>String Manipulation]
    end
    
    subgraph "Custom Modules"
        UserModule[User Defined<br/>Custom Logic]
        GameLogic[Game Logic<br/>Business Rules]
        Integration[Integration<br/>External APIs]
    end
    
    NakamaCore --> Match
    JSON --> Tournament
    HTTP --> Leaderboard
    Base64 --> Notification
    
    Match --> Crypto
    Tournament --> Time
    Leaderboard --> Math
    Notification --> String
    
    Crypto --> UserModule
    Time --> GameLogic
    Math --> Integration
```

### 3. Lua Execution Context

```lua
-- Example Lua hook implementation
local nk = require("nakama")

function authenticate_custom(context, payload)
    -- Access context information
    local user_id = context.user_id
    local username = context.username
    local vars = context.vars or {}
    
    -- Custom authentication logic
    local custom_id = payload.account.id
    if not custom_id or #custom_id < 6 then
        error("Custom ID must be at least 6 characters")
    end
    
    -- External validation
    local http_headers = {
        ["Content-Type"] = "application/json",
        ["Authorization"] = "Bearer " .. vars.api_token
    }
    
    local http_body = nk.json_encode({
        custom_id = custom_id,
        timestamp = os.time()
    })
    
    local result = nk.http_request("https://auth.example.com/validate", 
                                   "POST", http_headers, http_body)
    
    if result.code ~= 200 then
        error("External authentication failed")
    end
    
    -- Return success with custom properties
    return {
        success = true,
        user_id = user_id,
        properties = {
            external_validated = true,
            validation_time = os.time()
        }
    }
end
```

## JavaScript Runtime Architecture

### 1. V8 JavaScript Engine

```mermaid
classDiagram
    class JSRuntime {
        +vm *goja.Runtime
        +modules map[string]interface{}
        +InitializeVM() error
        +LoadScript() error
        +ExecuteFunction() (interface{}, error)
        +SetGlobal() error
        +GetGlobal() interface{}
    }
    
    class JSContext {
        +ctx context.Context
        +logger Logger
        +db Database
        +nk NakamaModule
        +userID string
        +username string
        +vars map[string]interface{}
        +sessionExp number
        +sessionID string
        +clientIP string
        +clientPort string
    }
    
    class NakamaJS {
        +authenticateApple() Promise
        +authenticateCustom() Promise
        +authenticateDevice() Promise
        +storageRead() Promise
        +storageWrite() Promise
        +storageDelete() Promise
        +httpRequest() Promise
        +logger Object
        +sqlExec() Promise
        +sqlQuery() Promise
    }
    
    JSRuntime --> JSContext : "provides"
    JSRuntime --> NakamaJS : "exposes"
```

### 2. JavaScript Module Loading

```mermaid
sequenceDiagram
    participant Server as Nakama Server
    participant JS as JS Runtime
    participant V8 as V8 Engine
    participant Module as JS Module
    participant API as Nakama API
    
    Server->>JS: Initialize Runtime
    JS->>V8: Create VM Instance
    V8->>JS: VM Ready
    JS->>Module: Load JavaScript Files
    Module->>V8: Parse & Compile
    V8->>Module: Compiled Code
    Module->>API: Register Functions
    API->>Module: API Bindings
    Module->>JS: Module Loaded
    JS->>Server: Runtime Ready
```

### 3. JavaScript Hook Example

```javascript
// Example JavaScript hook implementation
function beforeStorageWrite(ctx, logger, nk, payload) {
    // Access context
    const userId = ctx.userId;
    const username = ctx.username;
    const vars = ctx.vars || {};
    
    // Validate storage objects
    payload.objects.forEach(obj => {
        // Validate collection permissions
        if (obj.collection === 'private' && obj.userId !== userId) {
            throw new Error('Cannot write to private collection of another user');
        }
        
        // Validate data size
        if (obj.value.length > 1024 * 1024) { // 1MB limit
            throw new Error('Storage object too large');
        }
        
        // Parse and validate JSON content
        try {
            const data = JSON.parse(obj.value);
            if (!data.version || data.version < 1) {
                throw new Error('Invalid data version');
            }
        } catch (e) {
            throw new Error('Invalid JSON in storage object');
        }
    });
    
    // Log the operation
    logger.info('Storage write validated', {
        userId: userId,
        objectCount: payload.objects.length
    });
    
    return payload;
}

// Example custom RPC function
function customMatchLogic(ctx, logger, nk, payload) {
    const userId = ctx.userId;
    const matchId = payload.match_id;
    
    // Get match state
    const match = nk.matchGet(matchId);
    if (!match) {
        throw new Error('Match not found');
    }
    
    // Custom business logic
    const playerAction = JSON.parse(payload.action);
    const gameState = JSON.parse(match.state);
    
    // Validate player action
    if (!isValidAction(playerAction, gameState, userId)) {
        throw new Error('Invalid action for current game state');
    }
    
    // Update game state
    const newState = processAction(gameState, playerAction, userId);
    
    // Send to match
    nk.matchSignal(matchId, JSON.stringify({
        type: 'state_update',
        state: newState,
        player: userId
    }));
    
    return {
        success: true,
        newState: newState
    };
}

function isValidAction(action, state, userId) {
    // Implement game-specific validation logic
    return true;
}

function processAction(state, action, userId) {
    // Implement game-specific state update logic
    return state;
}
```

## Runtime Event System

### 1. Event Lifecycle

```mermaid
stateDiagram-v2
    [*] --> EventTriggered
    EventTriggered --> HookLookup: Find Registered Hook
    HookLookup --> BeforeHook: Execute Before Hook
    BeforeHook --> Validation: Validate Hook Result
    Validation --> CoreLogic: Execute Core Logic
    CoreLogic --> AfterHook: Execute After Hook
    AfterHook --> Completion: Hook Complete
    Completion --> [*]
    
    Validation --> Error: Invalid Result
    BeforeHook --> Error: Hook Exception
    CoreLogic --> Error: Core Exception
    AfterHook --> Error: Hook Exception
    Error --> [*]
    
    state BeforeHook {
        [*] --> ContextSetup
        ContextSetup --> RuntimeExecution
        RuntimeExecution --> ResultValidation
        ResultValidation --> [*]
    }
    
    state AfterHook {
        [*] --> ContextSetup
        ContextSetup --> RuntimeExecution
        RuntimeExecution --> SideEffects
        SideEffects --> [*]
    }
```

### 2. Hook Registration System

```mermaid
classDiagram
    class RuntimeInfo {
        +string name
        +RuntimeType type
        +map[string]RuntimeHook hooks
        +map[string]RuntimeRPC rpcs
        +map[string]RuntimeMatch matches
        +*RuntimeConfig config
    }
    
    class RuntimeHook {
        +string name
        +HookType type
        +interface{} handler
        +bool enabled
        +map[string]interface{} config
    }
    
    class RuntimeRPC {
        +string id
        +interface{} handler
        +HTTPMethod method
        +string path
        +bool authenticated
    }
    
    class RuntimeMatch {
        +string name
        +interface{} core
        +map[string]interface{} config
        +bool authoritative
    }
    
    class HookType {
        +BEFORE_AUTHENTICATE = "before_authenticate"
        +AFTER_AUTHENTICATE = "after_authenticate"
        +BEFORE_STORAGE_READ = "before_storage_read"
        +AFTER_STORAGE_WRITE = "after_storage_write"
        +MATCH_INIT = "match_init"
        +MATCH_JOIN_ATTEMPT = "match_join_attempt"
        +RPC = "rpc"
    }
    
    RuntimeInfo --> RuntimeHook : "contains"
    RuntimeInfo --> RuntimeRPC : "contains"
    RuntimeInfo --> RuntimeMatch : "contains"
    RuntimeHook --> HookType : "categorized_by"
```

## Custom RPC System

### 1. RPC Registration and Routing

```mermaid
graph TB
    subgraph "RPC Registration"
        GoRPC[Go RPC<br/>Native Functions]
        LuaRPC[Lua RPC<br/>Lua Functions]
        JSRPC[JavaScript RPC<br/>JS Functions]
    end
    
    subgraph "RPC Router"
        HTTPEndpoint[HTTP Endpoint<br/>/v2/rpc/function_name]
        gRPCEndpoint[gRPC Endpoint<br/>RPC Service Call]
        WSEndpoint[WebSocket Endpoint<br/>Real-time RPC]
    end
    
    subgraph "RPC Execution"
        Authentication[Authentication Check]
        RateLimiting[Rate Limiting]
        ParameterValidation[Parameter Validation]
        FunctionExecution[Function Execution]
        ResponseFormatting[Response Formatting]
    end
    
    GoRPC --> HTTPEndpoint
    LuaRPC --> gRPCEndpoint
    JSRPC --> WSEndpoint
    
    HTTPEndpoint --> Authentication
    gRPCEndpoint --> RateLimiting
    WSEndpoint --> ParameterValidation
    
    Authentication --> FunctionExecution
    RateLimiting --> FunctionExecution
    ParameterValidation --> FunctionExecution
    FunctionExecution --> ResponseFormatting
```

### 2. RPC Security Model

```mermaid
classDiagram
    class RPCConfig {
        +bool authenticated
        +[]string allowedUsers
        +[]string allowedRoles
        +map[string]interface{} rateLimit
        +map[string]interface{} permissions
        +bool httpEnabled
        +bool grpcEnabled
        +bool wsEnabled
    }
    
    class RPCContext {
        +string userID
        +string sessionID
        +string username
        +map[string]string vars
        +string clientIP
        +string clientPort
        +string userAgent
        +map[string]interface{} permissions
    }
    
    class RPCValidator {
        +ValidateAuth(ctx RPCContext, config RPCConfig) error
        +ValidateRateLimit(ctx RPCContext, config RPCConfig) error
        +ValidatePermissions(ctx RPCContext, config RPCConfig) error
        +ValidateInput(payload interface{}) error
    }
    
    RPCConfig --> RPCContext : "applied_to"
    RPCValidator --> RPCConfig : "validates"
    RPCValidator --> RPCContext : "checks"
```

## Match Runtime System

### 1. Authoritative Server Architecture

```mermaid
graph TB
    subgraph "Match Creation"
        MatchRequest[Match Create Request]
        MatchValidation[Validate Parameters]
        RuntimeInit[Initialize Runtime]
        MatchRegistry[Register Match]
    end
    
    subgraph "Match Runtime"
        MatchState[Game State Management]
        TickLoop[Game Tick Loop]
        PlayerActions[Player Action Processing]
        StateSync[State Synchronization]
    end
    
    subgraph "Match Communication"
        InputHandler[Input Handler]
        OutputBroadcast[Output Broadcast]
        EventDispatcher[Event Dispatcher]
        MessageQueue[Message Queue]
    end
    
    MatchRequest --> MatchValidation
    MatchValidation --> RuntimeInit
    RuntimeInit --> MatchRegistry
    MatchRegistry --> MatchState
    
    MatchState --> TickLoop
    TickLoop --> PlayerActions
    PlayerActions --> StateSync
    StateSync --> InputHandler
    
    InputHandler --> OutputBroadcast
    OutputBroadcast --> EventDispatcher
    EventDispatcher --> MessageQueue
```

### 2. Match Lifecycle Hooks

```mermaid
sequenceDiagram
    participant MM as Matchmaker
    participant MR as Match Registry
    participant RT as Runtime
    participant P1 as Player 1
    participant P2 as Player 2
    
    MM->>MR: Create Match
    MR->>RT: Match Init Hook
    RT->>MR: Initial State
    
    P1->>MR: Join Request
    MR->>RT: Join Attempt Hook
    RT->>MR: Allow/Deny Join
    MR->>RT: Join Hook (if allowed)
    RT->>P1: Welcome Message
    
    P2->>MR: Join Request
    MR->>RT: Join Attempt Hook
    RT->>MR: Allow/Deny Join
    MR->>RT: Join Hook (if allowed)
    RT->>P2: Welcome Message
    
    loop Game Loop
        RT->>RT: Match Loop Hook
        RT->>MR: Broadcast State
        MR->>P1: Game State
        MR->>P2: Game State
        P1->>MR: Player Action
        MR->>RT: Process Action
        P2->>MR: Player Action
        MR->>RT: Process Action
    end
    
    RT->>MR: Match End
    MR->>RT: Terminate Hook
    RT->>MR: Final State
```

## Performance and Resource Management

### 1. Resource Limits

```mermaid
classDiagram
    class RuntimeLimits {
        +time.Duration maxExecutionTime
        +int64 maxMemoryUsage
        +int maxGoroutines
        +int maxHTTPRequests
        +int maxDatabaseQueries
        +int maxFileOperations
        +time.Duration httpTimeout
        +int64 maxPayloadSize
    }
    
    class ResourceMonitor {
        +startTime time.Time
        +memoryUsed int64
        +goroutineCount int
        +httpRequestCount int
        +dbQueryCount int
        +fileOpCount int
        +CheckLimits() error
        +ResetCounters()
    }
    
    class RuntimePool {
        +pool sync.Pool
        +maxInstances int
        +currentInstances int
        +Get() Runtime
        +Put(runtime Runtime)
        +Close() error
    }
    
    RuntimeLimits --> ResourceMonitor : "enforced_by"
    ResourceMonitor --> RuntimePool : "monitored_by"
```

### 2. Performance Optimization

```mermaid
graph TB
    subgraph "Execution Optimization"
        CodeCaching[Code Caching<br/>Compiled Bytecode]
        PoolingVMs[VM Pooling<br/>Reuse Instances]
        JITCompilation[JIT Compilation<br/>Native Code Gen]
        Profiling[Performance Profiling<br/>Bottleneck Detection]
    end
    
    subgraph "Memory Management"
        GCTuning[GC Tuning<br/>Garbage Collection]
        MemoryPooling[Memory Pooling<br/>Object Reuse]
        MemoryLimits[Memory Limits<br/>Per-Runtime Caps]
        LeakDetection[Leak Detection<br/>Memory Monitoring]
    end
    
    subgraph "I/O Optimization"
        ConnectionPooling[Connection Pooling<br/>Database Connections]
        RequestBatching[Request Batching<br/>Bulk Operations]
        Caching[Result Caching<br/>Expensive Operations]
        AsyncIO[Async I/O<br/>Non-blocking Operations]
    end
    
    CodeCaching --> GCTuning
    PoolingVMs --> MemoryPooling
    JITCompilation --> MemoryLimits
    Profiling --> LeakDetection
    
    GCTuning --> ConnectionPooling
    MemoryPooling --> RequestBatching
    MemoryLimits --> Caching
    LeakDetection --> AsyncIO
```

## Development and Debugging

### 1. Runtime Development Workflow

```mermaid
graph LR
    subgraph "Development"
        WriteCode[Write Runtime Code<br/>Go/Lua/JS]
        LocalTest[Local Testing<br/>Development Server]
        UnitTest[Unit Testing<br/>Function Tests]
        Integration[Integration Testing<br/>End-to-end Tests]
    end
    
    subgraph "Deployment"
        Package[Package Code<br/>Bundle Files]
        Upload[Upload to Server<br/>Hot Reload]
        Validate[Validate Deployment<br/>Health Checks]
        Monitor[Monitor Performance<br/>Runtime Metrics]
    end
    
    subgraph "Debugging"
        Logging[Runtime Logging<br/>Debug Output]
        ErrorTracking[Error Tracking<br/>Exception Handling]
        Profiling[Performance Profiling<br/>CPU/Memory Usage]
        Tracing[Request Tracing<br/>Execution Flow]
    end
    
    WriteCode --> LocalTest
    LocalTest --> UnitTest
    UnitTest --> Integration
    Integration --> Package
    
    Package --> Upload
    Upload --> Validate
    Validate --> Monitor
    Monitor --> Logging
    
    Logging --> ErrorTracking
    ErrorTracking --> Profiling
    Profiling --> Tracing
```

### 2. Error Handling and Recovery

```mermaid
graph TB
    subgraph "Error Types"
        SyntaxError[Syntax Errors<br/>Code Parsing]
        RuntimeError[Runtime Errors<br/>Execution Exceptions]
        ResourceError[Resource Errors<br/>Limit Exceeded]
        TimeoutError[Timeout Errors<br/>Execution Timeout]
    end
    
    subgraph "Error Handling"
        ErrorCapture[Error Capture<br/>Exception Handling]
        ErrorLogging[Error Logging<br/>Structured Logs]
        ErrorRecovery[Error Recovery<br/>Graceful Degradation]
        ErrorNotification[Error Notification<br/>Alert System]
    end
    
    subgraph "Recovery Strategies"
        Retry[Retry Logic<br/>Transient Failures]
        Fallback[Fallback Logic<br/>Default Behavior]
        CircuitBreaker[Circuit Breaker<br/>Failure Detection]
        Isolation[Error Isolation<br/>Prevent Cascade]
    end
    
    SyntaxError --> ErrorCapture
    RuntimeError --> ErrorLogging
    ResourceError --> ErrorRecovery
    TimeoutError --> ErrorNotification
    
    ErrorCapture --> Retry
    ErrorLogging --> Fallback
    ErrorRecovery --> CircuitBreaker
    ErrorNotification --> Isolation
```

For more information on related topics:
- [Component Architecture](components.md) - Runtime component integration
- [Authentication & Authorization](auth.md) - Runtime security context
- [API Architecture](api.md) - Custom RPC endpoint design