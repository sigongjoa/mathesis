# Common Library Cases (`mathesis-common`)

## 📌 Overview
`mathesis-common` provides the shared infrastructure (MCP Base, DB Clients, Export Tools) used by all Nodes. While not a standalone service, its reliability is critical.

## 1. Use Cases

### UC-1: Handle Tool Call (BaseMCP)
**Actor**: Node Server (e.g., Node 1)
**Goal**: dispatch a request to the correct internal method.
**Flow**:
1. External Agent calls `handle_tool_call("find_concept_gap", args)`.
2. `BaseMCPServer` validates tool existence.
3. Logs the request payload.
4. Executes `node1.find_concept_gap(**args)`.
5. Logs success/failure and returns result.

### UC-2: Generate PPT (PPT Agent)
**Actor**: Node 3 / Student
**Goal**: Create a presentation summary.
**Flow**:
1. User calls `PPTGeneratorAgent.generate_presentation("Triangles")`.
2. Agent calls LLM to structure slides (Title, Bullets, Notes).
3. `python-pptx` renders the JSON to a `.pptx` file.
4. Returns file path.

### UC-3: Neo4j Connection Pooling
**Actor**: Node 1 & Node 5
**Goal**: Reuse DB connections to prevent overhead.
**Flow**:
1. Node 1 initializes `Neo4jManager`.
2. Node 5 initializes `Neo4jManager`.
3. Both receive the *same* singleton instance (or shared driver pool).
4. Connection handshake occurs only once per process (if shared) or efficiently per process.

## 2. Test Cases

### TC-1: BaseMCP Error Handling
*   **Input**: Call `handle_tool_call` with a tool name that doesn't exist.
*   **Expected Output**:
    *   Raises `ValueError` or `ToolExecutionError`.
    *   Logs `WARNING: Tool not found`.
    *   Does NOT crash the python process.

### TC-2: PPT Structure Generation
*   **Input**: Topic="Quantum Physics", Slides=3.
*   **Expected Output**:
    *   LLM returns JSON list of length 3.
    *   Keys `title`, `content` exist in each item.
    *   `python-pptx` generates file without raising `AttributeError`.

### TC-3: Tokenizer Standard
*   **Input**: "수학의 정석".
*   **Expected Output**:
    *   `KoreanTokenizer.tokenize()` returns same list `["수학", "정석"]` across Node 2 and Node 6.
    *   Ensures search consistency.

## 3. Edge Cases

### EC-1: LLM Malformed JSON (PPT Agent)
*   **Scenario**: LLM returns markdown text `Here is your JSON: ...` instead of pure JSON.
*   **Behavior**:
    *   `PPTGeneratorAgent` cleanly strips markdown fences.
    *   If parsing still fails, logs error and returns a "Fallback Slide" explaining the failure, rather than crashing code.

### EC-2: Database Down (Neo4jManager)
*   **Scenario**: Neo4j container is stopped.
*   **Behavior**:
    *   `initialize()` logs "Connection Failed".
    *   `get_driver()` returns `None`.
    *   Consumer (Node 1) enters "Offline/Mock Mode" gracefully without `ConnectionRefusedError` loop.
