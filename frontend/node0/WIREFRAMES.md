# Node 0: Student Hub Frontend

Central hub for managing students and viewing aggregated profiles.

## Wireframes

### 2.1 Student Dashboard (Wireframe)

```mermaid
graph TB
    subgraph Dashboard [Student Hub Dashboard]
        StatsRow[Statistics Cards]
        StatsRow --> TotalStudents[Total Students]
        StatsRow --> AtRisk[At Risk Count]
        StatsRow --> ActiveInterventions[Active Interventions]
        
        FilterRow[Filter & Search]
        FilterRow --> Search[Search by Name/ID]
        FilterRow --> GradeFilter[Grade Filter]
        FilterRow --> ClassFilter[Class Filter]
        
        ListArea[Student List Table]
        ListArea --> Cols[Columns: Name, ID, Grade, Mastery Avg, Status, Actions]
        
        DetailsPanel[Student Details Drawer (Overlay)]
    end
```

### 2.2 Student Detail View (Unified Profile)

Aggregates data from other nodes.

```mermaid
graph LR
    subgraph ProfileView [Student Detail Page]
        LeftCol[Left Column]
        RightCol[Right Column]
        
        LeftCol --> BasicInfo[Basic Info Card]
        LeftCol --> SchoolInfo[School Info]
        LeftCol --> Tags[Metadata Tags]
        
        RightCol --> Tabs[Tab Navigation]
        
        Tabs --> Tab1[Overview (Dashboard)]
        Tabs --> Tab2[Mastery (Node 2)]
        Tabs --> Tab3[Activities (Node 4)]
        Tabs --> Tab4[Reports (Node 5)]
        Tabs --> Tab5[Interventions (Node 0)]
        
        Tab1 --> MasteryChart[Mastery Radar Chart]
        Tab1 --> RecentActivity[Recent Activity Timeline]
        Tab1 --> WeaknessList[Weak Concept List]
    end
```
