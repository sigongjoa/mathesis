# Node 5 Cases (Report Node)

## 📌 Overview
Node 5 orchestrates file generation (Typst PDFs, Charts, PPTs) based on aggregated data.

## 1. Use Cases

### UC-1: Generate Diagnostic PDF
**Actor**: Agent
**Goal**: Create a physical PDF file for a student.
**Flow**:
1. Agent calls `generate_typst_report` with all analysis data.
2. Node 5 generates Charts (matplotlib/plotly) -> saves as images.
3. Node 5 renders Typst template (`report.typ`) injecting data & images.
4. Compiles to PDF and returns file path.

### UC-2: Synthesize Insights
**Actor**: Agent (Internal)
**Goal**: Add AI commentary to report.
**Flow**:
1. Agent calls `synthesize_diagnostic_insight`.
2. Node 5 prompts LLM with student stats.
3. Returns paragraph: "Student shows strong algebra skills but struggles with geometry spatial reasoning..."

## 2. Test Cases

### TC-1: Typst Compilation
*   **Input**: Valid data dictionary, valid template.
*   **Expected Output**:
    *   Function returns path ending in `.pdf`.
    *   File exists on disk.
    *   File size > 0 bytes.

### TC-2: Chart Generation
*   **Input**: `create_growth_chart` with `[0.1, 0.2, 0.5, 0.8]`.
*   **Expected Output**:
    *   Saves `.png` or `.svg` file.
    *   Image is legible (labels present).

## 3. Edge Cases

### EC-1: Missing Typst Binary
*   **Scenario**: Server environment lacks `typst` CLI installed.
*   **Behavior**:
    *   `TypstWrapper` raises `EnvironmentError` or `ExecutableNotFound`.
    *   Caught by BaseMCP -> Returns clear error to Agent "Typst not installed".

### EC-2: Data Missing for Template
*   **Scenario**: Template expects `student_score` but data is `None`.
*   **Behavior**:
    *   Typst compilation usually fails with valid error.
    *   Wrapper should catch this and log `stderr` from Typst process for debugging.
    *   Ideally, use default/empty placeholders instead of crashing.
