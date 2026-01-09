# Node 4: Lab Node - Frontend Architecture

## 1. Information Architecture (IA)

```mermaid
graph TD
    Root[Node 4 Root] --> LabList[Experiments Library]
    Root --> Workspace[Virtual Lab Workspace]
    Root --> Assignments[Lab Assignments]
```

## 2. User Flows

### 2.1 Performing a Geometry Experiment

```mermaid
sequenceDiagram
    actor Student
    participant Workspace as Lab Workspace
    participant Tool as Geometry Tool
    participant API as /api/v1/lab
    
    Student->>Workspace: Open "Triangle Properties"
    Student->>Tool: Select "Draw Triangle"
    Student->>Workspace: Click 3 points
    Workspace->>Student: Render Triangle
    Student->>Tool: Select "Measure Angle"
    Student->>Workspace: Click Vertex A
    Workspace->>Student: Show "60°"
    Student->>Workspace: Drag Vertex B
    Workspace->>Student: Update Angles realtime
    Student->>Workspace: Click "Submit Results"
    Workspace->>API: POST /submit-result
```

## 3. Component Architecture

### 3.1 Components
- `CanvasWorkspace`: The core interactive area (HTML5 Canvas/WebGL).
- `ToolPalette`: Floating toolbar.
- `StepGuide`: Tutorial overlay.

## 4. State Management
- **Canvas State**: **Crucial**. Needs high-performance state (not just React state). Possibly Redux or a specialized history stack (undo/redo).
- **Session State**: Tracking time spent, steps completed.

## 5. Directory Structure
```
node4/
├── components/
│   ├── Canvas/
│   │   ├── Renderer.ts
│   │   └── InteractionLayer.tsx
│   ├── ToolPalette.tsx
│   └── Console.tsx
├── hooks/
│   └── useLabSession.ts
└── pages/
    └── LabWorkspace.tsx
```
