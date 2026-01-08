# Node 4 (Lab Node) Architecture Diagrams

## 1. 🏗️ Internal Architecture (Component Diagram)
Node 4 manages execution data (student logs, LMS features). It is the primary "Write" node for student activity.

```mermaid
graph TD
    %% Node 4: Internal Architecture

    subgraph Interface_Layer [Interface Layer]
        direction TB
        MCP["MCP Server"]
        GRPC["gRPC Service"]
        REST["FastAPI Rest (Legacy)"]
    end

    subgraph Service_Layer [Service Layer]
        direction TB
        Student["StudentService"]
        Logger["ActivityLogger"]
        BKT["HeatmapEngine"]
    end

    subgraph Data_Layer [Data Layer]
        direction TB
        PG[("PostgreSQL (User Data)")]
        Cache[("Redis (Session)")]
    end

    MCP -->|"get_failure_pattern"| Student
    MCP -->|"log_activity"| Logger
    
    GRPC -->|"GetStudentProfile"| Student

    Student -->|"CRUD User"| PG
    BKT -->|"Update Mastery"| PG
    Logger -->|"Insert Log"| PG
```

## 2. 🔗 Dual Protocol Sequence Diagram
*Scenario: "Update Student Mastery after Test"*

```mermaid
sequenceDiagram
    title Node 4: Dual Protocol Flow

    participant Agent as LLM Agent
    participant N4_MCP as Node 4 MCP
    participant N4_Service as Lab Service
    participant N1_GRPC as Node 1 gRPC
    participant DB as PostgreSQL

    Agent->>N4_MCP: log_activity(student="s1", result="correct", concept="calculus")
    activate N4_MCP
    
    N4_MCP->>N4_Service: process_result("s1", "correct")
    activate N4_Service

    N4_Service->>DB: INSERT into attempts
    
    rect rgb(240, 240, 255)
        note right of N4_Service: Verify Concept Dependencies
        N4_Service->>N1_GRPC: GetConcept("calculus")
        N1_GRPC-->>N4_Service: { prerequisites: ["algebra"] }
    end

    N4_Service->>N4_Service: BKT Algorithm Update (Heatmap)
    N4_Service->>DB: UPDATE heatmap SET level=0.85

    N4_Service-->>N4_MCP: Success
    deactivate N4_Service

    N4_MCP-->>Agent: "Logged and map updated"
    deactivate N4_MCP
```

## 3. 📦 Dependency & Reuse Diagram

```mermaid
classDiagram
    title Node 4: Dependency on mathesis-common

    namespace mathesis-common {
        class DBConnectionParams
        class BaseMCPServer
    }

    namespace node4_lab_node {
        class Node4MCPServer
        class HeatmapEngine
    }

    Node4MCPServer --|> BaseMCPServer : inherits
    HeatmapEngine ..> DBConnectionParams : uses
```
