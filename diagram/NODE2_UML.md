# Node 2 (Q-DNA) Architecture Diagrams

## 1. 🏗️ Internal Architecture (Component Diagram)
Node 2 handles Question DNA analysis. It exposes DNA extraction to Agents via MCP and allows other nodes (like Gen Node) to fetch question data via gRPC.

```mermaid
graph TD
    %% Node 2: Internal Architecture

    subgraph Interface_Layer [Interface Layer]
        direction TB
        MCP["MCP Server"]
        GRPC["gRPC Service"]
    end

    subgraph Service_Layer [Service Layer]
        direction TB
        QService["QuestionService"]
        Tagger["TaggingService"]
        Crawler["CrawlerService"]
    end

    subgraph Data_Layer [Data Layer]
        direction TB
        PG[("PostgreSQL (ltree)")]
        Vision[("Ollama (Vision)")]
    end

    MCP -->|"extract_question_dna"| QService
    MCP -->|"fetch_public_exams"| Crawler
    
    GRPC -->|"GetQuestion / AnalyzeProblem"| QService

    QService -->|"OCR & DNA Analysis"| Vision
    QService -->|"Save/Query Questions"| PG
    Tagger -->|"Fetch Tag Taxonomy"| PG
```

## 2. 🔗 Dual Protocol Sequence Diagram
*Scenario: "Analyze Problem Image and Save DNA"*

```mermaid
sequenceDiagram
    title Node 2: Dual Protocol Flow

    participant Agent as LLM Agent (Orchestrator)
    participant N2_MCP as Node 2 MCP
    participant N2_Service as Node 2 Service
    participant Common_LLM as Common LLM Client
    participant DB as PostgreSQL

    Agent->>N2_MCP: extract_question_dna(image_url="...")
    activate N2_MCP
    
    N2_MCP->>N2_Service: analyze_problem_image(url)
    activate N2_Service

    N2_Service->>Common_LLM: Vision Analysis (Image)
    Common_LLM-->>N2_Service: Text + Visual Features

    N2_Service->>DB: INSERT INTO questions (dna, tags...)
    DB-->>N2_Service: new_question_id

    N2_Service-->>N2_MCP: { dna_type: "geometry", difficulty: 0.8 }
    deactivate N2_Service

    N2_MCP-->>Agent: Result JSON
    deactivate N2_MCP

    note over N2_Service, DB: Other nodes can now access this question<br/>via gRPC GetQuestion(new_question_id)
```

## 3. 📦 Dependency & Reuse Diagram

```mermaid
classDiagram
    title Node 2: Dependency on mathesis-common

    namespace mathesis-common {
        class VisionClient
        class KoreanTokenizer
        class BaseMCPServer
    }

    namespace node2_q_dna {
        class Node2MCPServer
        class QuestionService
        class CrawlerService
    }

    Node2MCPServer --|> BaseMCPServer : inherits
    QuestionService ..> VisionClient : uses (for OCR)
    QuestionService ..> KoreanTokenizer : uses (for Tagging)
```
