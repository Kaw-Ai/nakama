# Authentication & Authorization Architecture

This document details Nakama's security architecture, authentication mechanisms, and authorization patterns.

## Security Overview

```mermaid
graph TB
    subgraph "Client Security"
        TLS[TLS Encryption<br/>Transport Security]
        ClientAuth[Client Authentication<br/>API Keys & Tokens]
        DeviceId[Device Fingerprinting<br/>Unique Identification]
    end
    
    subgraph "Authentication Layer"
        MultiAuth[Multi-Provider Auth<br/>Social & Custom]
        JWT[JWT Token Management<br/>Session Tokens]
        MFA[Multi-Factor Auth<br/>TOTP Support]
    end
    
    subgraph "Authorization Layer"
        RBAC[Role-Based Access<br/>User Permissions]
        ResourceAuth[Resource Authorization<br/>Ownership & Permissions]
        RateLimit[Rate Limiting<br/>Abuse Prevention]
    end
    
    subgraph "Backend Security"
        Encryption[Data Encryption<br/>At Rest & Transit]
        Audit[Audit Logging<br/>Security Events]
        Monitoring[Security Monitoring<br/>Threat Detection]
    end
    
    TLS --> MultiAuth
    ClientAuth --> JWT
    DeviceId --> MFA
    
    MultiAuth --> RBAC
    JWT --> ResourceAuth
    MFA --> RateLimit
    
    RBAC --> Encryption
    ResourceAuth --> Audit
    RateLimit --> Monitoring
```

## Authentication Mechanisms

### 1. Multi-Provider Authentication

```mermaid
graph TB
    subgraph "Social Providers"
        Google[Google OAuth2<br/>Play Games, Gmail]
        Apple[Apple ID<br/>Sign in with Apple]
        Facebook[Facebook Login<br/>Graph API]
        Steam[Steam OpenID<br/>Steam Platform]
        GameCenter[Game Center<br/>iOS Gaming]
    end
    
    subgraph "Custom Authentication"
        Email[Email/Password<br/>Traditional Auth]
        Device[Device ID<br/>Anonymous Auth]
        Custom[Custom ID<br/>External Systems]
    end
    
    subgraph "Authentication Flow"
        Verification[Provider Verification]
        UserCreation[User Account Creation]
        SessionCreation[Session Token Creation]
    end
    
    Google --> Verification
    Apple --> Verification
    Facebook --> Verification
    Steam --> Verification
    GameCenter --> Verification
    
    Email --> UserCreation
    Device --> UserCreation
    Custom --> UserCreation
    
    Verification --> SessionCreation
    UserCreation --> SessionCreation
```

### 2. Authentication Flow Details

```mermaid
sequenceDiagram
    participant C as Client
    participant API as Auth API
    participant Provider as OAuth Provider
    participant Auth as Auth Service
    participant DB as Database
    participant Session as Session Registry
    
    C->>API: Authentication Request
    
    alt Social Authentication
        API->>Provider: Verify OAuth Token
        Provider->>API: User Profile Data
    else Custom Authentication
        API->>Auth: Verify Credentials
        Auth->>DB: Lookup User Account
        DB->>Auth: User Data
    end
    
    API->>Auth: Process Authentication
    
    alt New User
        Auth->>DB: Create User Account
        DB->>Auth: User Created
    else Existing User
        Auth->>DB: Update Last Login
        DB->>Auth: Updated
    end
    
    Auth->>Session: Create Session
    Session->>Auth: Session Token
    Auth->>API: JWT Token
    API->>C: Authentication Response
```

## JWT Token Architecture

### 1. Token Structure

```mermaid
classDiagram
    class JWTHeader {
        +string alg "HS256"
        +string typ "JWT"
    }
    
    class JWTPayload {
        +string uid "User ID"
        +string sid "Session ID" 
        +string usn "Username"
        +int64 exp "Expiry Time"
        +int64 iat "Issued At"
        +map vars "Session Variables"
        +int fmt "Session Format"
    }
    
    class JWTSignature {
        +string signature "HMAC SHA256"
        +string secret "Server Secret"
    }
    
    JWTHeader --> JWTPayload : "header.payload"
    JWTPayload --> JWTSignature : "payload.signature"
```

### 2. Token Lifecycle

```mermaid
stateDiagram-v2
    [*] --> Issued
    Issued --> Active: Token Validated
    Active --> Refreshed: Before Expiry
    Active --> Expired: After Expiry
    Refreshed --> Active: New Token Issued
    Expired --> Revoked: Manual Revocation
    Active --> Revoked: Security Event
    Revoked --> [*]
    Expired --> [*]
    
    state Active {
        [*] --> Valid
        Valid --> RateLimited: Abuse Detected
        RateLimited --> Valid: Rate Limit Reset
    }
```

## Session Management

### 1. Session Architecture

```mermaid
classDiagram
    class Session {
        +UUID sessionID
        +UUID userID
        +string username
        +int64 expiry
        +map vars
        +SessionFormat format
        +context.Context ctx
        +*zap.Logger logger
        +string clientIP
        +string clientPort
    }
    
    class SessionRegistry {
        +sessions sync.Map
        +Add(session *Session) error
        +Remove(sessionID string) bool
        +Get(sessionID string) *Session
        +Count() int
        +List() []*Session
    }
    
    class SessionFormat {
        +JSON = 0
        +PROTOBUF = 1
    }
    
    SessionRegistry --> Session : "manages"
    Session --> SessionFormat : "uses"
```

### 2. Session Security Features

```mermaid
graph TB
    subgraph "Session Security"
        Expiry[Session Expiry<br/>Configurable Timeout]
        Revocation[Session Revocation<br/>Immediate Invalidation]
        Refresh[Token Refresh<br/>Sliding Expiration]
        Variables[Session Variables<br/>Custom Metadata]
    end
    
    subgraph "Security Monitoring"
        IPTracking[IP Address Tracking<br/>Location Changes]
        DeviceTracking[Device Tracking<br/>Device Changes]
        ActivityTracking[Activity Tracking<br/>Unusual Patterns]
        BruteForce[Brute Force Protection<br/>Failed Attempts]
    end
    
    Expiry --> IPTracking
    Revocation --> DeviceTracking
    Refresh --> ActivityTracking
    Variables --> BruteForce
```

## Authorization Patterns

### 1. Resource-Based Authorization

```mermaid
graph TB
    subgraph "Authorization Checks"
        Ownership[Resource Ownership<br/>User Owns Resource]
        Permissions[Permission Levels<br/>Read/Write Access]
        Groups[Group Membership<br/>Shared Resources]
        Public[Public Access<br/>No Restrictions]
    end
    
    subgraph "Resource Types"
        Storage[Storage Objects<br/>User/Global Data]
        Matches[Match Data<br/>Participant Access]
        Groups[Group Resources<br/>Member Access]
        Friends[Friend Data<br/>Relationship Access]
    end
    
    Ownership --> Storage
    Permissions --> Matches
    Groups --> Groups
    Public --> Friends
```

### 2. Permission Matrix

```mermaid
graph LR
    subgraph "Permission Levels"
        NoAccess[No Access<br/>0]
        OwnerRead[Owner Read<br/>1]
        PublicRead[Public Read<br/>2]
        OwnerWrite[Owner Write<br/>1]
        NoWrite[No Write<br/>0]
    end
    
    subgraph "Storage Permissions"
        Private[Private Storage<br/>Read: 1, Write: 1]
        ReadOnly[Read-Only Public<br/>Read: 2, Write: 1]
        PublicRW[Public Read/Write<br/>Read: 2, Write: 2]
        SystemOnly[System Only<br/>Read: 0, Write: 0]
    end
    
    OwnerRead --> Private
    PublicRead --> ReadOnly
    PublicRead --> PublicRW
    NoAccess --> SystemOnly
```

## Multi-Factor Authentication

### 1. MFA Flow

```mermaid
sequenceDiagram
    participant U as User
    participant C as Client
    participant API as Auth API
    participant MFA as MFA Service
    participant TOTP as TOTP Provider
    
    U->>C: Initial Authentication
    C->>API: Username/Password
    API->>API: Validate Credentials
    
    alt MFA Required
        API->>C: MFA Challenge Required
        C->>U: Request MFA Code
        U->>C: Enter TOTP Code
        C->>API: Submit MFA Code
        API->>MFA: Validate TOTP
        MFA->>TOTP: Verify Code
        TOTP->>MFA: Validation Result
        MFA->>API: MFA Success
    end
    
    API->>C: Authentication Success
    C->>U: Logged In
```

### 2. TOTP Implementation

```mermaid
classDiagram
    class TOTPService {
        +GenerateSecret() string
        +GenerateQRCode(secret, username) string
        +ValidateCode(secret, code) bool
        +GenerateRecoveryCodes() []string
    }
    
    class MFAData {
        +string secret
        +bool enabled
        +[]string recoveryCodes
        +timestamp lastUsed
        +int failedAttempts
    }
    
    class MFAValidation {
        +string code
        +int64 timestamp
        +bool isRecoveryCode
        +string userAgent
        +string clientIP
    }
    
    TOTPService --> MFAData : "manages"
    TOTPService --> MFAValidation : "validates"
```

## API Security

### 1. API Authentication

```mermaid
graph TB
    subgraph "API Access Methods"
        ServerKey[Server Key<br/>Admin Operations]
        SessionToken[Session Token<br/>User Operations]
        HTTPBasic[HTTP Basic Auth<br/>Legacy Support]
    end
    
    subgraph "API Endpoints"
        Public[Public Endpoints<br/>Authentication, Health]
        Protected[Protected Endpoints<br/>User Data, Operations]
        Admin[Admin Endpoints<br/>Console, Management]
    end
    
    subgraph "Security Middleware"
        KeyValidation[Key Validation]
        TokenValidation[Token Validation]
        RateLimiting[Rate Limiting]
        RequestLogging[Request Logging]
    end
    
    ServerKey --> Admin
    SessionToken --> Protected
    HTTPBasic --> Public
    
    Admin --> KeyValidation
    Protected --> TokenValidation
    Public --> RateLimiting
    TokenValidation --> RequestLogging
```

### 2. Rate Limiting Architecture

```mermaid
classDiagram
    class RateLimiter {
        +map buckets
        +CheckLimit(key string, limit int, window time.Duration) bool
        +GetRemaining(key string) int
        +Reset(key string)
    }
    
    class TokenBucket {
        +int capacity
        +int tokens
        +time.Time lastRefill
        +time.Duration refillRate
        +Consume(tokens int) bool
        +Refill()
    }
    
    class RateLimit {
        +string key
        +int limit
        +time.Duration window
        +RateLimitType type
    }
    
    class RateLimitType {
        +USER_GLOBAL = "user_global"
        +USER_ENDPOINT = "user_endpoint"
        +IP_GLOBAL = "ip_global"
        +API_KEY = "api_key"
    }
    
    RateLimiter --> TokenBucket : "manages"
    RateLimiter --> RateLimit : "enforces"
    RateLimit --> RateLimitType : "categorizes"
```

## Runtime Security

### 1. Runtime Code Isolation

```mermaid
graph TB
    subgraph "Runtime Environments"
        GoRuntime[Go Runtime<br/>Native Code]
        LuaVM[Lua VM<br/>Sandboxed]
        JSEngine[JavaScript Engine<br/>V8 Isolation]
    end
    
    subgraph "Security Boundaries"
        ProcessIsolation[Process Isolation<br/>Separate Processes]
        MemoryIsolation[Memory Isolation<br/>Heap Separation]
        APIRestrictions[API Restrictions<br/>Limited Access]
        TimeoutLimits[Timeout Limits<br/>Execution Time]
    end
    
    subgraph "Access Controls"
        DatabaseAccess[Database Access<br/>Read/Write Permissions]
        HTTPAccess[HTTP Access<br/>External API Calls]
        FileAccess[File Access<br/>Local File System]
        NetworkAccess[Network Access<br/>Socket Operations]
    end
    
    GoRuntime --> ProcessIsolation
    LuaVM --> MemoryIsolation
    JSEngine --> APIRestrictions
    APIRestrictions --> TimeoutLimits
    
    TimeoutLimits --> DatabaseAccess
    DatabaseAccess --> HTTPAccess
    HTTPAccess --> FileAccess
    FileAccess --> NetworkAccess
```

### 2. Runtime Permission Model

```mermaid
classDiagram
    class RuntimeContext {
        +UUID userID
        +string username
        +map vars
        +*zap.Logger logger
        +RuntimePermissions permissions
        +ExecutionLimits limits
    }
    
    class RuntimePermissions {
        +bool databaseRead
        +bool databaseWrite
        +bool httpClient
        +bool fileSystem
        +[]string allowedDomains
        +map customPermissions
    }
    
    class ExecutionLimits {
        +time.Duration maxExecutionTime
        +int64 maxMemoryUsage
        +int maxHTTPRequests
        +int maxDatabaseQueries
    }
    
    RuntimeContext --> RuntimePermissions : "enforces"
    RuntimeContext --> ExecutionLimits : "applies"
```

## Security Monitoring

### 1. Security Events

```mermaid
graph TB
    subgraph "Authentication Events"
        LoginSuccess[Successful Login]
        LoginFailure[Failed Login]
        AccountLockout[Account Lockout]
        PasswordChange[Password Change]
        MFAFailure[MFA Failure]
    end
    
    subgraph "Authorization Events"
        AccessDenied[Access Denied]
        PermissionEscalation[Permission Escalation]
        ResourceAccess[Resource Access]
        APIAbuse[API Abuse]
    end
    
    subgraph "System Events"
        ConfigChange[Configuration Change]
        KeyRotation[Key Rotation]
        SecurityUpdate[Security Update]
        RuntimeError[Runtime Error]
    end
    
    subgraph "Response Actions"
        AlertGeneration[Alert Generation]
        AutoBlocking[Automatic Blocking]
        AuditLogging[Audit Logging]
        IncidentResponse[Incident Response]
    end
    
    LoginFailure --> AlertGeneration
    APIAbuse --> AutoBlocking
    AccessDenied --> AuditLogging
    ConfigChange --> IncidentResponse
```

### 2. Audit Logging

```mermaid
classDiagram
    class SecurityEvent {
        +UUID eventID
        +string eventType
        +UUID userID
        +string username
        +string clientIP
        +string userAgent
        +map metadata
        +timestamp eventTime
        +SecurityLevel level
    }
    
    class SecurityLevel {
        +INFO = "info"
        +WARN = "warn"
        +ERROR = "error"
        +CRITICAL = "critical"
    }
    
    class AuditLogger {
        +LogEvent(event SecurityEvent)
        +QueryEvents(filters map[string]interface{}) []SecurityEvent
        +AlertOnPattern(pattern string, threshold int)
        +ArchiveOldEvents(age time.Duration)
    }
    
    SecurityEvent --> SecurityLevel : "categorized_by"
    AuditLogger --> SecurityEvent : "manages"
```

## Threat Model

### 1. Attack Vectors

```mermaid
graph TB
    subgraph "External Threats"
        BruteForce[Brute Force Attacks<br/>Credential Guessing]
        DDoS[DDoS Attacks<br/>Service Disruption]
        Injection[Injection Attacks<br/>SQL/NoSQL Injection]
        MITM[Man-in-the-Middle<br/>Traffic Interception]
    end
    
    subgraph "Internal Threats"
        PrivilegeEscalation[Privilege Escalation<br/>Unauthorized Access]
        DataExfiltration[Data Exfiltration<br/>Sensitive Data Theft]
        RuntimeExploit[Runtime Exploits<br/>Code Execution]
        ConfigManipulation[Config Manipulation<br/>System Compromise]
    end
    
    subgraph "Mitigation Strategies"
        RateLimiting[Rate Limiting<br/>Request Throttling]
        Encryption[Strong Encryption<br/>Data Protection]
        InputValidation[Input Validation<br/>Sanitization]
        AccessControl[Access Control<br/>Least Privilege]
        Monitoring[Security Monitoring<br/>Threat Detection]
    end
    
    BruteForce --> RateLimiting
    DDoS --> RateLimiting
    Injection --> InputValidation
    MITM --> Encryption
    PrivilegeEscalation --> AccessControl
    DataExfiltration --> Monitoring
    RuntimeExploit --> InputValidation
    ConfigManipulation --> AccessControl
```

### 2. Security Best Practices

```mermaid
graph LR
    subgraph "Development Security"
        SecureCoding[Secure Coding<br/>Best Practices]
        CodeReview[Security Code Review<br/>Peer Review]
        VulnScanning[Vulnerability Scanning<br/>Automated Testing]
        PenTesting[Penetration Testing<br/>Security Assessment]
    end
    
    subgraph "Operational Security"
        KeyManagement[Key Management<br/>Rotation & Storage]
        AccessReview[Access Review<br/>Regular Audits]
        IncidentResponse[Incident Response<br/>Security Procedures]
        SecurityTraining[Security Training<br/>Team Education]
    end
    
    subgraph "Infrastructure Security"
        NetworkSecurity[Network Security<br/>Firewalls & VPNs]
        HostSecurity[Host Security<br/>OS Hardening]
        ContainerSecurity[Container Security<br/>Image Scanning]
        CloudSecurity[Cloud Security<br/>IAM & Compliance]
    end
    
    SecureCoding --> KeyManagement
    CodeReview --> AccessReview
    VulnScanning --> IncidentResponse
    PenTesting --> SecurityTraining
    
    KeyManagement --> NetworkSecurity
    AccessReview --> HostSecurity
    IncidentResponse --> ContainerSecurity
    SecurityTraining --> CloudSecurity
```

For more information on related topics:
- [API Architecture](api.md) - API security implementation details
- [Runtime Extensions](runtime.md) - Runtime security and isolation
- [Deployment Architecture](deployment.md) - Production security considerations