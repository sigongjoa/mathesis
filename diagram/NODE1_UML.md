# Node 1 (Logic Engine) Architecture Diagrams

## 1. 🏗️ Internal Architecture (Component Diagram)
This diagram illustrates how the **MCP Server** and **gRPC Service** interfaces co-exist within Node 1, sharing the same underlying `GraphBuilder` and `ConceptExtractor` logic.

```mermaid
graph TD
    %% Internal Architecture (Shared Logic)

    subgraph Interface_Layer [Interface Layer]
        direction TB
        MCP["MCP Server (server.py)"]
        GRPC["gRPC Service (services.py)"]
    end

    subgraph Service_Layer [Service Layer]
        direction TB
        Graph["GraphBuilder"]
        Extractor["ConceptExtractor"]
    end

    subgraph Data_Layer [Data Layer]
        direction TB
        DB[("Neo4j")]
        LLM[("Ollama (LLM)")]
    end

    MCP -->|"calls find_concept_gap()"| Graph
    MCP -->|"calls extract_concepts()"| Extractor
    
    GRPC -->|"calls get_prerequisites()"| Graph
    GRPC -->|"calls extract_from_pdf()"| Extractor

    Graph -->|"Cypher Queries"| DB
    Extractor -->|"Prompt Generation"| LLM

    %% Notes simulated as nodes or ignored if complex, using simple text nodes for now or ignoring to prevent clutter
    %% note right of MCP: Exposes tools for Agent
    %% note left of GRPC: Exposes API for other Nodes
```

## 2. 🔗 Dual Protocol Sequence Diagram
This diagram shows the flow where the Orchestrator uses **MCP** to request a gap analysis, while Node 1 internally uses **gRPC** (hypothetically, if it needed to fetch data from another node, e.g., checking if the concept exists in a global registry or ensuring student mastery from Node 4).

*Scenario: "Identify Concept Gaps for Student"*

```mermaid
sequenceDiagram
    title Node 1: Dual Protocol Flow (MCP Entry -> Internal gRPC)

    participant Agent as LLM Agent (Orchestrator)
    participant N1_MCP as Node 1 MCP
    participant N1_Service as Node 1 Logic
    participant N4_GRPC as Node 4 gRPC (Lab)
    participant Neo4j

    Agent->>N1_MCP: find_concept_gap(student_id="s1", concept="calculus")
    activate N1_MCP
    
    N1_MCP->>N1_Service: find_knowledge_gaps("s1", "calculus")
    activate N1_Service

    rect rgb(240, 240, 255)
        note right of N1_Service: Internal gRPC Call
        N1_Service->>N4_GRPC: GetStudentMastery(student_id="s1")
        activate N4_GRPC
        N4_GRPC-->>N1_Service: { "algebra": 0.8, "calculus": 0.2 }
        deactivate N4_GRPC
    end

    N1_Service->>Neo4j: MATCH (c:Concept {name: "calculus"})-[:PREREQ]->(p)
    Neo4j-->>N1_Service: Prerequisites: ["algebra", "limits"]

    N1_Service->>N1_Service: Compare Mastery vs Prereqs
    N1_Service-->>N1_MCP: Gap Found: "limits" (Mastery: 0.0)
    deactivate N1_Service

    N1_MCP-->>Agent: { "gaps": [{"concept": "limits", "priority": "high"}] }
    deactivate N1_MCP
```

## 3. 📦 Dependency & Reuse Diagram
This diagram highlights how Node 1 relies on `mathesis-common` for core utilities while maintaining its own specific implementations.

```mermaid
classDiagram
    title Node 1: Dependency on mathesis-common

    namespace mathesis-common {
        class OllamaClient
        class BaseMCPServer
        class ProtoDefinitions
    }

    namespace node1_logic_engine {
        class Node1MCPServer
        class ConceptExtractor
        class GraphBuilder
    }

    Node1MCPServer --|> BaseMCPServer : inherits
    ConceptExtractor ..> OllamaClient : uses
    Node1MCPServer ..> ProtoDefinitions : implements (via gRPC)
    
    note for ProtoDefinitions "Defines Concept & Service messages"
```
