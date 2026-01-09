# Node 3: Gen Node - Frontend Architecture

## 1. Information Architecture (IA)

```mermaid
graph TD
    Root[Node 3 Root] --> Generator[Content Generator]
    Root --> Review[Review Queue]
    Root --> Archive[Content Archive]
```

## 2. User Flows

### 2.1 Generating Problems with Human-in-the-Loop

```mermaid
sequenceDiagram
    actor Teacher
    participant Form as Generator Form
    participant Preview as Preview Panel
    participant API as /api/v1/generation
    
    Teacher->>Form: Config (Topic: Calculus, Diff: Hard, Count: 5)
    Teacher->>Form: Click "Generate"
    Form->>API: POST /generate (stream=true)
    loop SSE Stream
        API-->>Preview: Stream Token/Item
        Preview->>Teacher: Show Loading... then Items
    end
    Teacher->>Preview: Review Item 1
    Teacher->>Preview: Click "Edit" -> Modify Text -> Save
    Teacher->>Preview: Click "Approve All"
    Preview->>API: POST /approve
```

## 3. Component Architecture

### 3.1 Components
- `GenerationConfigForm`: Complex form with concept selectors.
- `StreamPreview`: Handles SSE/WebSocket data for real-time generation.
- `ItemEditor`: WYSIWYG editor for math content (LaTeX support).

## 4. State Management
- **Streaming State**: Custom hook to handle SSE connection and buffering.
- **Form State**: React Hook Form for validation.

## 5. Directory Structure
```
node3/
├── components/
│   ├── ConfigForm.tsx
│   ├── StreamPreview.tsx
│   └── ItemEditor.tsx
├── hooks/
│   └── useGenerationStream.ts
└── pages/
    └── GeneratorPage.tsx
```
