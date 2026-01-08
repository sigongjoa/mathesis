# Node 5 (Report Node) Architecture Diagrams

## 1. 🏗️ Internal Architecture (Component Diagram)
Node 5 generates high-quality PDF reports. It acts as a standard MCP server for generation tools and uses `mathesis-common`'s Typst wrapper.

```mermaid
graph TD
    %% Node 5: Internal Architecture

    subgraph Interface_Layer [Interface Layer]
        direction TB
        MCP["MCP Server"]
        GRPC["gRPC Service"]
    end

    subgraph Service_Layer [Service Layer]
        direction TB
        Gen["ReportGenerator"]
        Charts["ChartEngine"]
        Insight["InsightSynthesizer"]
    end

    subgraph Data_Layer [Data Layer]
        direction TB
        Typst["Typst CLI"]
        Tpl["Templates (.typ)"]
    end

    MCP -->|"generate_typst_report"| Gen
    MCP -->|"create_growth_chart"| Charts
    
    GRPC -->|"GenerateReportAsync"| Gen

    Gen -->|"Embed Images"| Charts
    Gen -->|"Add AI Comments"| Insight
    Gen -->|"Compile PDF"| Typst
    Gen -->|"Load Layout"| Tpl
```

## 2. 🔗 Dual Protocol Sequence Diagram
*Scenario: "Generate Comprehensive Report"*

```mermaid
sequenceDiagram
    title Node 5: Dual Protocol Flow

    participant Agent as LLM Agent
    participant N5_MCP as Node 5 MCP
    participant N5_Service as Report Service
    participant N1_GRPC as Node 1 gRPC
    participant Disk

    Agent->>N5_MCP: generate_typst_report(student_id="s1")
    activate N5_MCP
    
    N5_MCP->>N5_Service: compile_report("s1")
    activate N5_Service

    rect rgb(240, 240, 255)
        note right of N5_Service: Fetch Knowledge Map Image
        N5_Service->>N1_GRPC: QueryGraphRAG(format="svg")
        N1_GRPC-->>N5_Service: SVG Data
    end

    N5_Service->>N5_Service: Render Typst Source
    N5_Service->>Disk: Save .typ file
    N5_Service->>Disk: Run 'typst compile' -> .pdf
    
    N5_Service-->>N5_MCP: PDF Path
    deactivate N5_Service

    N5_MCP-->>Agent: "/reports/s1_final.pdf"
    deactivate N5_MCP
```

## 3. 📦 Dependency & Reuse Diagram

```mermaid
classDiagram
    title Node 5: Dependency on mathesis-common

    namespace mathesis-common {
        class TypstWrapper
        class PDFUtils
        class BaseMCPServer
    }

    namespace node5_report_node {
        class Node5MCPServer
        class ReportGenerator
    }

    Node5MCPServer --|> BaseMCPServer : inherits
    ReportGenerator ..> TypstWrapper : uses (Core Logic)
    ReportGenerator ..> PDFUtils : uses
```
