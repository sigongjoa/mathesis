# Node 2 Cases (Q-DNA)

## 📌 Overview
Node 2 manages the "Question DNA" (metadata, difficulty, vector embeddings) and provides problem search/tagging services.

## 1. Use Cases

### UC-1: Analyze Problem Content
**Actor**: Teacher / Admin
**Goal**: Get AI analysis of a new problem image.
**Flow**:
1. Agent calls `extract_question_dna` with `image_url`.
2. Node 2 uses Vision model (Ollama) to OCR and describe the problem.
3. Node 2 classifies "DNA Type" (e.g., "Calculation", "Conceptual") and Difficulty.
4. Returns complete metadata JSON.

### UC-2: Find Similar Problems
**Actor**: Student (Adaptive Learning)
**Goal**: Practice more problems like the one just failed.
**Flow**:
1. Agent calls `find_similar_dna_problems` with `reference_dna` or `question_id`.
2. Node 2 performs Vector Search (Cosine Similarity) on `questions` table.
3. Returns Top-5 most similar questions.

### UC-3: Crawl Public Exams
**Actor**: System Cron / Admin
**Goal**: Populate DB with latest CSAT/Mock exams.
**Flow**:
1. Agent calls `fetch_public_exams` with `source="CSAT"`, `year="2025"`.
2. Node 2 downloads PDF, parses items, and auto-ingests them.
3. Returns count of successfully ingested questions.

## 2. Test Cases

### TC-1: Embedding Generation
*   **Input**: Text "Calculate the limit of x^2 as x approaches 0."
*   **Expected Output**:
    *   Vector of size (e.g., 768 or 1536).
    *   Not all zeros.

### TC-2: Tag Recommendation
*   **Input**: Problem text about "Derivatives of trigonometric functions".
*   **Expected Output**:
    *   Tags include "Calculus", "Differentiation", "Trigonometry".
    *   Confidence scores > 0.5.

## 3. Edge Cases

### EC-1: Blurry/Empty Image
*   **Scenario**: `extract_question_dna` called with unreadable image.
*   **Behavior**:
    *   Vision model returns low confidence or "Cannot read text".
    *   Tool raises `ToolExecutionError` with clear message "OCR Failed: Image unclear".

### EC-2: SQL Injection in Tags
*   **Scenario**: `find_similar_dna_problems` called with malicious string.
*   **Behavior**:
    *   `psycopg2` / `asyncpg` parameter binding prevents injection.
    *   Search returns 0 results cleanly.

### EC-3: Vector Dimension Mismatch
*   **Scenario**: System upgrades embedding model (768 -> 1536) but DB has old vectors.
*   **Behavior**:
    *   Search query fails with DB error.
    *   System log should strictly catch this. (Requires migration script logic).
