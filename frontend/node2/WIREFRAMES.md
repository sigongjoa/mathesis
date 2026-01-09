# Node 2: Q-DNA Frontend

Item Response Theory (IRT) and Mastery tracking.

## Wireframes

### 4.1 Mastery Heatmap UI

```mermaid
graph TD
    subgraph HeatmapView [Class Mastery Heatmap]
        Controls[Heatmap Controls]
        Controls --> TopicSel[Select Topic]
        Controls --> ViewMode[Student vs Concept Mode]
        
        Grid[Heatmap Grid]
        Grid --> Rows[Rows: Students]
        Grid --> Cols[Columns: Concepts]
        Grid --> Cells[Color Coded: Red(Low) -> Green(High)]
        
        Tooltip[Hover Tooltip]
    end
```
