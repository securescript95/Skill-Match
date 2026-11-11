# Skill Exchange — Diagrams and Testing Artifacts

These Mermaid diagrams can be rendered in GitHub Markdown or imported into compatible Mermaid editors. They represent the **proposed target MVP**, not the current HTML prototype.

## 1. Context DFD (Level 0)

```mermaid
flowchart LR
  U[Student Member] -->|profile, skills, swipe, messages, session actions| SE((Skill Exchange System))
  SE -->|profiles, requests, matches, messages, session status| U
  M[Moderator] -->|review report, moderation action| SE
  SE -->|report queue, audit information| M
  SE <-->|verification / notification request| E[Email or Notification Provider]
```

## 2. DFD Level 1

```mermaid
flowchart TB
  U[Member]
  P1((1.0 Account and Profile))
  P2((2.0 Skill Management))
  P3((3.0 Discovery and Interaction))
  P4((4.0 Request and Match))
  P5((5.0 Messaging))
  P6((6.0 Exchange Scheduling))
  D1[(Users and Profiles)]
  D2[(Skills and UserSkills)]
  D3[(Interactions and Matches)]
  D4[(Conversations and Messages)]
  D5[(Exchange Sessions)]
  U --> P1
  P1 <--> D1
  U --> P2
  P2 <--> D2
  P1 --> P3
  P2 --> P3
  P3 <--> D1
  P3 <--> D2
  P3 --> P4
  P4 <--> D3
  P4 --> P5
  P5 <--> D4
  P5 --> P6
  P6 <--> D5
  P4 --> U
  P5 --> U
  P6 --> U
```

## 3. Use-case diagram

```mermaid
flowchart LR
  Visitor[Visitor]
  Member[Registered Member]
  Matched[Matched Member]
  Moderator[Moderator]
  subgraph System[Skill Exchange]
    UC1((Register / Login))
    UC2((Manage Profile))
    UC3((Manage Teach / Learn Skills))
    UC4((Discover Profiles))
    UC5((Pass / Express Interest))
    UC6((Review Incoming Requests))
    UC7((Accept / Reject Request))
    UC8((View Match))
    UC9((Send / Read Messages))
    UC10((Propose / Respond to Session))
    UC11((Block / Report))
    UC12((Review Reports))
  end
  Visitor --> UC1
  Member --> UC1
  Member --> UC2
  Member --> UC3
  Member --> UC4
  Member --> UC5
  Member --> UC6
  Member --> UC7
  Member --> UC8
  Matched --> UC9
  Matched --> UC10
  Member --> UC11
  Moderator --> UC12
```

## 4. Sequence diagram — interest and acceptance

```mermaid
sequenceDiagram
  actor A as Member A
  participant UI as Web Client
  participant API as FastAPI
  participant DB as PostgreSQL
  actor B as Member B
  A->>UI: Express interest in B
  UI->>API: POST /interactions
  API->>API: Authenticate and validate eligibility
  API->>DB: Insert pending interaction
  DB-->>API: Saved interaction
  API-->>UI: 201 Pending request
  B->>UI: Open Requests and accept A
  UI->>API: POST /requests/{id}/accept
  API->>API: Verify recipient and pending state
  API->>DB: Transaction: update request + create unique match + conversation
  DB-->>API: Commit
  API-->>UI: Match created
  API-->>A: Notify match
  API-->>B: Notify match
```

## 5. Sequence diagram — send message

```mermaid
sequenceDiagram
  actor Sender
  participant Client
  participant API
  participant DB
  actor Receiver
  Sender->>Client: Compose message
  Client->>API: POST /conversations/{id}/messages
  API->>DB: Verify active match and participant membership
  DB-->>API: Authorization result
  alt authorized
    API->>DB: Persist message
    DB-->>API: Message record
    API-->>Client: Message accepted
    API-->>Receiver: Deliver / notify
  else unauthorized
    API-->>Client: 403 Forbidden
  end
```

## 6. ER diagram (conceptual)

```mermaid
erDiagram
  USER ||--|| PROFILE : has
  USER ||--o{ USER_SKILL : lists
  SKILL ||--o{ USER_SKILL : classifies
  USER ||--o{ INTERACTION : acts
  USER ||--o{ INTERACTION : receives
  USER ||--o{ MATCH : participant_low
  USER ||--o{ MATCH : participant_high
  MATCH ||--|| CONVERSATION : owns
  CONVERSATION ||--o{ MESSAGE : contains
  USER ||--o{ MESSAGE : sends
  MATCH ||--o{ EXCHANGE_SESSION : schedules
  USER ||--o{ BLOCK : blocks
  USER ||--o{ REPORT : submits
  USER ||--o{ NOTIFICATION : receives
  USER {
    uuid user_id PK
    string email UK
    string password_hash
    string account_status
    datetime created_at
  }
  PROFILE {
    uuid profile_id PK
    uuid user_id FK
    string display_name
    string username
    string bio
    string timezone
  }
  SKILL {
    uuid skill_id PK
    string name
    string category
  }
  USER_SKILL {
    uuid user_skill_id PK
    uuid user_id FK
    uuid skill_id FK
    string role
    string proficiency
  }
  INTERACTION {
    uuid interaction_id PK
    uuid actor_id FK
    uuid target_id FK
    string action
    string status
    datetime created_at
  }
  MATCH {
    uuid match_id PK
    uuid user_low_id FK
    uuid user_high_id FK
    string status
    datetime created_at
  }
  CONVERSATION {
    uuid conversation_id PK
    uuid match_id FK
    datetime created_at
  }
  MESSAGE {
    uuid message_id PK
    uuid conversation_id FK
    uuid sender_id FK
    string body
    datetime created_at
  }
  EXCHANGE_SESSION {
    uuid session_id PK
    uuid match_id FK
    uuid proposer_id FK
    string topic
    datetime starts_at_utc
    int duration_minutes
    string timezone_id
    string status
  }
```

## 7. Control-flow graph — compatibility module

```mermaid
flowchart TD
  A([Start]) --> B{userA.id == userB.id?}
  B -- Yes --> X[Return invalid]
  B -- No --> C[Compute A learns from B and B learns from A]
  C --> D{Both overlap sets empty?}
  D -- Yes --> Y[Return incompatible]
  D -- No --> E{Both overlap sets non-empty?}
  E -- Yes --> R[Return compatible, reciprocal]
  E -- No --> S[Return compatible, one-way]
  X --> Z([End])
  Y --> Z
  R --> Z
  S --> Z
```

Decision nodes = 3, so cyclomatic complexity **V(G) = D + 1 = 4**.

## 8. Basis paths and test cases

| Test ID | Path | Input | Expected result |
|---|---|---|---|
| TC-COMP-01 | Same user | A compared with A | Invalid; no compatibility |
| TC-COMP-02 | No overlap | A teaches Python/wants UI; B teaches Guitar/wants Excel | Incompatible |
| TC-COMP-03 | One-way overlap | A wants Python; B teaches Python; no reverse overlap | Compatible, reciprocal=false |
| TC-COMP-04 | Reciprocal overlap | A teaches UI/wants Python; B teaches Python/wants UI | Compatible, reciprocal=true |
| TC-COMP-05 | Case/whitespace normalization | " Python " vs "python" | Match after normalization |
| TC-COMP-06 | Empty skill sets | Both sets empty | Incompatible |

TC-COMP-05 and 06 are robustness tests beyond the four independent basis paths.

## 9. Additional system-level test scenarios

| ID | Scenario | Expected result |
|---|---|---|
| TC-MATCH-01 | A expresses interest in B | Pending request for B; no match yet |
| TC-MATCH-02 | B accepts A's pending request | Exactly one active match |
| TC-MATCH-03 | B rejects A's pending request | No match |
| TC-MATCH-04 | Duplicate acceptance request | No duplicate match; current state returned |
| TC-MATCH-05 | Member tries to swipe self | Rejected |
| TC-CHAT-01 | Non-participant accesses conversation ID | 403/404; no message data leaked |
| TC-CHAT-02 | Match participant sends message | Message persisted and visible to both |
| TC-SAFE-01 | A blocks B | New messages and discovery between pair denied |
| TC-SESSION-01 | Session uses different participant timezones | Same instant displayed in each local timezone |
| TC-SESSION-02 | Invalid or past session date | Validation error; no session created |

## 10. Prototype-to-target workflow gap

Current prototype: right swipe immediately opens a simulated match screen and creates a sample chat.  
Target requirement: right swipe creates a pending incoming request; recipient accepts; only then is a match and conversation created.  
Implementation task: update client state transitions and enforce them transactionally in the API/database.
