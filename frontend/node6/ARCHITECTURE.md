# Node 6: School Info - Frontend Architecture

## 1. Information Architecture (IA)

```mermaid
graph TD
    Root[Node 6 Root] --> Search[School Search]
    Root --> Detail[School Detail]
    Detail --> Stats[Statistics]
    Detail --> Docs[Documents]
    Detail --> Curriculum[Curriculum]
```

## 2. User Flows

### 2.1 Downloading School Documents

```mermaid
sequenceDiagram
    actor Parent
    participant Search as Search Page
    participant Detail as Detail Page
    participant API as /api/v1/schools
    
    Parent->>Search: Type "Dongdo Middle"
    Search->>API: GET /search?q=Dongdo
    API-->>Search: [SchoolA, SchoolB]
    Parent->>Search: Click "Dongdo Middle School"
    Search->>Detail: Navigate to /schools/:id
    Detail->>API: GET /schools/:id/documents
    API-->>Detail: Document List
    Parent->>Detail: Click "Download Plan"
    Detail->>API: GET /download/:docId
    API-->>Parent: File Stream
```

## 3. Component Architecture

### 3.1 Components
- `SchoolSearchInput`: Autocomplete search bar.
- `DocumentList`: Grid/List view of files with icons.
- `StatCard`: Visualizing school metrics (e.g., student counts).

## 4. State Management
- **Search State**: Query string, filters, pagination.
- **Cache**: React Query for school details (rarely changes).

## 5. Directory Structure
```
node6/
├── components/
│   ├── SearchBar.tsx
│   ├── DocumentList.tsx
│   └── FilterPanel.tsx
├── hooks/
│   └── useSchoolSearch.ts
└── pages/
    ├── SearchPage.tsx
    └── SchoolDetail.tsx
```
