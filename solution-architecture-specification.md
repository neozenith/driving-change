# Serverless Collaborative Application Architecture – 2025-06-25

This document captures the architectural vision, implementation patterns, and confirmed decisions for a serverless, real-time collaborative web application.

---

## ✅ Confirmed Architectural Decisions

### Runtime Stack

- **Language:** Python
- **Web framework:** FastAPI
- **Testing framework:** PyTest
- **Infrastructure as Code:** AWS CDK
- **Local Development:** LocalStack via `docker-compose.yml`

### Cloud Architecture

- **Cloud Provider:** AWS
- **Compute:** AWS Lambda
- **Storage:** DynamoDB
- **Pub/Sub:** DynamoDB Streams
- **Authentication:** AWS Cognito (user sign-up/login required)
- **Session Validation:** JWT (ID tokens passed via WebSocket messages)
- **Data API (Read-Only):** DuckDB connected to S3 in Delta Lake format
- **Frontend Hosting:** React app hosted on S3 and served via CloudFront
- **Authentication Routing:** Lambda\@Edge for redirect + token enforcement at CloudFront edge

### Application Architecture

- **WebSocket-based collaboration:** API Gateway WebSocket API
- **CRDT state model:** Each collaborative session is scoped to a DynamoDB table
- **Diff-only updates:** Clients send and receive changes as diffs to avoid large state sync
- **Optimistic concurrency:** Conditional writes using `ConditionExpression` on version field
- **Stream-driven updates:** DynamoDB Streams notify other clients and Lambdas
- **No locks:** Conflict resolution is CRDT-based only
- **Retry-on-failure logic:** On conditional write failure, merge and retry
- **UI Architecture:** React app connects to WebSocket after Cognito login

---

## 🧪 Explored But Not Finalized

- CRDT implementation details (type, structure, and merge resolution)
- Stream de-duplication cache/strategy
- Delta serialization/compression
- Conflict retry UX: marking, retrying, or visual feedback to users

---

## 🚫 Explicitly Rejected

- Distributed locking (due to complexity and bottlenecks)

---

## 📊 Sequence Diagram – Real-Time CRDT Sync with Retry Logic

```mermaid
sequenceDiagram
  participant TabA as Browser Tab A
  participant LambdaA as Lambda A
  participant Dynamo as DynamoDB
  participant Stream as DynamoDB Stream
  participant LambdaB as Lambda B
  participant TabB as Browser Tab B

  Note over TabA: User makes change
  TabA->>LambdaA: Send CRDT diff with version V1
  LambdaA->>Dynamo: Conditional write (if version == V1)
  Dynamo-->>LambdaA: Success
  Dynamo-->>Stream: Emit stream record with new state
  Stream-->>LambdaB: Trigger Lambda B
  LambdaB->>TabB: Push updated CRDT state

  Note over TabB: User makes change (before seeing update)
  TabB->>LambdaB: Send CRDT diff with version V1 (stale)
  LambdaB->>Dynamo: Conditional write (if version == V1)
  Dynamo-->>LambdaB: ConditionalCheckFailedException

  alt Stream update has already arrived
      LambdaB->>LambdaB: Merge local diff into latest state
      LambdaB->>Dynamo: Retry conditional write (if version == V2)
      alt Retry succeeds
          Dynamo-->>LambdaB: Success
          Dynamo-->>Stream: Emit updated state
          LambdaB->>TabB: Confirm merge success
      else Retry fails again
          LambdaB->>TabB: Notify write failed, mark local state as unsynced
      end
  else Stream hasn’t arrived yet
      LambdaB->>TabB: Wait or notify conflict, mark local change as unsynced
  end
```

---

## 📈 State Diagram – CRDT Conflict Resolution

```mermaid
stateDiagram-v2
  [*] --> V1 : Initial session state loaded

  state V1 {
      [*] --> TabA_Edit : Tab A sends update DeltaA
      [*] --> TabB_Edit : Tab B sends update DeltaB (stale)
  }

  TabA_Edit --> V2 : DynamoDB write succeeds
  V2 --> [*] : Stream propagates DeltaA

  TabB_Edit --> Conflict : Conditional write fails (based on V1)

  state Conflict {
      [*] --> MergeRetry : Lambda receives DeltaA from stream
      MergeRetry --> V3 : DeltaB merged into DeltaA to form DeltaMerged
      V3 --> [*] : Conditional write succeeds
      MergeRetry --> Failed : Retry also fails
  }

  Failed --> [*] : Notify Tab B of failed update
```

---

## 🏗️ Infrastructure Diagram – Cloud Resources (CDK)

```mermaid
graph TD
  subgraph Frontend
    A["React App"] -->|Deployed to| B["S3 Bucket (static)"]
    B -->|Served via| C["CloudFront"]
    C -->|JWT check| D["Lambda@Edge"]
  end

  subgraph Auth
    E["Cognito Hosted UI"] -->|Auth Flow| F["Cognito User Pool"]
    F --> G["JWT issued"]
  end

  subgraph Backend
    H["API Gateway - WebSocket API"] --> I["Lambda (WebSocket handler)"]
    I --> J["DynamoDB - Session Table"]
    J --> K["DynamoDB Stream"]
    K --> L["Lambda (Stream processor)"]
    L --> M["Browser Clients via WebSocket"]
  end

  subgraph DataAPI
    N["API Gateway - REST /api/"] --> O["Lambda (DuckDB query)"]
    O --> P["S3 - Delta Lake Format"]
  end
```

---

This document will evolve as implementation progresses and more design decisions are locked in. 

