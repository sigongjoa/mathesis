# Node 6 (School Info) Architecture Diagrams

## 1. 🏗️ Internal Architecture (Component Diagram)
Node 6 provides School Information via RAG. It crawls data, builds an enhanced vector index, and serves queries.

```mermaid
graph TD
    %% Node 6: Internal Architecture

    subgraph Interface_Layer [Interface Layer]
        direction TB
        MCP["MCP Server"]
        GRPC["gRPC Service"]
    end

    subgraph Service_Layer [Service Layer]
        direction TB
        RAG["IntegratedRAGPipeline"]
        Crawler["SchoolInfoCrawler"]
        Transformer["EnhancedJSONGenerator"]
    end

    subgraph Data_Layer [Data Layer]
        direction TB
        VectorDB[("ChromaDB (Hierarchical)")]
        LLM[("Ollama")]
        Storage["PDF/HWP Files"]
    end

    MCP -->|"query_school_info"| RAG
    MCP -->|"ingest_school_documents"| Crawler
    
    GRPC -->|"QueryRAG"| RAG

    RAG -->|"Hybrid Search"| VectorDB
    RAG -->|"Answer Synthesis"| LLM
    Crawler -->|"Download"| Storage
    Transformer -->|"Read & Parse"| Storage
```

## 2. 🔗 Dual Protocol Sequence Diagram
*Scenario: "Query School Evaluation Plan"*

```mermaid
sequenceDiagram
    title Node 6: Dual Protocol Flow

    participant Agent as LLM Agent
    participant N6_MCP as Node 6 MCP
    participant N6_Service as RAG Service
    participant Common_Chroma as Hybrid Store
    participant Common_Crawler as School Crawler

    Agent->>N6_MCP: query_school_info("동도중 2025 평가계획")
    activate N6_MCP
    
    N6_MCP->>N6_Service: query("동도중 2025 평가계획")
    activate N6_Service

    note right of N6_Service: Note: Deep Reuse of Common
    N6_Service->>Common_Chroma: query_with_parent_context()
    Common_Chroma-->>N6_Service: Context Chunks

    N6_Service->>N6_Service: Synthesize Answer (Ollama)
    
    N6_Service-->>N6_MCP: "중간고사 30%, 수행평가 70%..."
    deactivate N6_Service

    N6_MCP-->>Agent: Answer Text
    deactivate N6_MCP
```

## 3. 📦 Dependency & Reuse Diagram

```mermaid
classDiagram
    title Node 6: Dependency on mathesis-common

    namespace mathesis-common {
        class SchoolInfoCrawler
        class HierarchicalChromaStore
        class EnhancedJSONGenerator
        class BaseMCPServer
    }

    namespace node6_school_info {
        class Node6MCPServer
        class IntegratedRAGPipeline
    }

    Node6MCPServer --|> BaseMCPServer : inherits
    IntegratedRAGPipeline --|> HierarchicalChromaStore : uses
    IntegratedRAGPipeline --|> SchoolInfoCrawler : uses
    IntegratedRAGPipeline --|> EnhancedJSONGenerator : uses
```
