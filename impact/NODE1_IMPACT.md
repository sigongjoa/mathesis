# Impact Analysis: Node 1 (Logic Engine)

> **Usage**: Read this BEFORE modifying Node 1.

## 1. 🕸️ Graph Schema (`app.models.knowledge`)
*   **If you modify `Concept` or `Question` node properties:**
    *   **Check Neo4j Database**: Existing data may need migration.
    *   **Check Node 1**: `GraphBuilder` class queries ("CREATE", "MATCH") must match new schema.
    *   **Check Node 5**: The Report Node **reads** this graph to visualize knowledge maps. Modifying `PREREQUISITE_OF` edge type breaks Node 5's visualization.

## 2. 🔌 MCP Tools (`app.mcp.server`)
*   **If you modify `find_concept_gap` signature:**
    *   **Check Orchestrator (Agent)**: The main agent prompts identifying this tool will fail.
    *   **Check Node 3**: Node 3 uses output of this tool to generate "Picket Problems". Changing output format breaks Node 3's generation logic.

## 3. 🔗 gRPC Services
*   **If you modify `LogicService` stub:**
    *   **Check Node 2**: May depend on concept structure for tagging.
