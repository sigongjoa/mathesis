# Node 0: Student Hub - Frontend Architecture

## 1. Information Architecture (IA)

Structure of the module's navigation and page hierarchy.

```mermaid
graph TD
    Root[Node 0 Root] --> Dashboard[Dashboard / Overview]
    Root --> StudentList[Student List]
    Root --> StudentDetail[Student Detail Profile]
    
    StudentDetail --> ProfileTab[Profile Overview]
    StudentDetail --> HistoryTab[Learning History]
    StudentDetail --> InterventionTab[Intervention Config]
    StudentDetail --> ReportTab[Generated Reports]
```

## 2. User Flows

### 2.1 Viewing a Student's Unified Profile

```mermaid
sequenceDiagram
    actor Teacher
    participant UI as StudentList Page
    participant Detail as StudentDetail Page
    participant API as /api/v1/profiles/unified
    
    Teacher->>UI: Clicks on Student Name
    UI->>Detail: Navigate to /students/:id
    Detail->>API: GET /unified?student_id=...
    API-->>Detail: UnifiedProfile JSON
    Detail->>Teacher: Renders Dashboard (Mastery, Activities)
```

### 2.2 Creating a Learning Intervention

```mermaid
sequenceDiagram
    actor Teacher
    participant UI as Intervention Tab
    participant Modal as Create Modal
    participant API as /api/v1/interventions
    
    Teacher->>UI: Click "New Intervention"
    UI->>Modal: Open Form
    Teacher->>Modal: Select Concept & Type
    Teacher->>Modal: Submit
    Modal->>API: POST /interventions
    API-->>Modal: 201 Created
    Modal->>UI: Close & Refresh List
    UI->>Teacher: Show Toast "Intervention Started"
```

## 3. Component Architecture

### 3.1 Page Components
- `StudentDashboardPage`: Main entry point.
- `StudentListPage`: Table view of all students.
- `StudentDetailPage`: Parent for tab views.

### 3.2 Feature Components (Atomic)
- `StudentCard`: Summary card.
- `MasteryRadarChart`: Visualization of weak/strong areas (uses Recharts/Chart.js).
- `ActivityTimeline`: List of recent actions.
- `InterventionList`: Interactive list of active interventions.

### 3.3 Props Interface (Example)

```typescript
interface StudentDetailProps {
  studentId: string;
}

interface InterventionListProps {
  interventions: Intervention[];
  onCancel: (id: string) => void;
}
```

## 4. State Management

### 4.1 Server State (React Query)
- `useStudent(id)`: Fetches `GET /students/:id`
- `useUnifiedProfile(id)`: Fetches `GET /profiles/unified`
- `useInterventions(studentId)`: Fetches `GET /interventions`

### 4.2 Local State
- `selectedTab`: Controls current view in Detail Page.
- `isCreateModalOpen`: Controls visibility of intervention form.

## 5. Directory Structure

```
node0/
├── pages/
│   ├── Dashboard.tsx
│   ├── StudentList.tsx
│   └── StudentDetail.tsx
├── components/
│   ├── StudentCard.tsx
│   ├── MasteryRadar.tsx
│   ├── InterventionForm.tsx
│   └── ...
├── hooks/
│   ├── useStudent.ts
│   └── useIntervention.ts
└── types/
    └── index.ts
```
