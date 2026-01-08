# Mathesis Common Library (`mathesis-common`)

> **Role**: Shared Utility Toolkit  
> **Type**: Python Library (not a standalone server)  
> **Status**: Active / Core Infrastructure

## 📋 Overview
`mathesis-common` contains the foundational building blocks for all Mathesis nodes. By centralizing infrastructure code here, we ensure consistency in:
- **MCP Communication**: Standardized server implementation with logging and error handling.
- **Database Connections**: Singleton managers for Neo4j, PostgreSQL, and ChromaDB.
- **LLM Interaction**: Unified clients for Ollama and Vision models.
- **Export Capabilities**: Shared engines for PDF (Typst) and PPT generation.

## 📦 Key Modules

### 1. 📡 MCP Framework (`mathesis_core.mcp`)
Provides the backbone for all Node servers.
*   **`BaseMCPServer`**: A robust base class that handles:
    *   Automatic tool registration via reflection.
    *   Standardized request/response logging.
    *   Global error handling (catches exceptions and formats them for the Agent).
    *   Built-in health checks (`ping`, `get_server_info`).

### 2. 🗄️ Database Managers (`mathesis_core.db`)
Centralized connection pooling and management.
*   **`Neo4jManager`**: Singleton connector for Node 1 (Logic) and Node 5 (Report). Handles driver lifecycle.
*   **`HierarchicalChromaStore`**: Advanced RAG storage supporting parent-child document retrieval (used by Node 6).
*   **`KoreanTokenizer`**: Shared tokenization logic for search and tagging.

### 3. 🧠 LLM & Vision (`mathesis_core.llm`)
*   **`OllamaClient`**: Standard wrapper for local LLM inference (supports automatic retries and JSON parsing).
*   **`VisionClient`**: Specialized client for processing images (OCR, diagram analysis) used by Q-DNA (Node 2).

### 4. 📤 Export Tools (`mathesis_core.export`)
*   **`TypstWrapper`**: Compiles Typst markup into professional PDF reports (Node 5).
*   **`PPTGeneratorAgent`**: **[New]** AI Agent that generates PowerPoint presentations from a topic string using LLM for structure and `python-pptx` for rendering.

### 5. 🕷️ Crawlers (`mathesis_core.crawlers`)
*   **`SchoolInfoCrawler`**: Playwright-based crawler for `schoolinfo.go.kr` (Node 6).
*   **`BaseCrawler`**: Abstract base class defining polite crawling behaviors (delays, user-agent rotation).

## 🚀 How to Use

### Installing
Each Node lists this library as a dependency in `requirements.txt`:
```bash
pip install -e ../../mathesis-common
```

### Implementing a New Node
Inherit from `BaseMCPServer` to get free logging and error handling:

```python
from mathesis_core.mcp.server import BaseMCPServer

class MyNodeServer(BaseMCPServer):
    def __init__(self):
        super().__init__("my-node", "1.0.0")
    
    async def my_tool(self, arg1: str):
        # ... logic ...
        return {"result": "success"}
```

### Generating a PPT
```python
from mathesis_core.export.ppt_agent import PPTGeneratorAgent

agent = PPTGeneratorAgent()
path = await agent.generate_presentation(
    topic="History of Mathematics", 
    num_slides=5
)
print(f"Saved at {path}")
```
