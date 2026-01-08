# Impact Analysis: Node 6 (School Info)

> **Usage**: Read this BEFORE modifying Node 6.

## 1. 📚 ChromaDB (RAG)
*   **If you modify Chunking Strategy (Parent-Child):**
    *   **Check `ingest_school_documents`**: Must re-index ALL documents. Old indices will be incompatible with new retrieval logic.

## 2. 🔌 MCP Tools
*   **If you modify `query_school_info`:**
    *   **Check Frontend/Agent**: Used for Q&A.
