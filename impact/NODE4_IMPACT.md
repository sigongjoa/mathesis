# Impact Analysis: Node 4 (Lab Node)

> **Usage**: Read this BEFORE modifying Node 4.

## 1. 📊 BKT / Heatmap Schema
*   **If you modify `student_heatmap` table:**
    *   **Check Node 5**: Node 5 **Directly Reads** (or requests) this data to generate PDF Reports. Changing column names (e.g., `mastery_level` -> `score`) crashes Node 5's report generation.
    *   **Check Node 1**: Node 1 uses this mastery data to find gaps. Formart change breaks `find_concept_gap`.

## 2. 🔌 MCP Tools
*   **If you modify `log_activity`:**
    *   **Check Frontend**: Main dashboard relies on this for progress tracking.
