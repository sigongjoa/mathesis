# Node 1 Cases (Logic Engine)

## 📌 Overview
Node 1 is responsible for managing the educational knowledge graph, concept prerequisites, and identifying knowledge gaps.

## 1. Use Cases (Success Scenarios)

### UC-1: Identify Concept Gaps
**Actor**: LLM Agent (Orchestrator)
**Goal**: Find what a student needs to study before tackling a target concept.
**Preconditions**: Student mastery data exists (or is passed).
**Flow**:
1. Agent calls `find_concept_gap` with `target_concept="Calculus"` and `student_mastery={"Algebra": 0.9, "Limits": 0.2}`.
2. Node 1 queries Graph for prerequisites of "Calculus" -> ["Algebra", "Limits"].
3. Computes gaps: "Limits" mastery (0.2) < Threshold (0.7).
4. Returns "Limits" as a High Priority gap.

### UC-2: Visualize Knowledge Map
**Actor**: Frontend / Teacher
**Goal**: View a subgraph of concepts for a specific topic.
**Flow**:
1. Agent calls `visualize_knowledge_map` with `topic_filter="Geometry"`.
2. Node 1 queries Graph for concepts/edges containing "Geometry".
3. Returns JSON with Nodes (Concepts) and Edges (Prerequisites).

### UC-3: Extract Concepts from Text
**Actor**: Content Ingest Pipeline
**Goal**: Convert raw textbook text into structured graph nodes.
**Flow**:
1. Agent calls `extract_concepts` with raw text from a PDF section.
2. Node 1 uses LLM (Ollama) to parse entities and relationships.
3. Returns structured JSON (Concept Name, Definition, Related Concepts).

## 2. Test Cases (Verification)

### TC-1: Gap Analysis Implementation
*   **Input**: `target="C"`, Graph: `A->B->C`, User Mastery: `{A: 1.0, B: 0.1}`
*   **Expected Output**:
    *   `gaps` list contains "B".
    *   `gap_size` for B should be approx `0.7 - 0.1 = 0.6`.
    *   `priority` should be "high".

### TC-2: Circular Dependency Check
*   **Input**: Try to add relationship `A -> B` where path `B -> ... -> A` already exists.
*   **Expected Output**:
    *   `EducationalGraphBuilder` should detect cycle.
    *   Log warning or raise `CircularDependencyError`.
    *   Relationship should NOT be created.

### TC-3: MCP Tool Registration
*   **Input**: Start `Node1MCPServer`.
*   **Expected Output**:
    *   `get_server_info()` returns tools list containing `['extract_concepts', 'find_concept_gap', 'visualize_knowledge_map']`.

## 3. Edge Cases (Failure Handling)

### EC-1: Non-Existent Concept
*   **Scenario**: Requesting gaps for `target_concept="UnicornMath"`.
*   **Behavior**:
    *   Graph query returns empty.
    *   Tool should return distinct error or empty gap list with warning "Concept not found".
    *   Should NOT crash server.

### EC-2: Disconnected Graph
*   **Scenario**: Target concept exists but has no prerequisites defined.
*   **Behavior**:
    *   `prerequisites` list is empty.
    *   `gaps` list is empty (Logic: No prerequisites = No gaps).
    *   Return valid JSON with empty lists.

### EC-3: Massive Graph Visualization
*   **Scenario**: Calling `visualize_knowledge_map` without filter on a graph with 10k nodes.
*   **Behavior**:
    *   Query should have `LIMIT` (e.g., 100 or 500) to prevent OOM.
    *   Return partial subgraph with "truncated: true" flag context (if implemented) or just top N nodes.
