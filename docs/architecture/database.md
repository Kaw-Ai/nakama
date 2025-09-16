# Database Architecture

This document details Nakama's database design, schema structure, and data modeling patterns. Nakama primarily uses CockroachDB but is compatible with PostgreSQL.

## Database Technology Stack

```mermaid
graph TB
    subgraph "Primary Database"
        CRDB[CockroachDB<br/>Distributed SQL]
        PG[PostgreSQL<br/>Compatible Alternative]
    end
    
    subgraph "Connection Management"
        Pool[Connection Pool<br/>pgx/v5]
        Migration[Schema Migration<br/>sql-migrate]
    end
    
    subgraph "Data Access Patterns"
        Direct[Direct SQL Queries]
        Transactions[Database Transactions]
        Prepared[Prepared Statements]
    end
    
    CRDB --> Pool
    PG --> Pool
    Pool --> Direct
    Pool --> Transactions
    Pool --> Prepared
    
    Migration --> CRDB
    Migration --> PG
```

## Schema Overview

```mermaid
erDiagram
    users {
        uuid id PK
        string username UK
        string display_name
        string avatar_url
        string lang_tag
        string location
        string timezone
        jsonb metadata
        timestamp create_time
        timestamp update_time
        timestamp disable_time
    }
    
    user_device {
        uuid id PK
        uuid user_id FK
        string hash UK
        timestamp create_time
        timestamp update_time
    }
    
    storage {
        string collection
        string key
        uuid user_id
        string value
        string version
        int read
        int write
        timestamp create_time
        timestamp update_time
        PRIMARY_KEY collection_key_user_id
    }
    
    leaderboard_record {
        string leaderboard_id
        uuid owner_id
        string username
        bigint score
        bigint subscore
        int num_score
        jsonb metadata
        timestamp create_time
        timestamp update_time
        timestamp expiry_time
        PRIMARY_KEY leaderboard_id_expiry_time_score_subscore_owner_id
    }
    
    groups {
        uuid id PK
        uuid creator_id FK
        string name
        string description
        string lang_tag
        string avatar_url
        int open
        int edge_count
        int max_count
        jsonb metadata
        timestamp create_time
        timestamp update_time
    }
    
    group_edge {
        uuid source_id FK
        uuid destination_id FK
        int state
        int position
        timestamp update_time
        PRIMARY_KEY source_id_destination_id
    }
    
    user_edge {
        uuid source_id FK
        uuid destination_id FK
        int state
        int position
        timestamp update_time
        PRIMARY_KEY source_id_destination_id
    }
    
    tournaments {
        string id PK
        string title
        string description
        int category
        int sort_order
        int size
        int max_size
        int max_num_score
        int join_required
        jsonb metadata
        timestamp create_time
        timestamp start_time
        timestamp end_time
        timestamp reset_schedule
        int duration
        timestamp next_reset
    }
    
    tournament_record {
        string tournament_id FK
        uuid owner_id FK
        string username
        bigint score
        bigint subscore
        int num_score
        jsonb metadata
        int rank_value
        timestamp create_time
        timestamp update_time
        timestamp expiry_time
        PRIMARY_KEY tournament_id_expiry_time_score_subscore_owner_id
    }
    
    users ||--o{ user_device : "owns"
    users ||--o{ storage : "stores"
    users ||--o{ leaderboard_record : "scores"
    users ||--|| groups : "creates"
    users ||--o{ group_edge : "member_of"
    users ||--o{ user_edge : "friend_with"
    users ||--o{ tournament_record : "participates"
    groups ||--o{ group_edge : "contains"
    tournaments ||--o{ tournament_record : "ranks"
```

## Core Data Models

### 1. User Management

```mermaid
classDiagram
    class User {
        +UUID id
        +string username
        +string display_name
        +string avatar_url
        +string lang_tag
        +string location
        +string timezone
        +map metadata
        +timestamp create_time
        +timestamp update_time
        +timestamp disable_time
    }
    
    class UserDevice {
        +UUID id
        +UUID user_id
        +string hash
        +timestamp create_time
        +timestamp update_time
    }
    
    class UserAccount {
        +UUID user_id
        +string email
        +string custom_id
        +string google_id
        +string apple_id
        +string facebook_id
        +string steam_id
        +string gameCenter_id
        +timestamp create_time
        +timestamp update_time
    }
    
    User ||--o{ UserDevice : "has"
    User ||--|| UserAccount : "authenticated_by"
```

### 2. Storage System

```mermaid
classDiagram
    class StorageObject {
        +string collection
        +string key
        +UUID user_id
        +string value
        +string version
        +int permission_read
        +int permission_write
        +timestamp create_time
        +timestamp update_time
    }
    
    class StoragePermission {
        +NO_READ = 0
        +OWNER_READ = 1
        +PUBLIC_READ = 2
        +NO_WRITE = 0
        +OWNER_WRITE = 1
    }
    
    StorageObject --> StoragePermission : "uses"
```

### 3. Social Features

```mermaid
classDiagram
    class UserEdge {
        +UUID source_id
        +UUID destination_id
        +int state
        +int position
        +timestamp update_time
    }
    
    class EdgeState {
        +FRIEND = 0
        +INVITE_SENT = 1
        +INVITE_RECEIVED = 2
        +BLOCKED = 3
    }
    
    class Group {
        +UUID id
        +UUID creator_id
        +string name
        +string description
        +string lang_tag
        +string avatar_url
        +int open
        +int edge_count
        +int max_count
        +map metadata
        +timestamp create_time
        +timestamp update_time
    }
    
    class GroupEdge {
        +UUID source_id
        +UUID destination_id
        +int state
        +int position
        +timestamp update_time
    }
    
    UserEdge --> EdgeState : "has"
    Group ||--o{ GroupEdge : "contains"
```

## Data Access Patterns

### 1. Read Patterns

```mermaid
graph TB
    subgraph "Read Operations"
        SingleRead[Single Record Read<br/>By Primary Key]
        BatchRead[Batch Read<br/>Multiple Records]
        IndexRead[Index-based Read<br/>Secondary Indices]
        ScanRead[Sequential Scan<br/>Full Table]
    end
    
    subgraph "Performance Characteristics"
        Fast[O(1) - Very Fast]
        Moderate[O(log n) - Moderate]
        Slow[O(n) - Slow]
    end
    
    SingleRead --> Fast
    IndexRead --> Fast
    BatchRead --> Moderate
    ScanRead --> Slow
```

### 2. Write Patterns

```mermaid
graph TB
    subgraph "Write Operations"
        Insert[Insert New Record]
        Update[Update Existing]
        Upsert[Insert or Update]
        Delete[Delete Record]
        Batch[Batch Operations]
    end
    
    subgraph "Transaction Patterns"
        Single[Single Statement]
        Multi[Multi-Statement]
        Conditional[Conditional Writes]
        Atomic[Atomic Operations]
    end
    
    Insert --> Single
    Update --> Multi
    Upsert --> Conditional
    Delete --> Single
    Batch --> Atomic
```

## Query Optimization Strategies

### 1. Indexing Strategy

```sql
-- Primary indices (automatically created)
PRIMARY KEY (id)
PRIMARY KEY (collection, key, user_id)

-- Secondary indices for common queries
CREATE INDEX users_username_index ON users (username) WHERE disable_time IS NULL;
CREATE INDEX storage_user_collection_index ON storage (user_id, collection);
CREATE INDEX leaderboard_record_rank_index ON leaderboard_record (leaderboard_id, expiry_time, score DESC, subscore DESC);
CREATE INDEX group_edge_destination_index ON group_edge (destination_id, state, position);
CREATE INDEX user_edge_destination_index ON user_edge (destination_id, state, position);

-- Composite indices for complex queries
CREATE INDEX tournament_record_tournament_rank ON tournament_record (tournament_id, expiry_time, rank_value);
```

### 2. Query Performance Patterns

```mermaid
graph LR
    subgraph "High Performance"
        PK[Primary Key Lookup]
        UK[Unique Key Lookup]
        IndexSeek[Index Seek]
    end
    
    subgraph "Medium Performance"
        IndexScan[Index Scan]
        FilteredScan[Filtered Scan]
        Join[Indexed Join]
    end
    
    subgraph "Low Performance"
        TableScan[Full Table Scan]
        UnindexedJoin[Unindexed Join]
        ComplexFilter[Complex Filtering]
    end
    
    PK --> IndexSeek
    UK --> IndexSeek
    IndexScan --> FilteredScan
    Join --> FilteredScan
```

## Transaction Management

### 1. Transaction Isolation Levels

```mermaid
graph TB
    subgraph "Isolation Levels"
        RC[Read Committed<br/>Default Level]
        RR[Repeatable Read<br/>For Consistency]
        S[Serializable<br/>For Critical Operations]
    end
    
    subgraph "Use Cases"
        ReadOps[Read Operations<br/>User Data Fetching]
        WriteOps[Write Operations<br/>Score Updates]
        CriticalOps[Critical Operations<br/>Tournament Results]
    end
    
    RC --> ReadOps
    RR --> WriteOps
    S --> CriticalOps
```

### 2. Transaction Patterns

```sql
-- Simple transaction
BEGIN;
UPDATE users SET display_name = $1, update_time = now() WHERE id = $2;
COMMIT;

-- Complex transaction with conditional logic
BEGIN;
SELECT version FROM storage WHERE collection = $1 AND key = $2 AND user_id = $3 FOR UPDATE;
-- Check version and update conditionally
UPDATE storage SET value = $4, version = $5, update_time = now() 
WHERE collection = $1 AND key = $2 AND user_id = $3 AND version = $6;
COMMIT;

-- Batch operation
BEGIN;
INSERT INTO leaderboard_record (leaderboard_id, owner_id, username, score, ...)
VALUES ($1, $2, $3, $4, ...), ($5, $6, $7, $8, ...), ...
ON CONFLICT (leaderboard_id, expiry_time, score, subscore, owner_id) 
DO UPDATE SET score = EXCLUDED.score, update_time = now();
COMMIT;
```

## Data Consistency Patterns

### 1. Eventual Consistency

```mermaid
sequenceDiagram
    participant App as Application
    participant DB1 as Database Node 1
    participant DB2 as Database Node 2
    participant DB3 as Database Node 3
    
    App->>DB1: Write Operation
    DB1->>App: Write Acknowledged
    DB1->>DB2: Replicate Data
    DB1->>DB3: Replicate Data
    DB2->>DB1: Replication Confirmed
    DB3->>DB1: Replication Confirmed
    
    Note over DB1,DB3: All nodes eventually consistent
```

### 2. Strong Consistency

```mermaid
sequenceDiagram
    participant App as Application
    participant DB1 as Database Node 1
    participant DB2 as Database Node 2
    participant DB3 as Database Node 3
    
    App->>DB1: Critical Write Operation
    DB1->>DB2: Synchronous Replication
    DB1->>DB3: Synchronous Replication
    DB2->>DB1: Replication Confirmed
    DB3->>DB1: Replication Confirmed
    DB1->>App: Write Acknowledged
    
    Note over DB1,DB3: Strong consistency maintained
```

## Scaling Strategies

### 1. Horizontal Scaling (CockroachDB)

```mermaid
graph TB
    subgraph "CockroachDB Cluster"
        Node1[Node 1<br/>Region: US-East]
        Node2[Node 2<br/>Region: US-West]
        Node3[Node 3<br/>Region: EU-West]
        Node4[Node 4<br/>Region: AP-Southeast]
    end
    
    subgraph "Data Distribution"
        Range1[Range 1<br/>Users A-F]
        Range2[Range 2<br/>Users G-M]
        Range3[Range 3<br/>Users N-S]
        Range4[Range 4<br/>Users T-Z]
    end
    
    Node1 --> Range1
    Node2 --> Range2
    Node3 --> Range3
    Node4 --> Range4
    
    Node1 -.->|Replicas| Node2
    Node2 -.->|Replicas| Node3
    Node3 -.->|Replicas| Node4
    Node4 -.->|Replicas| Node1
```

### 2. Read Replicas (PostgreSQL)

```mermaid
graph TB
    subgraph "Master Database"
        Master[PostgreSQL Master<br/>Write Operations]
    end
    
    subgraph "Read Replicas"
        Replica1[Read Replica 1<br/>User Queries]
        Replica2[Read Replica 2<br/>Analytics]
        Replica3[Read Replica 3<br/>Reporting]
    end
    
    Master --> Replica1
    Master --> Replica2
    Master --> Replica3
    
    Replica1 -.->|Load Balance| Replica2
    Replica2 -.->|Load Balance| Replica3
```

## Data Migration and Versioning

### 1. Schema Migration Process

```mermaid
graph LR
    subgraph "Migration Stages"
        Dev[Development<br/>Schema Changes]
        Test[Testing<br/>Migration Scripts]
        Stage[Staging<br/>Full Migration Test]
        Prod[Production<br/>Zero-Downtime Deploy]
    end
    
    subgraph "Migration Tools"
        Scripts[SQL Migration Scripts]
        Rollback[Rollback Procedures]
        Validation[Data Validation]
        Monitoring[Migration Monitoring]
    end
    
    Dev --> Test
    Test --> Stage
    Stage --> Prod
    
    Scripts --> Dev
    Rollback --> Test
    Validation --> Stage
    Monitoring --> Prod
```

### 2. Zero-Downtime Migration Strategy

```mermaid
sequenceDiagram
    participant App as Application
    participant OldSchema as Old Schema
    participant NewSchema as New Schema
    participant Migration as Migration Process
    
    Migration->>NewSchema: Create New Schema
    Migration->>App: Deploy Dual-Write Code
    App->>OldSchema: Write to Old Schema
    App->>NewSchema: Write to New Schema
    Migration->>NewSchema: Backfill Historical Data
    Migration->>App: Deploy New-Read Code
    App->>NewSchema: Read from New Schema
    Migration->>OldSchema: Drop Old Schema
```

## Performance Monitoring

### 1. Key Metrics

```mermaid
graph TB
    subgraph "Database Metrics"
        Latency[Query Latency<br/>P50, P95, P99]
        Throughput[Throughput<br/>QPS, TPS]
        Errors[Error Rate<br/>Failed Queries]
        Connections[Connection Pool<br/>Active/Idle]
    end
    
    subgraph "System Metrics"
        CPU[CPU Usage<br/>Per Node]
        Memory[Memory Usage<br/>Cache Hit Rate]
        Disk[Disk I/O<br/>Read/Write IOPS]
        Network[Network I/O<br/>Bandwidth Usage]
    end
    
    subgraph "Business Metrics"
        Users[Active Users<br/>Concurrent Sessions]
        Storage[Storage Growth<br/>Data Volume]
        Operations[Operation Types<br/>Read/Write Ratio]
    end
```

### 2. Query Performance Analysis

```sql
-- Slow query analysis (PostgreSQL)
SELECT query, calls, total_time, mean_time, rows
FROM pg_stat_statements
ORDER BY mean_time DESC
LIMIT 10;

-- Index usage analysis
SELECT schemaname, tablename, attname, n_distinct, correlation
FROM pg_stats
WHERE tablename IN ('users', 'storage', 'leaderboard_record')
ORDER BY n_distinct DESC;

-- Connection monitoring
SELECT state, count(*)
FROM pg_stat_activity
GROUP BY state;
```

## Backup and Recovery

### 1. Backup Strategy

```mermaid
graph TB
    subgraph "Backup Types"
        Full[Full Backup<br/>Complete Database]
        Incremental[Incremental Backup<br/>Changes Since Last]
        Point[Point-in-Time<br/>Transaction Log]
    end
    
    subgraph "Backup Schedule"
        Daily[Daily Full Backup<br/>Off-peak Hours]
        Hourly[Hourly Incremental<br/>Continuous]
        PITR[Continuous WAL<br/>Point-in-Time Recovery]
    end
    
    subgraph "Storage Locations"
        Local[Local Storage<br/>Fast Recovery]
        Cloud[Cloud Storage<br/>Long-term Retention]
        Geographic[Geographic Replication<br/>Disaster Recovery]
    end
    
    Full --> Daily
    Incremental --> Hourly
    Point --> PITR
    
    Daily --> Local
    Hourly --> Cloud
    PITR --> Geographic
```

### 2. Recovery Procedures

```mermaid
graph LR
    subgraph "Recovery Scenarios"
        Corruption[Data Corruption<br/>Restore from Backup]
        Hardware[Hardware Failure<br/>Switch to Replica]
        Disaster[Disaster Recovery<br/>Geographic Failover]
    end
    
    subgraph "Recovery Steps"
        Assessment[Assess Damage]
        Isolation[Isolate Failed Components]
        Restore[Restore from Backup]
        Validation[Validate Data Integrity]
        Switchover[Switch Traffic]
    end
    
    Corruption --> Assessment
    Hardware --> Isolation
    Disaster --> Restore
    Assessment --> Validation
    Isolation --> Switchover
```

For more information on related topics:
- [Component Architecture](components.md) - How components interact with the database
- [API Architecture](api.md) - How APIs access and modify data
- [Deployment Architecture](deployment.md) - Production database deployment patterns