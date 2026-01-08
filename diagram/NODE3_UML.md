# Node 3 (Gen Node) Architecture Diagrams

## 1. 🏗️ Internal Architecture (Component Diagram)
Node 3 is responsible for generative content (problems, explanations). It heavily relies on LLM prompts and potentially Python computation tools (SymPy) for verification.

```mermaid
graph TD
    %% Node 3: Internal Architecture

    subgraph Interface_Layer [Interface Layer]
        direction TB
        MCP["MCP Server"]
        GRPC["gRPC Service"]
    end

    subgraph Service_Layer [Service Layer]
        direction TB
        Gen["ProblemGenerator"]
        Variator["MathVariator"]
        Renderer["SolutionRenderer"]
    end

    subgraph Data_Layer [Data Layer]
        direction TB
        LLM[("Ollama (LLM)")]
        Tpl["LaTeX Templates"]
    end

    MCP -->|"generate_picket_problem"| Gen
    MCP -->|"create_variant"| Variator
    
    GRPC -->|"GenerateProblemBatch"| Gen

    Gen -->|"Few-shot Prompting"| LLM
    Gen -->|"Load Context"| Tpl
    Renderer -->|"Render LaTeX"| Tpl
```

## 2. 🔗 Dual Protocol Sequence Diagram
*Scenario: "Generate Personalized Problem"*

```mermaid
sequenceDiagram
    title Node 3: Dual Protocol Flow

    participant Agent as LLM Agent
    participant N3_MCP as Node 3 MCP
    participant N3_Service as Gen Service
    participant N2_GRPC as Node 2 gRPC
    participant LLM

    Agent->>N3_MCP: generate_picket_problem(target_concept="vector")
    activate N3_MCP
    
    N3_MCP->>N3_Service: create_problem("vector")
    activate N3_Service

    rect rgb(240, 240, 255)
        note right of N3_Service: Fetch Reference Problem
        N3_Service->>N2_GRPC: FindSimilarProblem(concept="vector")
        N2_GRPC-->>N3_Service: Reference Question Data
    end

    N3_Service->>LLM: Generate Variant(Reference)
    LLM-->>N3_Service: New Problem LaTeX

    N3_Service-->>N3_MCP: { problem_tex: "...", solution: "..." }
    deactivate N3_Service

    N3_MCP-->>Agent: Final Output
    deactivate N3_MCP
```

## 3. 📦 Dependency & Reuse Diagram

```mermaid
classDiagram
    title Node 3: Dependency on mathesis-common

    namespace mathesis-common {
        class OllamaClient
        class PromptTemplates
        class BaseMCPServer
    }

    namespace node3_gen_node {
        class Node3MCPServer
        class ProblemGenerator
    }

    Node3MCPServer --|> BaseMCPServer : inherits
    ProblemGenerator ..> OllamaClient : uses
    ProblemGenerator ..> PromptTemplates : imports
```
