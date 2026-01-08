# Impact Analysis: Node 3 (Gen Node)

> **Usage**: Read this BEFORE modifying Node 3.

## 1. 📝 Generation Logic (`ProblemGenerator`)
*   **If you modify Prompt Templates:**
    *   **Check Quality**: No direct code breakage, but "Math Validity" might drop.
    *   **Check Node 4**: Node 4 logs attempts on these problems. If problem format (JSON/LaTeX) changes, Node 4's logging UI might break.

## 2. 🔌 MCP Tools
*   **If you modify `generate_problem_variant`:**
    *   **Check Frontend/Agent**: The immediate consumer is the user interface requesting practice problems.
