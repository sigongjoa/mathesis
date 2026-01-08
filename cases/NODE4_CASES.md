# Node 4 Cases (Lab Node)

## 📌 Overview
Node 4 tracks student interactions, manages user sessions, and computes real-time analytics (Heatmaps, BKT).

## 1. Use Cases

### UC-1: Log Activity & Update Heatmap
**Actor**: Student
**Goal**: Record a problem attempt and update mastery.
**Flow**:
1. Agent calls `log_activity` with `student_id`, `result={correct: false}`, `concept`.
2. Node 4 saves raw log to `activity_logs`.
3. Node 4 triggers BKT (Bayesian Knowledge Tracing) engine.
4. Updates `student_heatmap` table with new probability.

### UC-2: Get Failure Patterns
**Actor**: Teacher / Report Agent
**Goal**: summaries wrong answer types.
**Flow**:
1. Agent calls `get_failure_pattern(student_id)`.
2. Node 4 aggregates recent logs.
3. Returns `{"calculation_error": 60%, "conceptual_error": 40%}`.

## 2. Test Cases

### TC-1: BKT Update Logic
*   **Input**: Prior Mastery=0.5. Result=Correct.
*   **Expected Output**:
    *   New Mastery > 0.5 (e.g., 0.65).
    *   DB updated correctly.

### TC-2: Heatmap Retrieval
*   **Input**: `update_student_heatmap` call.
*   **Expected Output**:
    *   Returns list of `{concept, level, status}`.
    *   Color codes (red/green/yellow) match levels.

## 3. Edge Cases

### EC-1: Cold Start (New Student)
*   **Scenario**: `log_activity` for student with no prior records.
*   **Behavior**:
    *   BKT initializes with default prior (e.g., 0.3).
    *   Creates new entries in `student_heatmap`.
    *   Do NOT fail with "User not found".

### EC-2: Concurrent Updates
*   **Scenario**: Student submits 2 problems rapidly (Simultaneous requests).
*   **Behavior**:
    *   DB Transaction isolation ensures no lost updates.
    *   Final mastery reflects both attempts.
