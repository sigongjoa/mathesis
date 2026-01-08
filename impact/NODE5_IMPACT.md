# Impact Analysis: Node 5 (Report Node)

> **Usage**: Read this BEFORE modifying Node 5.

## 1. 📄 Typst Templates (`report.typ`)
*   **If you modify `.typ` files:**
    *   **Check `typst_wrapper.py`**: The python code injects variables (`#student_name`). Removing/Renaming a variable in Typst without updating Python code yields Compilation Error.

## 2. 🔌 MCP Tools
*   **If you modify `generate_typst_report`:**
    *   **Check Orchestrator**: This is the final output step. Failure here means no deliverable to user.
