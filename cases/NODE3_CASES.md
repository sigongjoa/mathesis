# Node 3 Cases (Gen Node)

## 📌 Overview
Node 3 generates new math problems, variants, and step-by-step explanations using LLMs and templates.

## 1. Use Cases

### UC-1: Generate Picket Problem (Weakness Targeting)
**Actor**: Student via Agent
**Goal**: Create a custom problem to target a specific gap.
**Flow**:
1. Agent calls `generate_picket_problem` with `gap_concept="Chain Rule"`.
2. Node 3 finds a "seed" problem from Node 2 (via gRPC).
3. Node 3 asks LLM to "Create a variant of this seed problem involving Chain Rule".
4. Returns LaTeX of new problem + solution.

### UC-2: Create Step-by-Step Explanation
**Actor**: Student
**Goal**: Get a Socratic tutor-style explanation.
**Flow**:
1. Agent calls `create_explanation_step` with `problem_tex`, `current_step`.
2. Node 3 generates the *next single logical step* and a hint.
3. Returns content.

## 2. Test Cases

### TC-1: LaTeX Validity
*   **Input**: Generate problem for "Quadratic Roots".
*   **Expected Output**:
    *   String starting with `$ ` or `$$`.
    *   Valid LaTeX syntax (can be compiled by KaTeX/MathJax without error).

### TC-2: Variant distinctness
*   **Input**: Seed problem "x^2 - 4 = 0".
*   **Expected Output**:
    *   Variant is NOT identical to seed (e.g., "x^2 - 9 = 0" or "2x^2 - 8 = 0").
    *   Logic/Method required remains same.

## 3. Edge Cases

### EC-1: Hallucination (Unsolvable Problem)
*   **Scenario**: LLM generates "x^2 + 1 = 0, find real roots".
*   **Behavior**:
    *   Node 3 (ideally) uses a `Solver` (SymPy) to verify solution existence.
    *   If verification fails, retry generation (up to 3 times).
    *   If all fail, return generic fallback problem or error.

### EC-2: Token Limit Exceeded
*   **Scenario**: `create_explanation_step` context includes entire chat history (too long).
*   **Behavior**:
    *   Ollama/API returns context length error.
    *   Node 3 should truncate context or summarize before prompting.
