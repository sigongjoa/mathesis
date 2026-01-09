# Node 5: Report Node - Frontend Architecture

## 1. Information Architecture (IA)

```mermaid
graph TD
    Root[Node 5 Root] --> Builder[Report Builder]
    Root --> Archive[Report Archive]
    Root --> Scheduler[Scheduled Reports]
```

## 2. User Flows

### 2.1 Generating a School Report

```mermaid
sequenceDiagram
    actor Teacher
    participant Builder as Report Builder
    participant API as /api/v1/export
    participant PDF as PDF Service
    
    Teacher->>Builder: Select "Monthly Stats" Template
    Teacher->>Builder: Select Date Range & Class
    Teacher->>Builder: Click "Preview"
    Builder->>API: POST /preview
    API->>PDF: Generate Preview (Typst)
    PDF-->>Builder: Rendered PDF Blob
    Builder->>Teacher: Show PDF Preview
    Teacher->>Builder: Click "Finalize & Send"
    Builder->>API: POST /send
```

## 3. Component Architecture

### 3.1 Components
- `ReportDesigner`: Drag-and-drop or section toggler for templates.
- `PDFViewer`: Canvas-based PDF renderer (e.g., pdf.js).
- `SchedulerForm`: Cron-like UI for recurring reports.

## 4. State Management
- **Preview State**: Blob URLs, loading status.
- **Form State**: Complex configuration for report parameters.

## 5. Directory Structure
```
node5/
├── components/
│   ├── ReportDesigner/
│   ├── PDFViewer.tsx
│   └── Scheduler.tsx
├── hooks/
│   └── useReportPreview.ts
└── pages/
    └── ReportBuilder.tsx
```
