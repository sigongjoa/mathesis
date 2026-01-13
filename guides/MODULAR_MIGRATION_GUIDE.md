# Modular Architecture Migration Guide

> Step-by-step guide to migrate existing services to module-based architecture

**Last Updated**: 2026-01-09
**Version**: 1.0
**Target**: Node 2 (Q-DNA) → mathesis-common modules

---

## 1. Migration Overview

### 1.1 Migration Goals

- ✅ Extract core business logic from services into reusable modules
- ✅ Maintain backward compatibility during migration
- ✅ Achieve >90% test coverage for new modules
- ✅ Zero downtime migration

### 1.2 Migration Timeline

| Phase | Duration | Tasks | Status |
|-------|----------|-------|--------|
| Phase 1 | Week 1 | Module creation + Tests | 🟡 In Progress |
| Phase 2 | Week 2 | Node 2 refactoring | ⏳ Pending |
| Phase 3 | Week 3 | Node 1 alignment | ⏳ Pending |
| Phase 4 | Week 4 | Node 7 implementation | ⏳ Pending |

---

## 2. Phase 1: Module Creation (Week 1)

### 2.1 Setup mathesis-common Structure

```bash
cd mathesis-common

# Create module directories
mkdir -p mathesis_core/{vision,analysis,generation,prompts,workflows}
mkdir -p tests/{test_vision,test_analysis,test_generation,test_workflows}

# Create __init__.py files
touch mathesis_core/vision/__init__.py
touch mathesis_core/analysis/__init__.py
touch mathesis_core/generation/__init__.py
touch mathesis_core/prompts/__init__.py
touch mathesis_core/workflows/__init__.py
```

### 2.2 Install Development Dependencies

```bash
# mathesis-common/pyproject.toml
[tool.poetry.dependencies]
python = "^3.11"
pytesseract = "^0.3.10"
Pillow = "^10.0.0"
httpx = "^0.25.0"
pydantic = "^2.0.0"

[tool.poetry.group.dev.dependencies]
pytest = "^7.4.0"
pytest-asyncio = "^0.21.0"
pytest-cov = "^4.1.0"
pytest-mock = "^3.11.1"
```

```bash
cd mathesis-common
poetry install
```

### 2.3 Create Base Module Structure

#### Step 1: Create Exceptions Module

```python
# mathesis_core/exceptions.py
class MathesisCoreError(Exception):
    """Base exception for mathesis-core."""
    pass

class OCRError(MathesisCoreError):
    """OCR extraction failed."""
    pass

class AnalysisError(MathesisCoreError):
    """DNA analysis failed."""
    pass

class GenerationError(MathesisCoreError):
    """Problem generation failed."""
    pass

class LLMTimeoutError(MathesisCoreError):
    """LLM request timed out."""
    pass
```

#### Step 2: Create Prompts Module (TDD)

**Test First:**

```python
# tests/test_prompts/test_analysis_prompts.py
import pytest
from mathesis_core.prompts.analysis_prompts import (
    get_tagging_prompt,
    get_metadata_prompt,
    get_curriculum_prompt
)

def test_tagging_prompt_includes_question_text():
    question = "x^2 + 2x + 1 = 0을 풀어라"
    prompt = get_tagging_prompt(question)

    assert question in prompt
    assert "tags" in prompt.lower()
    assert "json" in prompt.lower()

def test_metadata_prompt_includes_required_fields():
    question = "테스트 문제"
    prompt = get_metadata_prompt(question)

    assert "cognitive_level" in prompt
    assert "difficulty" in prompt
    assert "json" in prompt.lower()

def test_curriculum_prompt_requests_ltree_format():
    question = "이차방정식"
    prompt = get_curriculum_prompt(question)

    assert "Math" in prompt or "curriculum" in prompt.lower()
    assert "path" in prompt.lower()
```

**Then Implementation:**

```python
# mathesis_core/prompts/analysis_prompts.py
def get_tagging_prompt(question_text: str) -> str:
    """Get prompt for tag extraction."""
    return f"""Analyze this educational question and generate relevant tags.

Question:
{question_text}

Identify and categorize tags in these types:
1. subject: Main subject area (Math, Science, English, etc.)
2. concept: Specific concepts (Algebra, Geometry, etc.)
3. skill: Required skills (Problem Solving, etc.)
4. cognitive_level: Bloom's taxonomy (Remember, Understand, Apply, Analyze, Evaluate, Create)
5. difficulty: Difficulty level (Easy, Medium, Hard)

Return JSON array of tags with confidence scores (0.0-1.0):
{{
    "tags": [
        {{"tag": "Mathematics", "type": "subject", "confidence": 0.99}},
        {{"tag": "Algebra", "type": "concept", "confidence": 0.95}},
        {{"tag": "Apply", "type": "cognitive_level", "confidence": 0.90}}
    ]
}}

Be precise and assign high confidence (>0.9) only to clearly relevant tags."""


def get_metadata_prompt(question_text: str) -> str:
    """Get prompt for metadata extraction."""
    return f"""Analyze this educational question and provide metadata.

Question:
{question_text}

Return JSON with exact fields:
{{
    "cognitive_level": "one of: Remember|Understand|Apply|Analyze|Evaluate|Create",
    "difficulty_estimation": 0.0-1.0 (0.0=easiest, 1.0=hardest),
    "estimated_time_minutes": integer (realistic time to solve),
    "subject_area": "Math|Science|English|Social Studies|etc",
    "curriculum_path": "Subject.Topic.Subtopic (e.g., Math.Algebra.Quadratics)",
    "requires_calculator": true/false,
    "language": "Korean|English|Mixed"
}}"""


def get_curriculum_prompt(question_text: str) -> str:
    """Get prompt for curriculum path suggestion."""
    return f"""Given this question, suggest the most specific curriculum path in dot notation.

Question: {question_text}

Example paths:
- Math.Algebra.Linear_Equations
- Math.Geometry.Triangles.Pythagorean_Theorem
- Science.Physics.Mechanics.Kinematics
- Math.Calculus.Derivatives.Chain_Rule

Return ONLY the path string, no explanation."""
```

#### Step 3: Run Tests

```bash
cd mathesis-common
pytest tests/test_prompts/ -v
```

---

## 3. Phase 1 Continued: Core Module Implementation

### 3.1 Vision Module (TDD)

**Test First:**

```python
# tests/test_vision/test_ocr_engine.py
import pytest
from mathesis_core.vision import OCREngine
from mathesis_core.llm.clients import LLMClient
from unittest.mock import AsyncMock, Mock

@pytest.fixture
def mock_llm_client():
    client = Mock(spec=LLMClient)
    client.generate = AsyncMock(return_value='{"text": "test", "latex_expressions": [], "combined_content": "test", "has_mathematical_content": false}')
    return client

@pytest.mark.asyncio
async def test_ocr_engine_extracts_text(mock_llm_client):
    ocr = OCREngine(mock_llm_client)

    # Mock image bytes
    test_image = b"fake_image_data"

    result = await ocr.extract(test_image)

    assert "text" in result
    assert "latex" in result
    assert "combined" in result
    assert "has_math" in result

@pytest.mark.asyncio
async def test_ocr_engine_returns_confidence(mock_llm_client):
    ocr = OCREngine(mock_llm_client)
    result = await ocr.extract(b"test")

    assert "confidence" in result or "tesseract_fallback" in result
```

**Then Implementation:**

```python
# mathesis_core/vision/ocr_engine.py
from typing import Dict, Any
from PIL import Image
import io
import pytesseract
from mathesis_core.llm.clients import LLMClient
from mathesis_core.llm.parsers import LLMJSONParser
from mathesis_core.exceptions import OCRError
import logging

logger = logging.getLogger(__name__)

class OCREngine:
    """
    OCR Engine using Tesseract + Vision LLM.
    Pure business logic, no dependencies on DB or FastAPI.
    """

    def __init__(self, llm_client: LLMClient):
        self.llm = llm_client

    async def extract(self, image_bytes: bytes) -> Dict[str, Any]:
        """
        Extract text and LaTeX from image.

        Args:
            image_bytes: Raw image bytes

        Returns:
            {
                "text": str,
                "latex": List[str],
                "combined": str,
                "has_math": bool,
                "confidence": float,
                "tesseract_fallback": str
            }

        Raises:
            OCRError: If extraction completely fails
        """
        try:
            # Step 1: Tesseract OCR
            image = Image.open(io.BytesIO(image_bytes))
            tesseract_text = pytesseract.image_to_string(image, lang='eng+kor')

            # Step 2: Vision LLM
            vision_result = await self._analyze_with_vision(image_bytes)

            return {
                "text": vision_result.get("text", tesseract_text),
                "latex": vision_result.get("latex_expressions", []),
                "combined": vision_result.get("combined_content", tesseract_text),
                "has_math": vision_result.get("has_mathematical_content", False),
                "confidence": vision_result.get("confidence", 0.8),
                "tesseract_fallback": tesseract_text
            }

        except Exception as e:
            logger.error(f"OCR extraction failed: {e}")
            raise OCRError(f"Failed to extract text from image: {str(e)}")

    async def _analyze_with_vision(self, image_bytes: bytes) -> Dict:
        """Call Vision LLM for refinement."""
        from mathesis_core.prompts.ocr_prompts import get_vision_prompt

        prompt = get_vision_prompt()

        # Note: This requires implementing analyze_image_bytes in LLMClient
        # For now, use generate with base64 encoding
        import base64
        image_b64 = base64.b64encode(image_bytes).decode('utf-8')

        # Construct vision prompt (implementation depends on LLM provider)
        response = await self.llm.generate(prompt, format="json")

        return LLMJSONParser.safe_parse(response, default={
            "text": "",
            "latex_expressions": [],
            "combined_content": "",
            "has_mathematical_content": False
        })
```

**OCR Prompts:**

```python
# mathesis_core/prompts/ocr_prompts.py
def get_vision_prompt() -> str:
    """Get prompt for vision-based OCR."""
    return """Analyze this image and extract:
1. All text content (Korean and English)
2. All mathematical expressions in LaTeX format
3. Structure and formatting

Return in this exact JSON format:
{
    "text": "extracted plain text",
    "latex_expressions": ["\\\\frac{1}{2}", "x^2 + y^2 = z^2"],
    "combined_content": "full content with $latex$ inline and $$latex$$ display math",
    "has_mathematical_content": true/false,
    "confidence": 0.0-1.0
}"""
```

### 3.2 Analysis Module (TDD)

**Test First:**

```python
# tests/test_analysis/test_dna_analyzer.py
import pytest
from mathesis_core.analysis import DNAAnalyzer
from unittest.mock import AsyncMock, Mock

@pytest.fixture
def mock_llm_client():
    client = Mock()
    client.generate = AsyncMock(side_effect=[
        '{"tags": [{"tag": "Algebra", "type": "concept", "confidence": 0.95}]}',  # Tags
        '{"cognitive_level": "Apply", "difficulty_estimation": 0.6, "subject_area": "Mathematics"}',  # Metadata
        'Math.Algebra.Quadratics'  # Curriculum
    ])
    return client

@pytest.mark.asyncio
async def test_dna_analyzer_extracts_tags(mock_llm_client):
    analyzer = DNAAnalyzer(mock_llm_client)

    dna = await analyzer.analyze("x^2 + 2x + 1 = 0을 풀어라")

    assert "tags" in dna
    assert len(dna["tags"]) > 0
    assert dna["tags"][0]["tag"] == "Algebra"

@pytest.mark.asyncio
async def test_dna_analyzer_extracts_metadata(mock_llm_client):
    analyzer = DNAAnalyzer(mock_llm_client)

    dna = await analyzer.analyze("test question")

    assert "metadata" in dna
    assert "cognitive_level" in dna["metadata"]
    assert "difficulty_estimation" in dna["metadata"]

@pytest.mark.asyncio
async def test_dna_analyzer_generates_signature(mock_llm_client):
    analyzer = DNAAnalyzer(mock_llm_client)

    dna = await analyzer.analyze("test question")

    assert "dna_signature" in dna
    assert len(dna["dna_signature"]) == 16  # MD5 hash truncated
```

**Then Implementation:**

```python
# mathesis_core/analysis/dna_analyzer.py
from typing import Dict, List
import hashlib
from mathesis_core.llm.clients import LLMClient
from mathesis_core.llm.parsers import LLMJSONParser
from mathesis_core.exceptions import AnalysisError
import logging

logger = logging.getLogger(__name__)

class DNAAnalyzer:
    """
    DNA Analyzer - Extract problem DNA (concepts, difficulty, tags).
    Pure business logic, reusable across all nodes.
    """

    def __init__(self, llm_client: LLMClient):
        self.llm = llm_client

    async def analyze(self, question_text: str) -> Dict:
        """
        Analyze question and extract DNA.

        Returns:
            {
                "tags": List[Dict],
                "metadata": Dict,
                "curriculum_path": str,
                "dna_signature": str,
                "keywords": List[str]
            }
        """
        try:
            # Extract tags
            tags = await self._extract_tags(question_text)

            # Extract metadata
            metadata = await self._extract_metadata(question_text)

            # Suggest curriculum
            curriculum_path = await self._suggest_curriculum(question_text)

            # Compute signature
            dna_signature = self._compute_signature(tags, metadata)

            # Extract keywords
            keywords = self._extract_keywords(question_text, tags)

            return {
                "tags": tags,
                "metadata": metadata,
                "curriculum_path": curriculum_path,
                "dna_signature": dna_signature,
                "keywords": keywords
            }

        except Exception as e:
            logger.error(f"DNA analysis failed: {e}")
            raise AnalysisError(f"Failed to analyze question: {str(e)}")

    async def _extract_tags(self, text: str) -> List[Dict]:
        """Extract tags using LLM."""
        from mathesis_core.prompts.analysis_prompts import get_tagging_prompt

        prompt = get_tagging_prompt(text)
        response = await self.llm.generate(prompt, format="json", temperature=0.3)

        result = LLMJSONParser.safe_parse(response, default={"tags": []})
        return result.get("tags", [])

    async def _extract_metadata(self, text: str) -> Dict:
        """Extract metadata using LLM."""
        from mathesis_core.prompts.analysis_prompts import get_metadata_prompt

        prompt = get_metadata_prompt(text)
        response = await self.llm.generate(prompt, format="json", temperature=0.2)

        return LLMJSONParser.safe_parse(response, default={
            "cognitive_level": "Apply",
            "difficulty_estimation": 0.5,
            "subject_area": "General"
        })

    async def _suggest_curriculum(self, text: str) -> str:
        """Suggest curriculum path using LLM."""
        from mathesis_core.prompts.analysis_prompts import get_curriculum_prompt

        prompt = get_curriculum_prompt(text)
        path = await self.llm.generate(prompt, temperature=0.1)
        return path.strip().strip('"\'').split('\n')[0]

    def _compute_signature(self, tags: List[Dict], metadata: Dict) -> str:
        """
        Compute DNA signature for similarity search.

        Signature = MD5(sorted_concepts + cognitive_level + difficulty)
        """
        concept_tags = [t["tag"] for t in tags if t.get("type") == "concept"]
        concept_tags.sort()

        cognitive = metadata.get("cognitive_level", "Apply")
        difficulty = metadata.get("difficulty_estimation", 0.5)

        signature_str = f"{','.join(concept_tags)}|{cognitive}|{difficulty:.1f}"
        return hashlib.md5(signature_str.encode()).hexdigest()[:16]

    def _extract_keywords(self, text: str, tags: List[Dict]) -> List[str]:
        """Extract keywords from text and tags."""
        keywords = set()

        # Add tag names
        for tag in tags:
            keywords.add(tag["tag"].lower())

        # Simple keyword extraction from text (can be improved)
        import re
        words = re.findall(r'\w+', text.lower())
        keywords.update([w for w in words if len(w) > 3])

        return list(keywords)[:10]  # Limit to 10 keywords
```

---

## 4. Running Tests

```bash
cd mathesis-common

# Run all tests
pytest tests/ -v

# Run with coverage
pytest tests/ --cov=mathesis_core --cov-report=html

# Run specific module tests
pytest tests/test_analysis/ -v
pytest tests/test_vision/ -v
```

Expected output:
```
tests/test_prompts/test_analysis_prompts.py::test_tagging_prompt_includes_question_text PASSED
tests/test_vision/test_ocr_engine.py::test_ocr_engine_extracts_text PASSED
tests/test_analysis/test_dna_analyzer.py::test_dna_analyzer_extracts_tags PASSED

========== 15 passed in 2.34s ==========
```

---

## 5. Phase 2: Node 2 Refactoring (Week 2)

### 5.1 Install mathesis-common in Node 2

```bash
cd node2_q_dna/backend

# Add to requirements or pyproject.toml
# Option 1: Development mode (recommended)
pip install -e ../../../mathesis-common

# Option 2: Add to pyproject.toml
[tool.poetry.dependencies]
mathesis-common = {path = "../../../mathesis-common", develop = true}
```

### 5.2 Refactor OCRService

**Before:**

```python
# node2_q_dna/backend/app/services/ocr_service.py (BEFORE)
from app.services.ollama_service import ollama_service

class OCRService:
    async def extract_from_image_bytes(self, image_content: bytes):
        # ... 100 lines of OCR logic ...
```

**After:**

```python
# node2_q_dna/backend/app/services/ocr_service.py (AFTER)
from mathesis_core.vision import OCREngine
from mathesis_core.llm.clients import create_ollama_client
from app.core.config import settings

class OCRService:
    """
    Service layer - thin wrapper around OCREngine.
    Handles FastAPI integration and DB operations only.
    """

    def __init__(self):
        # Use mathesis_core standard client
        self.llm_client = create_ollama_client(
            base_url=settings.OLLAMA_URL,
            model=settings.OLLAMA_VISION_MODEL
        )
        # Delegate core logic to module
        self.ocr_engine = OCREngine(self.llm_client)

    async def extract_from_image_bytes(self, image_content: bytes):
        """API-compatible wrapper."""
        return await self.ocr_engine.extract(image_content)

    # Service-specific methods (DB operations)
    async def extract_and_save_to_db(self, db, image_content: bytes):
        """Extract + DB save (service-specific logic)."""
        result = await self.ocr_engine.extract(image_content)

        # Save to database (service layer responsibility)
        # ...

        return result

# Singleton instance
ocr_service = OCRService()
```

### 5.3 Refactor TaggingService

```python
# node2_q_dna/backend/app/services/tagging_service.py (AFTER)
from mathesis_core.analysis import DNAAnalyzer
from mathesis_core.llm.clients import create_ollama_client
from app.core.config import settings

class TaggingService:
    """Service layer - delegates to DNAAnalyzer."""

    def __init__(self):
        self.llm_client = create_ollama_client(
            base_url=settings.OLLAMA_URL,
            model=settings.OLLAMA_MODEL
        )
        self.dna_analyzer = DNAAnalyzer(self.llm_client)

    async def get_tag_recommendations(self, question_text: str):
        """API-compatible wrapper."""
        dna = await self.dna_analyzer.analyze(question_text)
        return dna["tags"]  # Return only tags for backward compatibility

    async def generate_metadata(self, question_text: str):
        """API-compatible wrapper."""
        dna = await self.dna_analyzer.analyze(question_text)
        return dna["metadata"]

tagging_service = TaggingService()
```

### 5.4 Test Backward Compatibility

```python
# node2_q_dna/backend/tests/integration/test_ocr_service.py
import pytest
from app.services.ocr_service import ocr_service

@pytest.mark.asyncio
async def test_ocr_service_backward_compatibility():
    """Ensure refactored service maintains same API."""
    test_image = load_test_image()

    result = await ocr_service.extract_from_image_bytes(test_image)

    # Same output format as before
    assert "text" in result
    assert "latex" in result
    assert "combined" in result
    assert "has_math" in result
```

---

## 6. Validation Checklist

### 6.1 Module Tests

- [ ] All unit tests pass in mathesis-common
- [ ] Test coverage > 90%
- [ ] No circular dependencies
- [ ] Modules have no DB/FastAPI dependencies

### 6.2 Node 2 Refactoring

- [ ] All existing tests pass
- [ ] API responses unchanged (backward compatible)
- [ ] Performance is equivalent or better
- [ ] No regression bugs

### 6.3 Documentation

- [ ] API documentation updated
- [ ] Migration guide complete
- [ ] Module specifications complete

---

## 7. Troubleshooting

### Issue: Import errors after refactoring

```python
# Solution: Ensure mathesis-common is in PYTHONPATH
import sys
sys.path.insert(0, "/path/to/mathesis-common")
```

### Issue: Tests fail due to missing LLM mocks

```python
# Solution: Use pytest fixtures for mock clients
@pytest.fixture
def mock_llm_client():
    client = Mock()
    client.generate = AsyncMock(return_value='{"result": "mocked"}')
    return client
```

### Issue: Async/await errors

```python
# Solution: Ensure all test functions are async
@pytest.mark.asyncio
async def test_my_async_function():
    result = await my_function()
    assert result is not None
```

---

## 8. Next Steps

After completing Phase 1 & 2:

1. Generate migration report: `pytest --cov-report=term-missing`
2. Update Node 1 to use same pattern (Phase 3)
3. Implement Node 7 using modules only (Phase 4)

---

## References

- [Modular Architecture](../architecture/02_MODULAR_ARCHITECTURE.md)
- [Module Specifications](../architecture/03_MODULE_SPECIFICATIONS.md)
- [TDD Best Practices](../TDD/)
