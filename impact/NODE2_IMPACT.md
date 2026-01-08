# Impact Analysis: Node 2 (Q-DNA)

> **Usage**: Read this BEFORE modifying Node 2.

## 1. 🏷️ Tagging System & Embeddings
*   **If you modify Embedding Model (e.g., `nomic-embed-text` -> `openai`):**
    *   **Check PostgreSQL (`questions` table)**: Vector column dimension size (768 vs 1536) must change.
    *   **Check Node 3**: Node 3 calls `GetSimilarProblems` via gRPC. Changing similarity scorings effects Node 3's "Variant Generation" quality.

## 2. 🔌 MCP Tools
*   **If you modify `extract_question_dna`:**
    *   **Check Node 5**: Node 5 aggregates question difficulty stats from here.

## 3. 🔗 gRPC Services
*   **If you modify `QDNAService`:**
    *   **Check Node 3**: Node 3 is the primary consumer (Seed Problem Retrieval).
