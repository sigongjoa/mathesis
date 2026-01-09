# Node 1: Logic Engine Frontend

Visualization of the educational concept graph.

## Wireframes

### 3.1 Graph Explorer UI

```mermaid
graph TD
    subgraph GraphExplorer [Knowledge Graph View]
        Toolbar[Graph Toolbar]
        Toolbar --> Zoom[Zoom In/Out]
        Toolbar --> Layout[Layout Switcher]
        Toolbar --> NodeFilter[Concept Filter]
        
        Canvas[Interactive Canvas (D3/Cytoscape)]
        Canvas --> Nodes[Concept Nodes]
        Canvas --> Edges[Relationship Links]
        
        Sidebar[Node Inspector]
        Sidebar --> NodeProps[Properties]
        Sidebar --> NodePre[Prerequisites]
        Sidebar --> NodePost[Post-requisites]
    end
```
