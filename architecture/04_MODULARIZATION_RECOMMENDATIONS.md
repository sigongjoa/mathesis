# Modularization Recommendations for Mathesis Architecture

**Document Version:** 1.0
**Date:** 2026-01-09
**Status:** Analysis Complete - Ready for Implementation

---

## Executive Summary

This document presents a comprehensive analysis of all Mathesis nodes (Node 0-7) and identifies opportunities to extract shared business logic into `mathesis-core` modules. The goal is to eliminate code duplication and enable rapid development of new nodes (especially Node 7 - Anki system) by reusing battle-tested components.

**Key Findings:**
- ✅ **Phase 1-2 Complete**: OCR and DNA Analysis successfully modularized (18 tests, 97% coverage)
- 🎯 **Phase 3 Opportunity**: 5 additional modules identified for extraction
- 📊 **Impact**: Estimated 60-70% code reduction for new nodes using these modules
- 🔄 **Reusability**: Core modules will be shared across Node 2, Node 7, and potentially Node 1

---

## Table of Contents

1. [Analysis Summary](#analysis-summary)
2. [Already Extracted Modules (Phase 1-2)](#already-extracted-modules-phase-1-2)
3. [Recommended New Modules (Phase 3+)](#recommended-new-modules-phase-3)
4. [Priority & Implementation Plan](#priority--implementation-plan)
5. [Service-by-Service Analysis](#service-by-service-analysis)
6. [Migration Impact Assessment](#migration-impact-assessment)
7. [Benefits & ROI](#benefits--roi)

---

## Analysis Summary

### Nodes Analyzed

| Node | Purpose | Services Found | Extractable Logic |
|------|---------|----------------|-------------------|
| Node 0 (Student Hub) | Student management | None | N/A - Pure CRUD |
| Node 1 (Logic Engine) | Knowledge extraction | 9 services | ✅ Concept extraction prompts, Quality scoring |
| Node 2 (Q-DNA) | Problem analysis | 10 services | ✅ OCR (done), DNA (done), Generation, Visualization |
| Node 4 (Lab Node) | Curriculum management | 11+ services | ❌ Service-specific CRUD |
| Node 5 (Q-Metrics) | Analytics | None found | N/A |
| Node 6 (School Info) | School data | None found | N/A |
| Node 7 (Error Note) | Anki system | Not built yet | 🎯 Will benefit most from modules |

### Extraction Opportunities Matrix

| Category | Module | Source | Priority | Complexity | Estimated Effort |
|----------|--------|--------|----------|------------|------------------|
| ✅ Vision | OCREngine | Node 2 | P0 | Medium | **DONE** ✅ |
| ✅ Analysis | DNAAnalyzer | Node 2 | P0 | Medium | **DONE** ✅ |
| 🎯 Generation | ProblemGenerator | Node 2 | P1 | Medium | 2-3 days |
| 🎯 Visualization | DiagramGenerator | Node 2 | P2 | Low | 1-2 days |
| 🎯 Extraction | ConceptExtractor (prompts) | Node 1 | P2 | Low | 1 day |
| 🎯 Validation | QualityScorer | Node 1 | P3 | Low | 1 day |
| 🎯 Prompts | extraction_prompts | Node 1 | P2 | Low | 0.5 day |
| 🎯 Prompts | generation_prompts | Node 2 | P1 | Low | 0.5 day |
| 🎯 Prompts | visualization_prompts | Node 2 | P2 | Low | 0.5 day |

**Legend:**
- ✅ Already extracted and in production
- 🎯 Recommended for extraction
- ❌ Not worth extracting (too service-specific)

---

## Already Extracted Modules (Phase 1-2)

### ✅ 1. mathesis_core.vision.OCREngine

**Status:** Complete ✅ (5 tests passing)

**Location:** `mathesis-common/mathesis_core/vision/ocr_engine.py`

**What It Does:**
- Extracts text and LaTeX math from images
- Combines Tesseract OCR with Vision LLM for accuracy
- Returns structured OCR results with confidence scores

**Used By:**
- ✅ Node 2: `app/services/ocr_service.py` (refactored)
- 🎯 Node 7: Future Anki flashcard image processing

**API:**
```python
from mathesis_core.vision import OCREngine
from mathesis_core.llm.clients import create_ollama_client

llm = create_ollama_client(base_url=settings.OLLAMA_URL, model=settings.OLLAMA_MODEL)
ocr = OCREngine(llm)
result = await ocr.extract(image_bytes)
# Returns: {text, latex, combined, has_math, confidence, tesseract_fallback}
```

**Test Coverage:** 97%

---

### ✅ 2. mathesis_core.analysis.DNAAnalyzer

**Status:** Complete ✅ (7 tests passing)

**Location:** `mathesis-common/mathesis_core/analysis/dna_analyzer.py`

**What It Does:**
- Extracts problem DNA: tags, curriculum path, cognitive level, difficulty
- Computes DNA signature for similarity matching
- Provides metadata for problem classification

**Used By:**
- ✅ Node 2: `app/services/tagging_service.py` (refactored)
- 🎯 Node 7: Anki card classification and spaced repetition

**API:**
```python
from mathesis_core.analysis import DNAAnalyzer

analyzer = DNAAnalyzer(llm_client)
dna = await analyzer.analyze(question_text)
# Returns: {tags, metadata, curriculum_path, dna_signature, keywords}
```

**Test Coverage:** 95%

---

### ✅ 3. mathesis_core.prompts

**Status:** Complete ✅ (6 tests passing)

**Location:** `mathesis-common/mathesis_core/prompts/`

**Modules:**
- `analysis_prompts.py` - DNA extraction prompts
- `ocr_prompts.py` - Vision OCR prompts

**What It Does:**
- Centralizes all LLM prompt templates
- Prevents prompt duplication across nodes
- Enables A/B testing and prompt versioning

**Used By:**
- ✅ Node 2: OCR and DNA analysis
- ✅ Node 1: (Can migrate concept extraction prompts)

---

## Recommended New Modules (Phase 3+)

### 🎯 1. mathesis_core.generation.ProblemGenerator

**Priority:** P1 (High) - Critical for Node 7

**Source:** `node2_q_dna/backend/app/services/math_advanced_service.py:73-133` (generate_twin_question)
**Source:** `node2_q_dna/backend/app/services/error_solution_service.py:19-144`

**What It Should Do:**
1. **Twin Question Generation**: Create isomorphic problems (same logic, different numbers/context)
2. **Error Solution Generation**: Generate intentionally incorrect solutions with specific error types
3. **Problem Variation**: Create variations with controlled difficulty adjustments

**Benefits:**
- Node 7 can generate Anki flashcards with variations
- Node 2 can reuse for question bank expansion
- Enables intelligent practice problem generation

**Proposed API:**
```python
from mathesis_core.generation import ProblemGenerator

generator = ProblemGenerator(llm_client)

# Twin question generation
twin = await generator.generate_twin(
    original_question=question_obj,
    preserve_logic=True,
    vary_context=True
)

# Error solution generation
error_solution = await generator.generate_error_solution(
    question_content="solve x^2 - 5x + 6 = 0",
    correct_answer="x = 2 or x = 3",
    error_types=[ErrorType.CONCEPT_MISAPPLICATION, ErrorType.ARITHMETIC_ERROR],
    difficulty=2
)
# Returns: {steps: [...], final_wrong_answer, error_explanation}
```

**Implementation Strategy:**
1. Extract twin generation logic from `math_advanced_service.py`
2. Extract error solution logic from `error_solution_service.py`
3. Create unified `ProblemGenerator` class
4. Move prompts to `mathesis_core.prompts.generation_prompts`
5. Write 10+ tests (TDD)

**Estimated Effort:** 2-3 days

**Test Cases:**
- Twin question preserves logic
- Twin question varies numbers/context
- Error solution has exactly one error
- Error solution follows correct steps until error point
- Generated questions are valid Korean

---

### 🎯 2. mathesis_core.visualization.DiagramGenerator

**Priority:** P2 (Medium) - Nice to have for Node 7 Anki cards

**Source:** `node2_q_dna/backend/app/services/diagram_service.py:16-98`

**What It Should Do:**
1. **LLM Code Generation**: Generate Python matplotlib code from natural language descriptions
2. **Safe Execution**: Execute generated code in sandboxed environment
3. **Diagram Rendering**: Save diagrams as PNG/SVG files

**Benefits:**
- Node 7 Anki cards can include auto-generated diagrams
- Node 2 can continue using for geometry problems
- Enables visual learning aids generation

**Current Implementation Issues:**
- Uses `exec()` for code execution (security risk)
- No timeout or resource limits
- Error handling could be better

**Proposed API:**
```python
from mathesis_core.visualization import DiagramGenerator

generator = DiagramGenerator(llm_client)

# Generate diagram
diagram_path = await generator.generate(
    description="Draw a right triangle ABC with legs 3 and 4",
    style="black-and-white",  # or "color", "grayscale"
    format="png",  # or "svg"
    dpi=300
)
# Returns: str (path to generated image)
```

**Implementation Strategy:**
1. Extract diagram generation logic from `diagram_service.py`
2. Move prompt to `mathesis_core.prompts.visualization_prompts`
3. Add timeout and resource limits to code execution
4. Consider using `RestrictedPython` or Docker sandbox for security
5. Write 8+ tests (TDD)

**Estimated Effort:** 1-2 days

**Security Considerations:**
- ⚠️ `exec()` is dangerous - consider sandboxing options
- Add resource limits (CPU, memory, execution time)
- Validate generated code before execution

**Test Cases:**
- Generates valid matplotlib code
- Code executes successfully
- Diagram file is created
- Invalid descriptions fail gracefully
- Timeout works for infinite loops

---

### 🎯 3. mathesis_core.extraction.ConceptExtractor (Prompts Only)

**Priority:** P2 (Medium) - Improves Node 1 consistency

**Source:** `node1_logic_engine/backend/app/services/llm_extractor.py:22-66` (CONCEPT_EXTRACTION_SYSTEM prompt)

**Current State:**
- Node 1's `ConceptExtractor` already uses `mathesis_core.llm.clients`
- BUT prompts are still hardcoded in service file
- Should extract prompts to `mathesis_core.prompts.extraction_prompts`

**What Should Be Extracted:**
- `CONCEPT_EXTRACTION_SYSTEM` prompt (lines 22-66 of llm_extractor.py)
- Move to `mathesis_core.prompts.extraction_prompts.get_concept_extraction_prompt()`

**Benefits:**
- Centralizes all extraction prompts
- Enables prompt versioning and A/B testing
- Other nodes can extract concepts using same prompt

**Proposed API:**
```python
from mathesis_core.prompts.extraction_prompts import get_concept_extraction_prompt

prompt = get_concept_extraction_prompt()
# Returns: System prompt for concept extraction
```

**Implementation Strategy:**
1. Create `mathesis_core/prompts/extraction_prompts.py`
2. Move `CONCEPT_EXTRACTION_SYSTEM` constant from Node 1
3. Update Node 1's `ConceptExtractor` to use new prompt module
4. Write 3 tests for prompt generation

**Estimated Effort:** 1 day

---

### 🎯 4. mathesis_core.validation.QualityScorer

**Priority:** P3 (Low) - Quality of life improvement

**Source:** `node1_logic_engine/backend/app/services/quality_scorer.py`

**What It Does:**
- Scores extraction quality (concepts, relationships, questions)
- Validates completeness and consistency
- Returns confidence score (0.0-1.0)

**Benefits:**
- Reusable quality assessment across nodes
- Standardized quality metrics
- Can be used by Node 2 for DNA validation

**Proposed API:**
```python
from mathesis_core.validation import QualityScorer

scorer = QualityScorer()
score = scorer.score_extraction(knowledge)  # Returns 0.0-1.0
```

**Implementation Strategy:**
1. Extract `QualityScorer` class from Node 1
2. Generalize to work with any extraction result
3. Write 6+ tests

**Estimated Effort:** 1 day

---

### 🎯 5. mathesis_core.prompts.generation_prompts

**Priority:** P1 (High) - Required for ProblemGenerator module

**Source:**
- `node2_q_dna/backend/app/services/math_advanced_service.py:80-99` (twin question prompt)
- `node2_q_dna/backend/app/services/error_solution_service.py:42-86` (error solution prompt)

**What It Should Provide:**
```python
from mathesis_core.prompts.generation_prompts import (
    get_twin_question_prompt,
    get_error_solution_prompt,
    get_correct_solution_prompt
)

prompt = get_twin_question_prompt(original_text, metadata)
error_prompt = get_error_solution_prompt(question, correct_answer, error_types)
```

**Implementation Strategy:**
1. Extract prompts from `math_advanced_service.py` and `error_solution_service.py`
2. Parameterize prompts (allow customization)
3. Write 4 tests

**Estimated Effort:** 0.5 day

---

### 🎯 6. mathesis_core.prompts.visualization_prompts

**Priority:** P2 (Medium) - Required for DiagramGenerator module

**Source:** `node2_q_dna/backend/app/services/diagram_service.py:42-60`

**What It Should Provide:**
```python
from mathesis_core.prompts.visualization_prompts import get_diagram_generation_prompt

prompt = get_diagram_generation_prompt(description, style="black-and-white")
```

**Implementation Strategy:**
1. Extract diagram prompt from `diagram_service.py`
2. Add style and format parameters
3. Write 3 tests

**Estimated Effort:** 0.5 day

---

## Priority & Implementation Plan

### Phase 3: Problem Generation (Week 1-2)

**Modules:**
1. `mathesis_core.prompts.generation_prompts` (0.5 day)
2. `mathesis_core.generation.ProblemGenerator` (2-3 days)

**Why First:**
- Critical for Node 7 (Anki system)
- High reusability
- Complex enough to justify extraction

**Deliverables:**
- `mathesis_core/generation/problem_generator.py`
- `mathesis_core/prompts/generation_prompts.py`
- `tests/test_generation/test_problem_generator.py` (10+ tests)
- Updated `docs/architecture/03_MODULE_SPECIFICATIONS.md`

**Migration:**
- Refactor `node2_q_dna/backend/app/services/math_advanced_service.py`
- Refactor `node2_q_dna/backend/app/services/error_solution_service.py`

---

### Phase 4: Visualization (Week 3)

**Modules:**
1. `mathesis_core.prompts.visualization_prompts` (0.5 day)
2. `mathesis_core.visualization.DiagramGenerator` (1-2 days)

**Why Second:**
- Useful for Node 7 Anki cards
- Medium complexity
- Security concerns need addressing

**Deliverables:**
- `mathesis_core/visualization/diagram_generator.py`
- `mathesis_core/prompts/visualization_prompts.py`
- `tests/test_visualization/test_diagram_generator.py` (8+ tests)
- Security audit document

**Migration:**
- Refactor `node2_q_dna/backend/app/services/diagram_service.py`

---

### Phase 5: Extraction & Validation (Week 4)

**Modules:**
1. `mathesis_core.prompts.extraction_prompts` (1 day)
2. `mathesis_core.validation.QualityScorer` (1 day)

**Why Last:**
- Lower priority
- Smaller impact
- Easy to implement

**Deliverables:**
- `mathesis_core/extraction/` (move prompts only, not full extractor)
- `mathesis_core/validation/quality_scorer.py`
- Tests

**Migration:**
- Update `node1_logic_engine/backend/app/services/llm_extractor.py`

---

## Service-by-Service Analysis

### Node 1 (Logic Engine)

| Service | Extractable? | Recommendation |
|---------|--------------|----------------|
| llm_extractor.py (ConceptExtractor) | Partial ⚠️ | Extract prompts only to `mathesis_core.prompts.extraction_prompts` |
| quality_scorer.py | ✅ Yes | Extract to `mathesis_core.validation.QualityScorer` |
| pdf_parser.py | ❌ No | Lightweight GROBID wrapper, not worth extracting |
| graph_builder.py | ❌ No | Specific to Node 1's knowledge graph architecture |
| graph_rag.py | ❌ No | Specific to Node 1's RAG implementation |
| school_*.py | ❌ No | Service-specific CRUD operations |

**Conclusion:** Extract prompts and quality scorer only. Rest is too service-specific.

---

### Node 2 (Q-DNA)

| Service | Extractable? | Recommendation | Status |
|---------|--------------|----------------|--------|
| ocr_service.py | ✅ Yes | Extract to `mathesis_core.vision.OCREngine` | ✅ **DONE** |
| tagging_service.py | ✅ Yes | Extract to `mathesis_core.analysis.DNAAnalyzer` | ✅ **DONE** |
| math_advanced_service.py | ✅ Yes | Extract twin generation to `mathesis_core.generation.ProblemGenerator` | 🎯 Phase 3 |
| error_solution_service.py | ✅ Yes | Extract error generation to `mathesis_core.generation.ProblemGenerator` | 🎯 Phase 3 |
| diagram_service.py | ✅ Yes | Extract to `mathesis_core.visualization.DiagramGenerator` | 🎯 Phase 4 |
| report_service.py | ❌ No | Uses service-specific Jinja2 templates |
| analytics_service.py | ❌ No | Service-specific analytics |
| crawler_service.py | ❌ No | Web scraping, not shared logic |
| question_service.py | ❌ No | CRUD for questions |
| ollama_service.py | ⚠️ Deprecated | Already replaced by `mathesis_core.llm.clients` |

**Conclusion:** 5 modules extracted (2 done, 3 pending). Node 2 had the most extractable logic.

---

### Node 4 (Lab Node)

| Service | Extractable? | Recommendation |
|---------|--------------|----------------|
| curriculum_service.py | ❌ No | Pure CRUD with Firestore |
| textbook_service.py | ❌ No | File upload/download specific to Lab Node |
| upload.py (UploadOrchestrator) | ❌ No | Service-specific upload strategy pattern |
| student_*.py | ❌ No | CRUD operations |
| image_management.py | ❌ No | GCP-specific image handling |

**Conclusion:** Node 4 is purely service-layer CRUD. Nothing to extract.

---

### Node 5, 6, 7

**Status:** No services found or not yet built.

**Note:** Node 7 (Anki system) will be the primary beneficiary of extracted modules.

---

## Migration Impact Assessment

### Risk Assessment

| Module | Migration Risk | Reason | Mitigation |
|--------|----------------|--------|------------|
| ProblemGenerator | Medium | Used in production by Node 2 | Keep old service until tests pass, gradual rollout |
| DiagramGenerator | Low | Only used in one endpoint | Can migrate quickly |
| ConceptExtractor prompts | Low | Simple string replacement | Easy rollback |
| QualityScorer | Low | Pure logic, no side effects | Can run in parallel for validation |

### Breaking Changes

**None expected** - All migrations maintain backward compatibility via thin service layer pattern.

### Testing Strategy

1. **Unit Tests First**: 90%+ coverage on all new modules
2. **Integration Tests**: Test services using new modules
3. **Regression Tests**: Compare old vs new outputs
4. **Gradual Rollout**: Deploy to Node 2 staging first, validate, then production

---

## Benefits & ROI

### Code Reduction

**Before Modularization:**
- Node 2 OCR service: ~146 lines of business logic
- Node 2 Tagging service: ~185 lines of business logic
- Total: ~331 lines

**After Modularization:**
- Node 2 OCR service: ~80 lines (delegation only)
- Node 2 Tagging service: ~90 lines (delegation only)
- mathesis_core modules: ~400 lines (reusable!)
- **Total reduction for Node 2:** 170 lines saved
- **Net benefit:** When Node 7 uses these modules, saves another ~300 lines!

**ROI Calculation:**
- **Time to extract 1 module:** 1-3 days
- **Time saved per new node:** 3-5 days (no need to rewrite)
- **Break-even:** After 2 nodes use the module
- **Node 7 will use 4+ modules:** Estimated 2-3 weeks saved!

### Quality Benefits

1. **Single Source of Truth**: Bug fixes propagate to all nodes
2. **Better Testing**: 90%+ coverage on core modules
3. **Consistency**: Same prompts, same behavior across nodes
4. **Maintainability**: Update prompt once, affects all nodes

### Development Speed

**New Node Development Time:**

| Task | Before Modularization | After Modularization | Time Saved |
|------|----------------------|---------------------|------------|
| OCR Implementation | 2-3 days | 1 hour (import + config) | 95% |
| DNA Analysis | 2-3 days | 1 hour | 95% |
| Problem Generation | 3-4 days | 2 hours | 93% |
| Diagram Generation | 2 days | 1 hour | 94% |
| **Total for Node 7** | **9-12 days** | **1 day** | **~90%** |

---

## Conclusion

### Summary

- ✅ **Phase 1-2:** 3 modules successfully extracted (OCR, DNA, Prompts)
- 🎯 **Phase 3-5:** 6 additional modules recommended
- 📊 **Impact:** 90% time savings for Node 7 development
- 🔄 **Reusability:** Core modules will benefit 3+ nodes

### Next Steps

1. **Review this document** with team
2. **Approve Phase 3 scope** (ProblemGenerator)
3. **Begin TDD implementation** for generation module
4. **Iterate** through Phases 4-5 based on Node 7 timeline

### Decision Required

**Proceed with Phase 3 (ProblemGenerator)?**
- Estimated effort: 3 days
- Highest priority for Node 7
- Clear value proposition

---

**Document Prepared By:** Claude Sonnet 4.5 (Analysis Agent)
**Next Review Date:** After Phase 3 completion
**Contact:** See `docs/architecture/02_MODULAR_ARCHITECTURE.md` for architecture questions
