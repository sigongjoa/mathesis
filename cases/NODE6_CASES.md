# Node 6 Cases (School Info)

## 📌 Overview
Node 6 handles RAG (Retrieval Augmented Generation) over crawled school documents (Hwp, PDF).

## 1. Use Cases

### UC-1: Query School Info
**Actor**: Parent / Agent
**Goal**: Ask "What is the evaluation method for Dongdo Middle School 2025?".
**Flow**:
1. Agent calls `query_school_info`.
2. Node 6 RAG retrieves chunks from "2025_Dongdo_Evaluation_Plan.pdf".
3. LLM synthesizes answer using context.
4. Returns text answer + citations (Source Document, Page).

### UC-2: Ingest New Document
**Actor**: Crawler
**Goal**: Add new school plan to index.
**Flow**:
1. Crawler downloads `file.hwp`.
2. `ingest_school_documents` tool is called.
3. File converted to text (hwp5txt/pdf).
4. Text chunked (Parent-Child) and embedded into ChromaDB.

## 2. Test Cases

### TC-1: RAG Retrieval Accuracy
*   **Input**: Query "Midterm weight". Document says "Midterm: 30%".
*   **Expected Output**:
    *   Retrieved chunks MUST contain the "30%" line.
    *   LLM Answer must state "30%".

### TC-2: HWP Parsing
*   **Input**: Standard Korean `.hwp` file from schoolinfo.go.kr.
*   **Expected Output**:
    *   Text extracted successfully (not garbage/encoding error).
    *   Tables preserved (linearized) if possible.

## 3. Edge Cases

### EC-1: Zero Retrieval (Unknown School)
*   **Scenario**: Query for a school/year not in DB.
*   **Behavior**:
    *   Distance threshold filters out irrelevant chunks.
    *   LLM responds: "I do not have information about that school yet."
    *   Should NOT hallucinate based on different schools.

### EC-2: Encoding/DRM Protected Files
*   **Scenario**: Crawler downloads a protected PDF.
*   **Behavior**:
    *   Parsing fails.
    *   Log error "File processing failed: DRM detected".
    *   Skip file and continue ingestion of others.

### EC-3: HWP Conversion Failure
*   **Scenario**: `hwp5txt` missing or fails.
*   **Behavior**:
    *   Fall back to alternate parser or log error.
    *   If using distinct HWP server, handle connection timeout.
