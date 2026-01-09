# Node 2: Q-DNA - Frontend Architecture

## 1. Information Architecture (IA)

```mermaid
graph TD
    Root[Node 2 Root] --> Heatmap[Class Mastery Heatmap]
    Root --> Analysis[Item Analysis]
    Root --> Trace[Knowledge Trace View]
```

## 2. User Flows

### 2.1 Analyzing Class Weaknesses

```mermaid
sequenceDiagram
    actor Teacher
    participant View as Heatmap View
    participant API as /api/v1/analytics
    
    Teacher->>View: Select "Algebra 1" Class
    View->>API: GET /mastery/heatmap?class=...
    API-->>View: Matrix Data [Students x Concepts]
    View->>Teacher: Render Color Grids
    Teacher->>View: Hover over Red Column
    View->>Teacher: Show "Quadratic Formula" (Avg: 30%)
```

## 3. Component Architecture

### 3.1 Components
- `HeatmapGrid`: SVG/Canvas based grid renderer.
- `TraceGraph`: Line chart showing mastery over time (BKT/DKT).
- `ItemCard`: Display of a specific question item with stats.

## 4. State Management
- **Analytics Data**: Heavily dependent on React Query with caching.
- **Filters**: Local state for time ranges, student groups.

## 5. Directory Structure
```
node2/
├── components/
│   ├── Heatmap/
│   ├── TraceGraph.tsx
│   └── ItemCard.tsx
├── hooks/
│   └── useAnalytics.ts
└── pages/
    ├── MasteryView.tsx
    └── ItemAnalysis.tsx
```
