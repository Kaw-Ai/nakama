# API Architecture

This document details Nakama's API design, protocol implementations, endpoint structures, and client communication patterns.

## API Overview

```mermaid
graph TB
    subgraph "Client Protocols"
        REST[REST API<br/>HTTP/JSON]
        gRPC[gRPC API<br/>Protocol Buffers]
        WebSocket[WebSocket API<br/>Real-time]
        UDP[rUDP API<br/>Low Latency]
    end
    
    subgraph "API Layers"
        Gateway[API Gateway<br/>Request Routing]
        Authentication[Authentication Layer<br/>Token Validation]
        Authorization[Authorization Layer<br/>Permission Checking]
        RateLimit[Rate Limiting<br/>Abuse Prevention]
        Validation[Input Validation<br/>Data Sanitization]
    end
    
    subgraph "Service Layer"
        UserService[User Service<br/>Account Management]
        SocialService[Social Service<br/>Friends & Groups]
        StorageService[Storage Service<br/>Data Persistence]
        MatchService[Match Service<br/>Game Instances]
        RealtimeService[Realtime Service<br/>Live Communication]
    end
    
    subgraph "Data Layer"
        Database[Database<br/>Persistent Storage]
        Cache[Cache<br/>Session & Temporary Data]
        Files[File Storage<br/>Assets & Uploads]
    end
    
    REST --> Gateway
    gRPC --> Gateway
    WebSocket --> Authentication
    UDP --> Authentication
    
    Gateway --> Authorization
    Authentication --> RateLimit
    Authorization --> Validation
    RateLimit --> UserService
    
    UserService --> Database
    SocialService --> Cache
    StorageService --> Files
    MatchService --> Database
    RealtimeService --> Cache
```

## Protocol Architecture

### 1. REST API Design

```mermaid
classDiagram
    class RESTEndpoint {
        +string method
        +string path
        +map[string]string headers
        +interface{} requestBody
        +interface{} responseBody
        +int statusCode
        +map[string]string queryParams
    }
    
    class RESTHandler {
        +HandleAuthenticate(w http.ResponseWriter, r *http.Request)
        +HandleStorageRead(w http.ResponseWriter, r *http.Request)
        +HandleStorageWrite(w http.ResponseWriter, r *http.Request)
        +HandleFriendAdd(w http.ResponseWriter, r *http.Request)
        +HandleGroupJoin(w http.ResponseWriter, r *http.Request)
        +HandleLeaderboardList(w http.ResponseWriter, r *http.Request)
        +HandleRPC(w http.ResponseWriter, r *http.Request)
    }
    
    class RESTMiddleware {
        +AuthenticationMiddleware(next http.Handler) http.Handler
        +AuthorizationMiddleware(next http.Handler) http.Handler
        +RateLimitMiddleware(next http.Handler) http.Handler
        +LoggingMiddleware(next http.Handler) http.Handler
        +CORSMiddleware(next http.Handler) http.Handler
    }
    
    RESTEndpoint --> RESTHandler : "handled_by"
    RESTHandler --> RESTMiddleware : "protected_by"
```

### 2. gRPC Service Definition

```protobuf
// Nakama API Protocol Buffer definitions
syntax = "proto3";

package nakama.api;

import "google/api/annotations.proto";
import "google/protobuf/empty.proto";
import "google/protobuf/timestamp.proto";
import "google/protobuf/wrappers.proto";
import "protoc-gen-openapiv2/options/annotations.proto";

// The Nakama RPC protocol service built with GRPC.
service Nakama {
  // Authenticate a user with a custom ID.
  rpc AuthenticateCustom (AuthenticateCustomRequest) returns (Session) {
    option (google.api.http) = {
      post: "/v2/account/authenticate/custom"
      body: "*"
    };
  }

  // Authenticate a user with a device ID.
  rpc AuthenticateDevice (AuthenticateDeviceRequest) returns (Session) {
    option (google.api.http) = {
      post: "/v2/account/authenticate/device"
      body: "*"
    };
  }

  // Read storage objects.
  rpc ReadStorageObjects (ReadStorageObjectsRequest) returns (StorageObjects) {
    option (google.api.http) = {
      post: "/v2/storage"
      body: "*"
    };
  }

  // Write storage objects.
  rpc WriteStorageObjects (WriteStorageObjectsRequest) returns (StorageObjectAcks) {
    option (google.api.http) = {
      put: "/v2/storage"
      body: "*"
    };
  }
}

// A user session.
message Session {
  // True if the corresponding account was just created, false otherwise.
  bool created = 1;
  // Authentication credentials.
  string token = 2;
  // Refresh token that can be used for session token renewal.
  string refresh_token = 3;
}

// Authenticate against the server with a custom ID.
message AuthenticateCustomRequest {
  // The custom account details.
  AccountCustom account = 1;
  // Register the account if the account does not exist.
  google.protobuf.BoolValue create = 2;
  // Set the username on the account at register. Must be unique.
  string username = 3;
}
```

### 3. WebSocket Protocol

```mermaid
sequenceDiagram
    participant C as Client
    participant WS as WebSocket Handler
    participant Auth as Auth Service
    participant Service as Service Layer
    
    C->>WS: WebSocket Handshake
    WS->>C: Connection Established
    
    C->>WS: Authentication Message
    WS->>Auth: Validate Token
    Auth->>WS: Authentication Result
    WS->>C: Authentication Response
    
    loop Real-time Communication
        C->>WS: Message Envelope
        WS->>Service: Route Message
        Service->>WS: Response/Broadcast
        WS->>C: Response Envelope
    end
    
    C->>WS: Close Connection
    WS->>C: Connection Closed
```

## Authentication API

### 1. Authentication Endpoints

```mermaid
graph TB
    subgraph "Social Authentication"
        AppleAuth[POST /v2/account/authenticate/apple<br/>Sign in with Apple]
        GoogleAuth[POST /v2/account/authenticate/google<br/>Google OAuth]
        FacebookAuth[POST /v2/account/authenticate/facebook<br/>Facebook Login]
        SteamAuth[POST /v2/account/authenticate/steam<br/>Steam OpenID]
        GameCenterAuth[POST /v2/account/authenticate/gamecenter<br/>Apple Game Center]
    end
    
    subgraph "Custom Authentication"
        EmailAuth[POST /v2/account/authenticate/email<br/>Email/Password]
        DeviceAuth[POST /v2/account/authenticate/device<br/>Device ID]
        CustomAuth[POST /v2/account/authenticate/custom<br/>Custom ID]
    end
    
    subgraph "Session Management"
        SessionLogout[POST /v2/session/logout<br/>End Session]
        SessionRefresh[POST /v2/session/refresh<br/>Refresh Token]
        SessionValidate[GET /v2/session<br/>Validate Session]
    end
    
    subgraph "Account Linking"
        LinkApple[POST /v2/account/link/apple<br/>Link Apple ID]
        LinkGoogle[POST /v2/account/link/google<br/>Link Google Account]
        LinkCustom[POST /v2/account/link/custom<br/>Link Custom ID]
        UnlinkApple[POST /v2/account/unlink/apple<br/>Unlink Apple ID]
    end
    
    AppleAuth --> SessionLogout
    GoogleAuth --> SessionRefresh
    FacebookAuth --> SessionValidate
    
    EmailAuth --> LinkApple
    DeviceAuth --> LinkGoogle
    CustomAuth --> LinkCustom
    
    LinkApple --> UnlinkApple
```

### 2. Authentication Flow

```mermaid
sequenceDiagram
    participant Client as Game Client
    participant API as Nakama API
    participant Provider as OAuth Provider
    participant DB as Database
    participant Session as Session Registry
    
    Client->>Provider: OAuth Request
    Provider->>Client: OAuth Token
    Client->>API: Authenticate with Token
    
    API->>Provider: Verify Token
    Provider->>API: User Profile
    
    API->>DB: Check/Create User
    DB->>API: User Account
    
    API->>Session: Create Session
    Session->>API: Session Token
    
    API->>Client: JWT Token + Refresh Token
    
    Note over Client,API: Subsequent API calls use JWT token
    
    Client->>API: API Request with JWT
    API->>API: Validate JWT
    API->>Client: API Response
```

## User Management API

### 1. Account Operations

```mermaid
graph LR
    subgraph "Account Management"
        GetAccount[GET /v2/account<br/>Get Account Info]
        UpdateAccount[PUT /v2/account<br/>Update Account]
        DeleteAccount[DELETE /v2/account<br/>Delete Account]
    end
    
    subgraph "User Operations"
        GetUsers[GET /v2/user<br/>Get Users by ID]
        BanUser[POST /v2/console/user/ban<br/>Ban User Account]
        UnbanUser[POST /v2/console/user/unban<br/>Unban User Account]
    end
    
    subgraph "Device Management"
        ListDevices[GET /v2/account/devices<br/>List User Devices]
        LinkDevice[POST /v2/account/link/device<br/>Link New Device]
        UnlinkDevice[POST /v2/account/unlink/device<br/>Unlink Device]
    end
    
    GetAccount --> GetUsers
    UpdateAccount --> BanUser
    DeleteAccount --> UnbanUser
    
    GetUsers --> ListDevices
    BanUser --> LinkDevice
    UnbanUser --> UnlinkDevice
```

### 2. User Profile Schema

```json
{
  "user": {
    "id": "user-uuid",
    "username": "unique-username",
    "display_name": "Display Name",
    "avatar_url": "https://example.com/avatar.jpg",
    "lang_tag": "en",
    "location": "San Francisco, CA",
    "timezone": "America/Los_Angeles",
    "metadata": {
      "level": 25,
      "class": "warrior",
      "guild": "dragons"
    },
    "facebook_id": "facebook-user-id",
    "google_id": "google-user-id",
    "gamecenter_id": "gamecenter-player-id",
    "steam_id": "steam-user-id",
    "custom_id": "custom-user-id",
    "edge_count": 15,
    "create_time": "2023-01-15T10:30:00Z",
    "update_time": "2023-12-01T14:20:00Z",
    "online": true
  }
}
```

## Storage API

### 1. Storage Operations

```mermaid
classDiagram
    class StorageAPI {
        +ReadStorageObjects(request *api.ReadStorageObjectsRequest) (*api.StorageObjects, error)
        +WriteStorageObjects(request *api.WriteStorageObjectsRequest) (*api.StorageObjectAcks, error)
        +DeleteStorageObjects(request *api.DeleteStorageObjectsRequest) error
        +ListStorageObjects(request *api.ListStorageObjectsRequest) (*api.StorageObjectList, error)
    }
    
    class StorageObject {
        +string collection
        +string key
        +string user_id
        +string value
        +string version
        +int32 permission_read
        +int32 permission_write
        +string create_time
        +string update_time
    }
    
    class StoragePermissions {
        +NO_READ = 0
        +OWNER_READ = 1
        +PUBLIC_READ = 2
        +NO_WRITE = 0
        +OWNER_WRITE = 1
    }
    
    StorageAPI --> StorageObject : "manages"
    StorageObject --> StoragePermissions : "uses"
```

### 2. Storage Access Patterns

```mermaid
graph TB
    subgraph "Read Patterns"
        SingleRead[Single Object Read<br/>Specific Key]
        BatchRead[Batch Read<br/>Multiple Keys]
        ListRead[List Read<br/>Collection Browse]
        SearchRead[Search Read<br/>Indexed Query]
    end
    
    subgraph "Write Patterns"
        CreateWrite[Create Write<br/>New Object]
        UpdateWrite[Update Write<br/>Version Check]
        UpsertWrite[Upsert Write<br/>Create or Update]
        ConditionalWrite[Conditional Write<br/>If-Match Header]
    end
    
    subgraph "Permission Levels"
        Private[Private<br/>Owner Only]
        PublicRead[Public Read<br/>Owner Write]
        FullPublic[Full Public<br/>Read/Write All]
        SystemOnly[System Only<br/>Runtime Access]
    end
    
    SingleRead --> Private
    BatchRead --> PublicRead
    ListRead --> FullPublic
    SearchRead --> SystemOnly
    
    CreateWrite --> Private
    UpdateWrite --> PublicRead
    UpsertWrite --> FullPublic
    ConditionalWrite --> SystemOnly
```

## Social API

### 1. Friends System

```mermaid
graph TB
    subgraph "Friend Operations"
        AddFriend[POST /v2/friend<br/>Send Friend Request]
        ListFriends[GET /v2/friend<br/>List Friends]
        DeleteFriend[DELETE /v2/friend<br/>Remove Friend]
        BlockUser[POST /v2/friend/block<br/>Block User]
        UnblockUser[DELETE /v2/friend/block<br/>Unblock User]
    end
    
    subgraph "Friend States"
        Pending[Pending Request<br/>State: 1]
        Accepted[Accepted Friend<br/>State: 0]
        Blocked[Blocked User<br/>State: 3]
        Mutual[Mutual Friends<br/>Bidirectional]
    end
    
    subgraph "Friend Discovery"
        ImportContacts[POST /v2/friend/import<br/>Import Contacts]
        SearchUsers[GET /v2/user<br/>Search by Username]
        FindByCode[GET /v2/user/code<br/>Friend Code Lookup]
    end
    
    AddFriend --> Pending
    ListFriends --> Accepted
    DeleteFriend --> Blocked
    BlockUser --> Mutual
    
    ImportContacts --> SearchUsers
    SearchUsers --> FindByCode
```

### 2. Groups System

```mermaid
classDiagram
    class GroupAPI {
        +CreateGroup(request *api.CreateGroupRequest) (*api.Group, error)
        +JoinGroup(request *api.JoinGroupRequest) error
        +LeaveGroup(request *api.LeaveGroupRequest) error
        +AddGroupUsers(request *api.AddGroupUsersRequest) error
        +BanGroupUsers(request *api.BanGroupUsersRequest) error
        +KickGroupUsers(request *api.KickGroupUsersRequest) error
        +PromoteGroupUsers(request *api.PromoteGroupUsersRequest) error
        +DemoteGroupUsers(request *api.DemoteGroupUsersRequest) error
        +ListGroupUsers(request *api.ListGroupUsersRequest) (*api.GroupUserList, error)
        +UpdateGroup(request *api.UpdateGroupRequest) error
        +DeleteGroup(request *api.DeleteGroupRequest) error
    }
    
    class Group {
        +string id
        +string creator_id
        +string name
        +string description
        +string lang_tag
        +string avatar_url
        +bool open
        +int32 edge_count
        +int32 max_count
        +map metadata
        +string create_time
        +string update_time
    }
    
    class GroupMember {
        +User user
        +int32 state
        +string create_time
        +string update_time
    }
    
    GroupAPI --> Group : "manages"
    Group --> GroupMember : "contains"
```

## Real-time API

### 1. WebSocket Message Types

```mermaid
graph TB
    subgraph "Connection Messages"
        Ping[Ping<br/>Keep-alive]
        Pong[Pong<br/>Keep-alive Response]
        Error[Error<br/>Error Response]
        SessionStart[Session Start<br/>Authentication]
    end
    
    subgraph "Channel Messages"
        ChannelJoin[Channel Join<br/>Subscribe to Channel]
        ChannelLeave[Channel Leave<br/>Unsubscribe]
        ChannelMessage[Channel Message<br/>Send Message]
        ChannelMessageAck[Channel Message Ack<br/>Delivery Confirmation]
        ChannelPresence[Channel Presence<br/>User Presence Update]
    end
    
    subgraph "Match Messages"
        MatchCreate[Match Create<br/>Create New Match]
        MatchJoin[Match Join<br/>Join Existing Match]
        MatchLeave[Match Leave<br/>Leave Match]
        MatchData[Match Data<br/>Game State Update]
        MatchPresence[Match Presence<br/>Player Presence]
    end
    
    subgraph "Status Messages"
        StatusFollow[Status Follow<br/>Follow User Status]
        StatusUnfollow[Status Unfollow<br/>Unfollow User]
        StatusUpdate[Status Update<br/>Update Own Status]
        StatusPresence[Status Presence<br/>Status Change Event]
    end
    
    Ping --> ChannelJoin
    SessionStart --> ChannelMessage
    ChannelMessage --> MatchCreate
    ChannelPresence --> MatchData
    
    MatchData --> StatusFollow
    MatchPresence --> StatusUpdate
```

### 2. Real-time Message Flow

```mermaid
sequenceDiagram
    participant C1 as Client 1
    participant WS as WebSocket Server
    participant SM as Stream Manager
    participant C2 as Client 2
    participant C3 as Client 3
    
    C1->>WS: Join Channel Message
    WS->>SM: Register Client on Stream
    SM->>WS: Subscription Confirmed
    WS->>C1: Channel Join Response
    
    C2->>WS: Join Same Channel
    WS->>SM: Register Client on Stream
    SM->>WS: Subscription Confirmed
    WS->>C2: Channel Join Response
    
    C1->>WS: Send Channel Message
    WS->>SM: Broadcast to Stream
    SM->>WS: Deliver to Subscribers
    WS->>C2: Channel Message
    WS->>C3: Channel Message
    
    C2->>WS: Leave Channel
    WS->>SM: Unregister Client
    SM->>WS: Unsubscription Confirmed
    WS->>C2: Channel Leave Response
```

## Match API

### 1. Match Operations

```mermaid
graph TB
    subgraph "Match Lifecycle"
        CreateMatch[POST /v2/match<br/>Create Match]
        JoinMatch[POST /v2/match/join<br/>Join Match]
        LeaveMatch[DELETE /v2/match/leave<br/>Leave Match]
        ListMatches[GET /v2/match<br/>List Matches]
    end
    
    subgraph "Match Communication"
        SendData[WebSocket Match Data<br/>Send Game Data]
        ReceiveData[WebSocket Match Data<br/>Receive Updates]
        MatchSignal[Match Signal<br/>Server Message]
        MatchState[Match State<br/>Current Game State]
    end
    
    subgraph "Match Management"
        AuthoritativeMatch[Authoritative Match<br/>Server-controlled]
        RelayMatch[Relay Match<br/>Peer-to-peer]
        PrivateMatch[Private Match<br/>Invite Only]
        PublicMatch[Public Match<br/>Anyone Can Join]
    end
    
    CreateMatch --> SendData
    JoinMatch --> ReceiveData
    LeaveMatch --> MatchSignal
    ListMatches --> MatchState
    
    SendData --> AuthoritativeMatch
    ReceiveData --> RelayMatch
    MatchSignal --> PrivateMatch
    MatchState --> PublicMatch
```

### 2. Matchmaking API

```mermaid
classDiagram
    class MatchmakerAPI {
        +AddMatchmaker(request *api.AddMatchmakerRequest) (*api.MatchmakerTicket, error)
        +RemoveMatchmaker(request *api.RemoveMatchmakerRequest) error
    }
    
    class MatchmakerRequest {
        +string query
        +int32 min_count
        +int32 max_count
        +map string_properties
        +map numeric_properties
        +int32 count_multiple
    }
    
    class MatchmakerTicket {
        +string ticket
    }
    
    class MatchmakerMatched {
        +string ticket
        +string match_id
        +string token
        +repeated MatchmakerUser users
        +MatchmakerUser self
    }
    
    MatchmakerAPI --> MatchmakerRequest : "processes"
    MatchmakerRequest --> MatchmakerTicket : "returns"
    MatchmakerTicket --> MatchmakerMatched : "becomes"
```

## Custom RPC API

### 1. RPC Endpoint Design

```mermaid
graph TB
    subgraph "RPC Registration"
        GoFunction[Go Function<br/>Native Implementation]
        LuaFunction[Lua Function<br/>Scripted Logic]
        JSFunction[JavaScript Function<br/>V8 Runtime]
    end
    
    subgraph "RPC Endpoints"
        HTTPEndpoint[HTTP /v2/rpc/{id}<br/>RESTful Access]
        gRPCEndpoint[gRPC Rpc Method<br/>Binary Protocol]
        WSEndpoint[WebSocket RPC<br/>Real-time Access]
    end
    
    subgraph "RPC Features"
        Authentication[Authentication Required<br/>Session Token]
        RateLimit[Rate Limiting<br/>Per User/Global]
        Validation[Input Validation<br/>Schema Checking]
        Logging[Request Logging<br/>Audit Trail]
    end
    
    GoFunction --> HTTPEndpoint
    LuaFunction --> gRPCEndpoint
    JSFunction --> WSEndpoint
    
    HTTPEndpoint --> Authentication
    gRPCEndpoint --> RateLimit
    WSEndpoint --> Validation
    
    Authentication --> Logging
    RateLimit --> Logging
    Validation --> Logging
```

### 2. RPC Implementation Pattern

```go
// Go RPC function example
func CustomGameLogic(ctx context.Context, logger runtime.Logger, db *sql.DB, nk runtime.NakamaModule, payload string) (string, error) {
    // Parse input payload
    var request struct {
        Action string `json:"action"`
        Data   map[string]interface{} `json:"data"`
    }
    if err := json.Unmarshal([]byte(payload), &request); err != nil {
        return "", runtime.NewError("Invalid JSON payload", 3)
    }
    
    // Get user context
    userID, ok := ctx.Value(runtime.RUNTIME_CTX_USER_ID).(string)
    if !ok {
        return "", runtime.NewError("User ID not found", 13)
    }
    
    // Validate user permissions
    account, err := nk.AccountGetId(ctx, userID)
    if err != nil {
        return "", runtime.NewError("Failed to get user account", 13)
    }
    
    // Custom business logic
    switch request.Action {
    case "purchase_item":
        return handleItemPurchase(ctx, logger, db, nk, account, request.Data)
    case "use_skill":
        return handleSkillUsage(ctx, logger, db, nk, account, request.Data)
    default:
        return "", runtime.NewError("Unknown action", 3)
    }
}
```

## Error Handling and Status Codes

### 1. HTTP Status Codes

```mermaid
graph TB
    subgraph "Success Codes"
        OK[200 OK<br/>Successful Request]
        Created[201 Created<br/>Resource Created]
        NoContent[204 No Content<br/>Successful Delete]
    end
    
    subgraph "Client Error Codes"
        BadRequest[400 Bad Request<br/>Invalid Input]
        Unauthorized[401 Unauthorized<br/>Missing/Invalid Auth]
        Forbidden[403 Forbidden<br/>Insufficient Permissions]
        NotFound[404 Not Found<br/>Resource Not Found]
        Conflict[409 Conflict<br/>Resource Conflict]
        TooManyRequests[429 Too Many Requests<br/>Rate Limit Exceeded]
    end
    
    subgraph "Server Error Codes"
        InternalError[500 Internal Server Error<br/>Server Fault]
        ServiceUnavailable[503 Service Unavailable<br/>Temporary Outage]
        GatewayTimeout[504 Gateway Timeout<br/>Upstream Timeout]
    end
```

### 2. Error Response Format

```json
{
  "error": {
    "code": 3,
    "message": "Invalid argument",
    "details": [
      {
        "type": "ValidationError",
        "field": "username",
        "description": "Username must be between 3 and 20 characters"
      }
    ]
  },
  "grpc_code": 3,
  "http_code": 400,
  "message": "Invalid argument",
  "timestamp": "2023-12-01T14:30:00Z",
  "path": "/v2/account/authenticate/custom"
}
```

## API Versioning and Compatibility

### 1. Versioning Strategy

```mermaid
graph LR
    subgraph "API Versions"
        V1[API v1<br/>Legacy Support]
        V2[API v2<br/>Current Version]
        V3[API v3<br/>Future Version]
    end
    
    subgraph "Compatibility Matrix"
        BackwardCompat[Backward Compatible<br/>v2 supports v1 clients]
        ForwardCompat[Forward Compatible<br/>v1 ignores new v2 fields]
        BreakingChange[Breaking Changes<br/>Major version bump]
    end
    
    subgraph "Migration Strategy"
        GracefulDegrade[Graceful Degradation<br/>Fallback to v1]
        FeatureFlag[Feature Flags<br/>Conditional Features]
        SunsetSchedule[Sunset Schedule<br/>v1 deprecation timeline]
    end
    
    V1 --> BackwardCompat
    V2 --> ForwardCompat
    V3 --> BreakingChange
    
    BackwardCompat --> GracefulDegrade
    ForwardCompat --> FeatureFlag
    BreakingChange --> SunsetSchedule
```

### 2. API Documentation

```mermaid
graph TB
    subgraph "Documentation Types"
        OpenAPISpec[OpenAPI Specification<br/>Machine-readable Schema]
        RESTDocs[REST API Documentation<br/>Human-readable Guides]
        gRPCDocs[gRPC Documentation<br/>Protocol Buffer Definitions]
        SDKDocs[SDK Documentation<br/>Client Library Guides]
    end
    
    subgraph "Interactive Tools"
        SwaggerUI[Swagger UI<br/>Interactive API Explorer]
        PostmanCollection[Postman Collections<br/>API Testing]
        gRPCReflection[gRPC Reflection<br/>Service Discovery]
        CodeSamples[Code Samples<br/>Implementation Examples]
    end
    
    subgraph "Documentation Publishing"
        GitHubPages[GitHub Pages<br/>Static Site]
        DocusaurusPortal[Docusaurus Portal<br/>Developer Portal]
        APIPortal[API Portal<br/>Centralized Hub]
        VersionedDocs[Versioned Docs<br/>Multi-version Support]
    end
    
    OpenAPISpec --> SwaggerUI
    RESTDocs --> PostmanCollection
    gRPCDocs --> gRPCReflection
    SDKDocs --> CodeSamples
    
    SwaggerUI --> GitHubPages
    PostmanCollection --> DocusaurusPortal
    gRPCReflection --> APIPortal
    CodeSamples --> VersionedDocs
```

For more information on related topics:
- [Authentication & Authorization](auth.md) - API security implementation
- [Real-time Communication](realtime.md) - WebSocket API details
- [Runtime Extensions](runtime.md) - Custom RPC implementation