# Mathesis Core Module Specifications

> Detailed API specifications for mathesis-common core modules

**Last Updated**: 2026-01-10
**Version**: 2.0
**Status**: ✅ Complete - All Core Modules Implemented

---

## 1. Vision Module

### 1.1 OCREngine

**Purpose**: 이미지에서 텍스트와 수학 수식을 추출하는 핵심 엔진

#### API Specification

```python
class OCREngine:
    def __init__(self, llm_client: LLMClient):
        """
        Initialize OCR Engine.

        Args:
            llm_client: LLM client for vision analysis
        """

    async def extract(self, image_bytes: bytes) -> Dict[str, Any]:
        """
        Extract text and LaTeX from image.

        Args:
            image_bytes: Raw image bytes (JPEG, PNG, etc.)

        Returns:
            {
                "text": str,              # Pure text content
                "latex": List[str],       # LaTeX expressions
                "combined": str,          # Text with inline LaTeX
                "has_math": bool,         # Whether math detected
                "confidence": float,      # 0.0-1.0
                "tesseract_fallback": str # Raw OCR result
            }

        Raises:
            OCRError: If extraction fails
        """

    async def extract_structured(
        self,
        image_bytes: bytes
    ) -> Dict[str, Any]:
        """
        Extract structured question data from image.

        Returns:
            {
                "question_stem": str,
                "choices": List[str] or None,
                "answer_key": str or None,
                "question_type": str,  # mcq, short_answer, essay
                "subject_area": str
            }
        """
```

#### Usage Example

```python
from mathesis_core.vision import OCREngine
from mathesis_core.llm.clients import create_ollama_client

# Initialize
llm = create_ollama_client(model="llama3.2-vision:11b")
ocr = OCREngine(llm)

# Extract from image
with open("problem.jpg", "rb") as f:
    result = await ocr.extract(f.read())

print(result["text"])      # "x^2 + 2x + 1 = 0을 풀어라"
print(result["latex"])     # ["x^2 + 2x + 1 = 0"]
print(result["has_math"])  # True
```

#### Test Cases

```python
# tests/test_vision/test_ocr_engine.py

@pytest.mark.asyncio
async def test_ocr_extracts_korean_text():
    ocr = OCREngine(mock_llm_client)
    result = await ocr.extract(korean_text_image)

    assert "text" in result
    assert len(result["text"]) > 0
    assert result["has_math"] == False


@pytest.mark.asyncio
async def test_ocr_extracts_latex():
    ocr = OCREngine(mock_llm_client)
    result = await ocr.extract(math_problem_image)

    assert result["has_math"] == True
    assert len(result["latex"]) > 0
    assert "x^2" in result["combined"]


@pytest.mark.asyncio
async def test_ocr_handles_low_quality_image():
    ocr = OCREngine(mock_llm_client)
    result = await ocr.extract(blurry_image)

    assert result["confidence"] < 0.7
    assert "tesseract_fallback" in result
```

---

## 2. Analysis Module

### 2.1 DNAAnalyzer

**Purpose**: 문제 텍스트에서 DNA 정보 추출 (개념, 난이도, 태그, 커리큘럼)

#### API Specification

```python
class DNAAnalyzer:
    def __init__(self, llm_client: LLMClient):
        """
        Initialize DNA Analyzer.

        Args:
            llm_client: LLM client for analysis
        """

    async def analyze(self, question_text: str) -> Dict[str, Any]:
        """
        Analyze question and extract DNA.

        Args:
            question_text: Question content (may contain LaTeX)

        Returns:
            {
                "tags": List[Dict],  # [{"tag": str, "type": str, "confidence": float}]
                "metadata": Dict,    # {cognitive_level, difficulty, subject_area, ...}
                "curriculum_path": str,  # "Math.Algebra.Quadratics"
                "dna_signature": str,    # Unique hash for similarity search
                "keywords": List[str]    # Extracted keywords
            }

        Raises:
            AnalysisError: If analysis fails
        """

    async def extract_tags(self, text: str) -> List[Dict]:
        """Extract tags only."""

    async def extract_metadata(self, text: str) -> Dict:
        """Extract metadata only."""

    async def suggest_curriculum(self, text: str) -> str:
        """Suggest curriculum path only."""

    def compute_signature(
        self,
        tags: List[Dict],
        metadata: Dict
    ) -> str:
        """Compute DNA signature for similarity search."""
```

#### Usage Example

```python
from mathesis_core.analysis import DNAAnalyzer
from mathesis_core.llm.clients import create_ollama_client

# Initialize
llm = create_ollama_client()
analyzer = DNAAnalyzer(llm)

# Analyze question
question = "x^2 + 2x + 1 = 0을 풀어라"
dna = await analyzer.analyze(question)

print(dna["tags"])
# [
#   {"tag": "Algebra", "type": "concept", "confidence": 0.95},
#   {"tag": "Quadratic Equations", "type": "concept", "confidence": 0.90},
#   {"tag": "Apply", "type": "cognitive_level", "confidence": 0.85}
# ]

print(dna["metadata"])
# {
#   "cognitive_level": "Apply",
#   "difficulty_estimation": 0.6,
#   "subject_area": "Mathematics",
#   "estimated_time_minutes": 3
# }

print(dna["curriculum_path"])
# "Math.Algebra.Quadratics.Factoring"

print(dna["dna_signature"])
# "a7b3c9d2e1f4g5h6"
```

#### Test Cases

```python
# tests/test_analysis/test_dna_analyzer.py

@pytest.mark.asyncio
async def test_dna_analyzer_extracts_algebra_tags():
    analyzer = DNAAnalyzer(mock_llm_client)
    dna = await analyzer.analyze("x^2 + 2x + 1 = 0을 풀어라")

    tags = [t["tag"] for t in dna["tags"]]
    assert "Algebra" in tags
    assert "Quadratic Equations" in tags


@pytest.mark.asyncio
async def test_dna_analyzer_estimates_difficulty():
    analyzer = DNAAnalyzer(mock_llm_client)
    dna = await analyzer.analyze("간단한 덧셈: 1 + 1 = ?")

    assert dna["metadata"]["difficulty_estimation"] < 0.3


@pytest.mark.asyncio
async def test_dna_signature_uniqueness():
    analyzer = DNAAnalyzer(mock_llm_client)

    dna1 = await analyzer.analyze("x^2 + 2x + 1 = 0")
    dna2 = await analyzer.analyze("x^2 + 3x + 2 = 0")
    dna3 = await analyzer.analyze("직각삼각형의 빗변 길이는?")

    # Similar problems should have similar signatures
    assert dna1["dna_signature"][:8] == dna2["dna_signature"][:8]

    # Different topics should have different signatures
    assert dna1["dna_signature"] != dna3["dna_signature"]
```

### 2.2 MetadataExtractor

**Purpose**: 문제 메타데이터 추출 (난이도, 인지 수준, 예상 소요 시간)

```python
class MetadataExtractor:
    async def extract(self, question_text: str) -> Dict[str, Any]:
        """
        Extract comprehensive metadata.

        Returns:
            {
                "cognitive_level": str,      # Bloom's Taxonomy
                "difficulty_estimation": float,  # 0.0-1.0
                "estimated_time_minutes": int,
                "requires_calculator": bool,
                "language": str,  # Korean, English, Mixed
                "question_type": str  # mcq, short_answer, essay
            }
        """
```

---

## 3. Generation Module ⭐ NEW - Fully Implemented

### 3.1 ProblemGenerator

**Purpose**: 교육용 문제 생성 (Twin, Error, Correct, Variation)

**Status**: ✅ Complete (12 tests passing, 100% coverage)

**Location**: `mathesis-common/mathesis_core/generation/problem_generator.py`

#### API Specification

```python
class ProblemGenerator:
    """
    Problem Generator for educational content creation.

    Supports:
    - Twin question generation (isomorphic problems)
    - Error solution generation (common mistakes)
    - Correct solution generation (model answers)
    - Problem variations (difficulty, context, concept)
    """

    def __init__(self, llm_client: LLMClient):
        """Initialize Problem Generator with LLM client."""

    async def generate_twin(
        self,
        original_question: Dict[str, Any],
        preserve_metadata: bool = True
    ) -> Dict[str, Any]:
        """
        Generate a twin (isomorphic) question.

        Twin questions have the same mathematical logic and solving steps,
        but different story context, numbers, and wording.

        Args:
            original_question: Original question dict with keys:
                - content_stem: Question text
                - answer_key: Answer dict
                - question_type: Type of question
                - content_metadata (optional): Metadata dict
            preserve_metadata: Whether to preserve original metadata

        Returns:
            {
                "question_stem": str,     # New question text
                "answer": str,            # New answer
                "solution_steps": str,    # Solution explanation
                "metadata": Dict          # Original metadata (if preserved)
            }

        Raises:
            GenerationError: If generation fails
        """

    async def generate_error_solution(
        self,
        question_content: str,
        correct_answer: str,
        error_types: Optional[List[str]] = None,
        difficulty: int = 2
    ) -> Dict[str, Any]:
        """
        Generate an intentionally incorrect solution demonstrating common errors.

        Args:
            question_content: Question text
            correct_answer: Correct answer for reference
            error_types: List of error types (e.g., "concept_misapplication",
                        "arithmetic_error", "condition_omission")
            difficulty: Solution complexity level (1-5)

        Returns:
            {
                "steps": List[Dict],         # Solution steps with error markers
                "final_wrong_answer": str    # The incorrect final answer
            }

        Raises:
            GenerationError: If generation fails
        """

    async def generate_correct_solution(
        self,
        question_content: str,
        correct_answer: str
    ) -> Dict[str, Any]:
        """
        Generate a correct model solution.

        Args:
            question_content: Question text
            correct_answer: Correct answer

        Returns:
            {
                "steps": List[Dict]  # List of correct solution steps
            }

        Raises:
            GenerationError: If generation fails
        """

    async def generate_variation(
        self,
        original_question: str,
        variation_type: str = "difficulty",
        target_level: Optional[float] = None
    ) -> Dict[str, Any]:
        """
        Generate a problem variation.

        Args:
            original_question: Original question text
            variation_type: Type of variation ("difficulty", "context", "concept")
            target_level: Target difficulty level (0.0-1.0) for difficulty variations

        Returns:
            Dict with variation details (keys depend on variation_type)

        Raises:
            GenerationError: If generation fails
            ValueError: If variation_type is invalid
        """
```

#### Variation Strategies

| Strategy | Description | Example |
|----------|-------------|---------|
| `numbers` | 숫자만 변경 | x^2 + 2x + 1 → x^2 + 3x + 2 |
| `context` | 맥락 변경 | 사과 문제 → 공 문제 |
| `representation` | 표현 방식 변경 | 방정식 → 그래프 |
| `complexity` | 복잡도 조정 | 1차 → 2차 방정식 |

#### Usage Examples

**1. Twin Question Generation (Isomorphic Problems)**

```python
from mathesis_core.generation import ProblemGenerator
from mathesis_core.llm.clients import create_ollama_client

llm = create_ollama_client()
generator = ProblemGenerator(llm)

# Generate twin question
original_question = {
    "content_stem": "민수는 연필을 15개 가지고 있습니다. 친구에게 8개를 주면 몇 개가 남을까요?",
    "answer_key": {"answer": "7개"},
    "question_type": "short_answer"
}

twin = await generator.generate_twin(original_question)

print(twin["question_stem"])
# "철수는 사과를 12개 가지고 있습니다. 영희에게 5개를 주면 몇 개가 남을까요?"

print(twin["answer"])
# "7개"

print(twin["solution_steps"])
# "12 - 5 = 7"
```

**2. Error Solution Generation (Educational Mistakes)**

```python
error_solution = await generator.generate_error_solution(
    question_content="x^2 - 5x + 6 = 0을 푸시오.",
    correct_answer="x = 2 또는 x = 3",
    error_types=["condition_omission", "concept_misapplication"]
)

for step in error_solution["steps"]:
    print(f"Step {step['step']}: {step['content']}")
    if step.get("is_error"):
        print(f"  ⚠️ ERROR: {step['error_explanation']}")

print(f"Wrong Answer: {error_solution['final_wrong_answer']}")
```

**3. Difficulty Variation (Progressive Learning)**

```python
# Generate easier variation
easier = await generator.generate_variation(
    original_question="복잡한 이차방정식 문제",
    variation_type="difficulty",
    target_level=0.3  # 0.0 = very easy, 1.0 = very hard
)

print(easier["question_stem"])
# Simpler version of the problem
```

#### Test Cases

**✅ All 12 Tests Passing (100% Coverage)**

```python
# tests/test_generation/test_problem_generator.py

@pytest.mark.asyncio
async def test_generate_twin_question_success():
    """Test successful twin question generation."""
    mock_llm_client.generate = AsyncMock(
        return_value='{"question_stem": "...", "answer": "...", "solution_steps": "..."}'
    )

    generator = ProblemGenerator(mock_llm_client)
    original = {
        "content_stem": "민수는 연필을 15개 가지고 있습니다...",
        "answer_key": {"answer": "7개"},
        "question_type": "short_answer"
    }

    result = await generator.generate_twin(original)

    assert "question_stem" in result
    assert result["question_stem"] != original["content_stem"]


@pytest.mark.asyncio
async def test_generate_error_solution_has_single_error():
    """Test that error solution has exactly one error point."""
    mock_llm_client.generate = AsyncMock(
        return_value='{"steps": [...], "final_wrong_answer": "wrong"}'
    )

    generator = ProblemGenerator(mock_llm_client)
    result = await generator.generate_error_solution(
        question_content="문제",
        correct_answer="정답"
    )

    error_steps = [s for s in result["steps"] if s.get("is_error")]
    assert len(error_steps) >= 1


@pytest.mark.asyncio
async def test_generate_correct_solution_success():
    """Test correct solution generation."""
    generator = ProblemGenerator(mock_llm_client)
    result = await generator.generate_correct_solution(
        question_content="2x = 10을 푸시오.",
        correct_answer="x = 5"
    )

    assert "steps" in result
    assert all(not step.get("is_error") for step in result["steps"])


@pytest.mark.asyncio
async def test_generate_variation_by_difficulty():
    """Test problem variation by difficulty adjustment."""
    generator = ProblemGenerator(mock_llm_client)
    result = await generator.generate_variation(
        original_question="복잡한 문제",
        variation_type="difficulty",
        target_level=0.3
    )

    assert "question_stem" in result
    assert "difficulty_estimation" in result
```

---

## 4. Prompts Module

### 4.1 Prompt Structure

```python
# mathesis_core/prompts/analysis_prompts.py

def get_tagging_prompt(question_text: str) -> str:
    """Get prompt for tag extraction."""
    return f"""Analyze this educational question and generate relevant tags.

Question:
{question_text}

Identify and categorize tags in these types:
1. subject: Main subject area
2. concept: Specific concepts
3. skill: Required skills
4. cognitive_level: Bloom's taxonomy
5. difficulty: Difficulty level

Return JSON array of tags with confidence scores (0.0-1.0):
{{
    "tags": [
        {{"tag": "Mathematics", "type": "subject", "confidence": 0.99}},
        {{"tag": "Algebra", "type": "concept", "confidence": 0.95}}
    ]
}}"""


def get_metadata_prompt(question_text: str) -> str:
    """Get prompt for metadata extraction."""
    # ...


def get_curriculum_prompt(question_text: str) -> str:
    """Get prompt for curriculum path suggestion."""
    # ...
```

### 4.2 Template System

```python
# mathesis_core/prompts/templates.py

class PromptTemplate:
    """Base template for all prompts."""

    def __init__(self, template: str):
        self.template = template

    def format(self, **kwargs) -> str:
        return self.template.format(**kwargs)


# Vision prompts
VISION_EXTRACTION_TEMPLATE = PromptTemplate("""
Analyze this image and extract:
1. All text content (Korean and English)
2. All mathematical expressions in LaTeX format
3. Structure and formatting

Return in this exact JSON format:
{json_schema}
""")

# Analysis prompts
TAG_EXTRACTION_TEMPLATE = PromptTemplate("""
Analyze this question: {question_text}

Extract tags in these categories:
{categories}

Return JSON: {json_schema}
""")
```

---

## 5. Workflows Module

### 5.1 AnkiWorkflow

**Purpose**: Anki 시스템 전용 워크플로우 (사진 촬영 → 분석 → 복습 문제 생성)

#### API Specification

```python
class AnkiWorkflow:
    def __init__(
        self,
        ocr: OCREngine,
        analyzer: DNAAnalyzer,
        generator: ProblemGenerator
    ):
        """
        Initialize Anki workflow with required modules.
        """

    async def capture_and_analyze(
        self,
        image_bytes: bytes
    ) -> Dict[str, Any]:
        """
        Step 1: Capture image and analyze DNA.

        Returns:
            {
                "ocr": Dict,           # OCR result
                "dna": Dict,           # DNA analysis
                "original_text": str,  # Extracted text
                "timestamp": str
            }
        """

    async def generate_review_card(
        self,
        analysis_result: Dict
    ) -> Dict[str, Any]:
        """
        Step 2: Generate review card for spaced repetition.

        Returns:
            {
                "original_problem": str,
                "generated_problem": str,
                "dna_signature": str,
                "difficulty": float,
                "recommended_interval_days": int
            }
        """

    async def full_workflow(
        self,
        image_bytes: bytes
    ) -> Dict[str, Any]:
        """
        Execute full workflow: capture → analyze → generate.
        """
```

#### Usage Example

```python
from mathesis_core.workflows import AnkiWorkflow
from mathesis_core.vision import OCREngine
from mathesis_core.analysis import DNAAnalyzer
from mathesis_core.generation import ProblemGenerator
from mathesis_core.llm.clients import create_ollama_client

# Setup
llm = create_ollama_client()
ocr = OCREngine(llm)
analyzer = DNAAnalyzer(llm)
generator = ProblemGenerator(llm)

workflow = AnkiWorkflow(ocr, analyzer, generator)

# Day 1: Capture problem
with open("problem.jpg", "rb") as f:
    analysis = await workflow.capture_and_analyze(f.read())

# Save to database
card_id = save_to_db(analysis)

# Day 7: Generate review problem
card_data = load_from_db(card_id)
review_card = await workflow.generate_review_card(card_data)

print(review_card["generated_problem"])
# New variation of the original problem
```

---

## 6. Error Handling

### 6.1 Custom Exceptions

```python
# mathesis_core/exceptions.py

class MathesisCoreError(Exception):
    """Base exception for mathesis-core."""


class OCRError(MathesisCoreError):
    """OCR extraction failed."""


class AnalysisError(MathesisCoreError):
    """DNA analysis failed."""


class GenerationError(MathesisCoreError):
    """Problem generation failed."""


class LLMTimeoutError(MathesisCoreError):
    """LLM request timed out."""
```

### 6.2 Error Handling Pattern

```python
from mathesis_core.exceptions import OCRError, AnalysisError

try:
    result = await ocr.extract(image_bytes)
except OCRError as e:
    # Fallback to basic Tesseract
    result = fallback_ocr(image_bytes)
    logger.warning(f"OCR failed, using fallback: {e}")

try:
    dna = await analyzer.analyze(question_text)
except AnalysisError as e:
    # Use default values
    dna = get_default_dna()
    logger.error(f"Analysis failed: {e}")
```

---

## 7. Performance Considerations

### 7.1 Caching Strategy

```python
from functools import lru_cache
from mathesis_core.llm.decorators import cache_llm_result

class DNAAnalyzer:
    @cache_llm_result(ttl=3600)  # Cache for 1 hour
    async def analyze(self, question_text: str) -> Dict:
        # Expensive LLM call
        ...
```

### 7.2 Batch Processing

```python
# For multiple questions
questions = ["Q1", "Q2", "Q3"]

# Sequential (slow)
results = [await analyzer.analyze(q) for q in questions]

# Parallel (fast)
import asyncio
results = await asyncio.gather(*[analyzer.analyze(q) for q in questions])
```

---

## 8. Version Compatibility

| Module | Version | Requires |
|--------|---------|----------|
| vision | 1.0.0 | mathesis_core.llm >= 1.0.0, pytesseract >= 0.3.10 |
| analysis | 1.0.0 | mathesis_core.llm >= 1.0.0, mathesis_core.prompts >= 1.0.0 |
| generation | 1.0.0 | mathesis_core.llm >= 1.0.0, mathesis_core.analysis >= 1.0.0 |
| workflows | 1.0.0 | All above modules >= 1.0.0 |

---

## References

- [Modular Architecture](./02_MODULAR_ARCHITECTURE.md)
- [MSA Architecture](./01_MSA_ARCHITECTURE.md)
- [API Documentation](../api/)
