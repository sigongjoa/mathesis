# Node 5: Report Node Frontend

Reporting and export interface.

## Wireframes

### 7.1 Report Builder

```mermaid
graph TD
    subgraph ReportUI [Report Manager]
        TemplateSel[Template Selector]
        
        Preview[Live Preview]
        Preview --> PDFRender[PDF Renderer (Typst)]
        
        Actions[Action Bar]
        Actions --> ExportPDF[Download PDF]
        Actions --> SendEmail[Email to Parents]
    end
```
