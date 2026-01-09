# Node 3: Gen Node Frontend

AI-powered content creation.

## Wireframes

### 5.1 Generator Interface

```mermaid
graph TD
    subgraph GeneratorUI [Content Generator]
        ConfigPanel[Configuration]
        ConfigPanel --> TypeSel[Type: Problem / Explanation]
        ConfigPanel --> ConceptSel[Target Concepts]
        ConfigPanel --> DiffSel[Difficulty Level]
        ConfigPanel --> CountIn[Count]
        
        PreviewPanel[Generation Preview]
        PreviewPanel --> Loading[Loading State (Streaming)]
        PreviewPanel --> Items[Generated Items List]
        
        Items --> Item1[Item Card]
        Item1 --> EditBtn[Edit]
        Item1 --> RegenBtn[Regenerate]
        Item1 --> ApproveBtn[Approve]
    end
```
