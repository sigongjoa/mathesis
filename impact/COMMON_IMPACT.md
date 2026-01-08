# Impact Analysis: Mathesis Common (`mathesis-common`)

> **Usage**: Read this BEFORE modifying any code in `mathesis-common`.  
> **Purpose**: Identifies which Nodes may break if you change shared logic.

## 1. 📡 BaseMCP Server (`mathesis_core.mcp.server`)
*   **If you modify `BaseMCPServer`:**
    *   **Check Node 1**: `Node1MCPServer` inherits from this. Verify `extract_concepts` tool registration.
    *   **Check Node 2**: `Node2MCPServer` inherits from this.
    *   **Check Node 5**: `Node5MCPServer` inherits from this.
    *   **Check Node 6**: `Node6MCPServer` (Planned) will inherit from this.
    *   **Verify**: Run `tests/mcp/test_server.py` to ensure tool registration and error handling wrappers still work.

## 2. 🗄️ Database Managers (`mathesis_core.db`)
*   **If you modify `Neo4jManager`:**
    *   **Check Node 1**: Uses this for `GraphBuilder`.
    *   **Check Node 5**: Uses this for `ReportGraphVisualizer`.
    *   **Critical**: Ensure connection pooling logic handles both async (Node 1) and sync usage if applicable.

*   **If you modify `HierarchicalChromaStore`:**
    *   **Check Node 6**: Heavily relies on this for RAG. Breaking embedding format triggers a full re-ingestion requirement.

## 3. 🧠 LLM Clients (`mathesis_core.llm`)
*   **If you modify `OllamaClient`:**
    *   **Check ALL Nodes**: Node 1 (Extraction), Node 2 (Analysis), Node 3 (Generation), Node 5 (Insight), Node 6 (RAG).
    *   **Critical**: Changing return format (e.g., raw text vs dict) breaks everyone.

## 4. 📤 Export Tools (`mathesis_core.export`)
*   **If you modify `PPTGeneratorAgent`:**
    *   **Check Node 3**: May use this for generating lesson summaries.
    *   **Check Node 5**: May use this for export features.
