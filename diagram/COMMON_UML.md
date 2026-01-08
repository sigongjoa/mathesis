# Mathesis Common Library Diagrams

## 1. 🏗️ Internal Components (Library Diagram)
`mathesis-common` is a shared library providing standardized utilities for all nodes. It is **not** a server itself, but a toolkit imported by Nodes.

```mermaid
graph TD
    %% Internal Components of mathesis-common

    subgraph Mathesis_Core [mathesis_core]
        direction TB

        subgraph LLM_Module [llm]
            Ollama["OllamaClient"]
            Parsers["LLMJSONParser"]
            Decorators["@retry_llm_call"]
        end

        subgraph DB_Module [db]
            Chroma["HierarchicalChromaStore"]
            Neo4j["Neo4jClient (Stub)"]
            Tokenizer["KoreanTokenizer"]
        end

        subgraph Crawlers_Module [crawlers]
            School["SchoolInfoCrawler"]
            BaseCrawler["BaseCrawler"]
        end

        subgraph Export_Module [export]
            Typst["TypstWrapper"]
            PDF["PDFUtils"]
            PPT["PPTGeneratorAgent"]
        end

        subgraph MCP_Module [mcp]
            BaseServer["BaseMCPServer"]
            Registry["ToolRegistry"]
        end

        subgraph gRPC_Module [grpc]
            Protos["Common Protos"]
            Client["GRPCClientBase"]
        end
    end

    %% Dependency Internal flow (Logical)
    Chroma -->|"uses"| Ollama
    Chroma -->|"uses"| Tokenizer
    School -->|"extends"| BaseCrawler
    BaseServer -->|"uses"| Registry
```

## 2. 🔌 Service Usage Diagram (Flowchart)
This diagram illustrates how a typical Node (e.g., Node 6) imports and utilizes multiple services from `mathesis-common` to build its logic.

*Example: Node 6 utilizing Common Services*

```mermaid
graph TD
    %% Usage Flow

    subgraph Node_6 [Node 6 School Info]
        Pipeline["IntegratedRAGPipeline"]
        Server["Node6MCPServer"]
    end

    subgraph Common_Lib [mathesis-common]
        Common_Crawler["SchoolInfoCrawler"]
        Common_Chroma["HierarchicalChromaStore"]
        Common_Ollama["OllamaClient"]
        Common_Base["BaseMCPServer"]
    end

    Server -- "inherits" --> Common_Base
    Pipeline -- "imports & instantiates" --> Common_Crawler
    Pipeline -- "imports & instantiates" --> Common_Chroma
    Pipeline -- "imports & instantiates" --> Common_Ollama

    Common_Chroma -- "embeds text via" --> Common_Ollama
    
    note_usage[/"Shared Logic Reuse"/]
    note_usage -.- Pipeline
```

## 3. 📦 External Dependency Diagram
This diagram shows the external Python libraries and Systems that `mathesis-common` bridges to.

```mermaid
graph LR
    %% External Dependencies

    subgraph Common [mathesis-common]
        Core["Core Modules"]
    end

    subgraph External_Systems [External Systems]
        Ollama_Sys[("Ollama (Local LLM)")]
        Chroma_Sys[("ChromaDB (Vector)")]
        School_Web["schoolinfo.go.kr"]
    end

    subgraph PyPI_Libraries [PyPI Libraries]
        Langchain["langchain"]
        Pydantic["pydantic"]
        GRPC_Lib["grpcio"]
        Playwright["playwright"]
        Typst_Bin["typst-cli"]
    end

    Core -->|"HTTP API"| Ollama_Sys
    Core -->|"Native Client"| Chroma_Sys
    Core -->|"Headless Browser"| School_Web

    Core -->|"wraps"| Langchain
    Core -->|"models"| Pydantic
    Core -->|"transport"| GRPC_Lib
    Core -->|"crawling"| Playwright
    Core -->|"subprocess"| Typst_Bin
```
