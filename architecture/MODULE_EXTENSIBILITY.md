# 모듈 확장성 가이드 (Module Extensibility Guide)

## 개요

Mathesis-Synapse는 **플러그인 아키텍처**를 지원하도록 설계되었습니다. 새로운 노드(MCP Server), MCP Tools, Flow를 손쉽게 추가할 수 있습니다.

---

## 1. 새 노드(MCP Server) 추가하기

### 시나리오: Node 7 "Quiz-Node" (자동 퀴즈 생성 엔진) 추가

#### Step 1: MCP Server 구조 생성

```bash
mkdir -p nodes/node7_quiz_node
cd nodes/node7_quiz_node
```

#### Step 2: MCP Server 구현

```python
# nodes/node7_quiz_node/server.py
from mcp.server import Server
from mcp.server.stdio import stdio_server
from pydantic import BaseModel
from typing import List, Dict, Any
import logging

logger = logging.getLogger(__name__)

app = Server("quiz-node-mcp")

# Tool 1: 퀴즈 생성
class GenerateQuizInput(BaseModel):
    topic: str
    question_count: int = 10
    difficulty: str = "medium"
    question_types: List[str] = ["multiple_choice", "true_false", "short_answer"]

class GenerateQuizOutput(BaseModel):
    quiz_id: str
    questions: List[Dict[str, Any]]
    total_points: int
    estimated_time_minutes: int

@app.tool()
async def generate_quiz(
    topic: str,
    question_count: int = 10,
    difficulty: str = "medium",
    question_types: List[str] = ["multiple_choice", "true_false"]
) -> dict:
    """
    주제에 맞는 퀴즈를 자동 생성

    Args:
        topic: 퀴즈 주제 (예: "2차 방정식")
        question_count: 문제 수
        difficulty: 난이도 (easy/medium/hard)
        question_types: 문제 유형 목록

    Returns:
        GenerateQuizOutput
    """
    # 1. Logic Node에서 관련 개념 가져오기
    from orchestrator.mcp_client import mcp_call

    concepts = await mcp_call(
        server="logic-engine-mcp",
        tool="get_prerequisites",
        params={"concept_id": topic, "depth": 1}
    )

    # 2. Q-DNA에서 유사 문제 검색
    similar_problems = await mcp_call(
        server="q-dna-mcp",
        tool="find_similar_dna_problems",
        params={
            "target_dna": {"concepts": [topic], "difficulty": difficulty},
            "limit": question_count * 2
        }
    )

    # 3. LLM으로 퀴즈 변형 생성
    from mathesis_common.llm_client import OllamaClient
    ollama = OllamaClient()

    questions = []
    for i, problem in enumerate(similar_problems["results"][:question_count]):
        prompt = f"""
다음 문제를 {question_types[i % len(question_types)]} 형태로 변형하세요:

원본 문제: {problem['content']}

변형 요구사항:
- 유형: {question_types[i % len(question_types)]}
- 난이도: {difficulty}
- 개념: {topic}

JSON 형식으로 응답:
{{
    "question": "문제 텍스트",
    "options": ["A", "B", "C", "D"],  // multiple_choice인 경우
    "correct_answer": "정답",
    "explanation": "해설",
    "points": 10
}}
        """

        response = await ollama.generate(prompt, format="json")
        questions.append(json.loads(response))

    quiz_id = f"quiz_{int(time.time())}"

    return {
        "quiz_id": quiz_id,
        "questions": questions,
        "total_points": sum(q["points"] for q in questions),
        "estimated_time_minutes": question_count * 2
    }


# Tool 2: 퀴즈 채점
@app.tool()
async def grade_quiz(
    quiz_id: str,
    student_answers: Dict[int, str]  # {question_index: answer}
) -> dict:
    """
    학생 답안 자동 채점
    """
    # DB에서 퀴즈 정답 조회
    correct_answers = await db.fetch(
        "SELECT question_index, correct_answer, points FROM quiz_questions WHERE quiz_id = $1",
        quiz_id
    )

    results = []
    total_score = 0

    for idx, correct in enumerate(correct_answers):
        student_answer = student_answers.get(idx, "")
        is_correct = student_answer.lower() == correct["correct_answer"].lower()

        if is_correct:
            total_score += correct["points"]

        results.append({
            "question_index": idx,
            "is_correct": is_correct,
            "student_answer": student_answer,
            "correct_answer": correct["correct_answer"]
        })

    return {
        "quiz_id": quiz_id,
        "total_score": total_score,
        "max_score": sum(c["points"] for c in correct_answers),
        "percentage": round(total_score / sum(c["points"] for c in correct_answers) * 100, 2),
        "results": results
    }


# Tool 3: 적응형 퀴즈 생성 (학생 수준에 맞춤)
@app.tool()
async def generate_adaptive_quiz(
    student_id: str,
    topic: str,
    target_mastery: float = 0.8
) -> dict:
    """
    학생의 현재 숙련도에 맞춰 적응형 퀴즈 생성
    """
    # Lab Node에서 학생 히트맵 가져오기
    heatmap = await mcp_call(
        server="lab-node-mcp",
        tool="update_student_heatmap",
        params={"student_id": student_id}
    )

    current_mastery = heatmap["heatmap_data"].get(topic, 0.5)

    # 현재 수준보다 10% 어려운 난이도로 설정
    if current_mastery < 0.5:
        difficulty = "easy"
    elif current_mastery < 0.7:
        difficulty = "medium"
    else:
        difficulty = "hard"

    # 퀴즈 생성
    quiz = await generate_quiz(
        topic=topic,
        question_count=10,
        difficulty=difficulty
    )

    return {
        **quiz,
        "current_mastery": current_mastery,
        "target_mastery": target_mastery,
        "difficulty": difficulty
    }


# Entry point
async def main():
    async with stdio_server() as (read, write):
        await app.run(read, write, app.create_initialization_options())

if __name__ == "__main__":
    import asyncio
    asyncio.run(main())
```

#### Step 3: 모듈 설정 파일 생성

```yaml
# nodes/node7_quiz_node/module.yaml
module:
  name: "quiz-node"
  version: "1.0.0"
  description: "자동 퀴즈 생성 및 채점 엔진"

  mcp_server:
    name: "quiz-node-mcp"
    command: "python"
    args: ["nodes/node7_quiz_node/server.py"]

  tools:
    - name: generate_quiz
      description: "주제에 맞는 퀴즈 자동 생성"
      input_schema: GenerateQuizInput
      output_schema: GenerateQuizOutput

    - name: grade_quiz
      description: "학생 답안 자동 채점"

    - name: generate_adaptive_quiz
      description: "학생 수준에 맞춘 적응형 퀴즈 생성"

  dependencies:
    mcp_servers: ["logic-engine-mcp", "q-dna-mcp", "lab-node-mcp"]
    common_modules: ["OllamaClient", "DatabaseClient"]

  database:
    tables:
      - name: quizzes
        schema: |
          CREATE TABLE quizzes (
            quiz_id VARCHAR(64) PRIMARY KEY,
            topic VARCHAR(100),
            difficulty VARCHAR(20),
            created_at TIMESTAMP DEFAULT NOW()
          );

      - name: quiz_questions
        schema: |
          CREATE TABLE quiz_questions (
            id SERIAL PRIMARY KEY,
            quiz_id VARCHAR(64) REFERENCES quizzes(quiz_id),
            question_index INT,
            question_text TEXT,
            correct_answer TEXT,
            points INT
          );
```

#### Step 4: 자동 등록 시스템

Orchestrator가 자동으로 새 모듈을 발견하고 등록하도록 개선:

```python
# orchestrator/module_registry.py
import yaml
import glob
from typing import Dict, List, Any
from pathlib import Path

class ModuleRegistry:
    """
    모든 노드 모듈을 자동으로 발견하고 등록
    """

    def __init__(self, nodes_dir: str = "nodes"):
        self.nodes_dir = Path(nodes_dir)
        self.modules: Dict[str, Dict[str, Any]] = {}

    def discover_modules(self) -> List[Dict[str, Any]]:
        """
        nodes/ 디렉토리에서 모든 module.yaml 파일 검색
        """
        discovered = []

        for module_yaml in self.nodes_dir.glob("*/module.yaml"):
            with open(module_yaml, 'r', encoding='utf-8') as f:
                config = yaml.safe_load(f)

            module_name = config['module']['name']
            self.modules[module_name] = config['module']

            discovered.append({
                "name": module_name,
                "path": module_yaml.parent,
                "config": config['module']
            })

        return discovered

    def get_server_configs(self) -> Dict[str, Dict[str, Any]]:
        """
        Orchestrator의 MCPClientManager에 전달할 서버 설정 생성
        """
        server_configs = {}

        for module_name, config in self.modules.items():
            mcp_config = config['mcp_server']
            server_configs[mcp_config['name']] = {
                "command": mcp_config['command'],
                "args": mcp_config['args']
            }

        return server_configs

    def validate_dependencies(self) -> List[str]:
        """
        모든 모듈의 의존성 검증
        """
        errors = []

        for module_name, config in self.modules.items():
            if 'dependencies' not in config:
                continue

            # MCP Server 의존성 체크
            for dep_server in config['dependencies'].get('mcp_servers', []):
                if dep_server not in [m['mcp_server']['name'] for m in self.modules.values()]:
                    errors.append(f"{module_name}: Missing dependency {dep_server}")

        return errors

    def setup_databases(self):
        """
        모든 모듈의 데이터베이스 스키마 자동 생성
        """
        import asyncpg

        async def create_schemas():
            conn = await asyncpg.connect(DATABASE_URL)

            for module_name, config in self.modules.items():
                if 'database' not in config:
                    continue

                for table_def in config['database']['tables']:
                    try:
                        await conn.execute(table_def['schema'])
                        logger.info(f"Created table {table_def['name']} for {module_name}")
                    except Exception as e:
                        logger.error(f"Failed to create {table_def['name']}: {e}")

            await conn.close()

        import asyncio
        asyncio.run(create_schemas())


# 사용 예시
registry = ModuleRegistry()
modules = registry.discover_modules()

print(f"Discovered {len(modules)} modules:")
for module in modules:
    print(f"  - {module['name']}: {module['config']['description']}")

# Orchestrator에 통합
server_configs = registry.get_server_configs()
# → MCPClientManager에 자동 전달
```

#### Step 5: Orchestrator 수정 (자동 로딩)

```python
# orchestrator/mcp_client_manager.py (개선 버전)
from module_registry import ModuleRegistry

class MCPClientManager:
    def __init__(self):
        self.servers: Dict[str, ClientSession] = {}
        self.tool_registry: Dict[str, Dict[str, Any]] = {}

        # 🆕 자동 모듈 발견
        self.registry = ModuleRegistry()
        self.modules = self.registry.discover_modules()

        # 의존성 검증
        errors = self.registry.validate_dependencies()
        if errors:
            raise RuntimeError(f"Dependency errors: {errors}")

    async def connect_all_servers(self):
        """
        registry에서 발견한 모든 서버에 자동 연결
        """
        server_configs = self.registry.get_server_configs()

        for server_name, config in server_configs.items():
            try:
                server_params = StdioServerParameters(
                    command=config["command"],
                    args=config["args"]
                )

                async with stdio_client(server_params) as (read, write):
                    async with ClientSession(read, write) as session:
                        await session.initialize()
                        tools_result = await session.list_tools()

                        self.servers[server_name] = session
                        self.tool_registry[server_name] = {
                            tool.name: tool for tool in tools_result.tools
                        }

                        logger.info(f"✓ Connected to {server_name}: {len(tools_result.tools)} tools")

            except Exception as e:
                logger.error(f"✗ Failed to connect to {server_name}: {e}")
```

---

## 2. 기존 노드에 새 Tool 추가하기

### 시나리오: Gen-Node에 `generate_video_explanation` Tool 추가

#### Step 1: Tool 함수 추가

```python
# nodes/node3_gen_node/server.py

@app.tool()
async def generate_video_explanation(
    concept_id: str,
    duration_seconds: int = 300,
    animation_style: str = "manim"  # manim, motion_canvas
) -> dict:
    """
    개념 설명 동영상 스크립트 및 애니메이션 코드 생성

    Args:
        concept_id: 설명할 개념 ID
        duration_seconds: 동영상 길이 (초)
        animation_style: 애니메이션 스타일

    Returns:
        {
            "script": "음성 스크립트",
            "animation_code": "Manim 파이썬 코드",
            "scene_breakdown": [...]
        }
    """
    # 1. Logic Node에서 개념 정보 가져오기
    concept_info = await mcp_call(
        server="logic-engine-mcp",
        tool="get_prerequisites",
        params={"concept_id": concept_id, "depth": 1}
    )

    # 2. LLM으로 스크립트 생성
    prompt = f"""
개념: {concept_info['title']}
설명: {concept_info['description']}

{duration_seconds}초 길이의 설명 동영상 스크립트를 작성하세요.
구성: 도입(10%) → 핵심 설명(60%) → 예시(20%) → 정리(10%)
    """

    script = await ollama.generate(prompt)

    # 3. Manim 애니메이션 코드 생성
    if animation_style == "manim":
        animation_code = await generate_manim_code(script, concept_id)

    return {
        "script": script,
        "animation_code": animation_code,
        "scene_breakdown": parse_scenes(script),
        "estimated_render_time": duration_seconds / 10
    }
```

#### Step 2: module.yaml 업데이트

```yaml
# nodes/node3_gen_node/module.yaml
module:
  name: "gen-node"
  version: "1.1.0"  # 버전 업

  tools:
    # 기존 Tools
    - name: generate_picket_problem
    - name: create_explanation_step
    - name: render_math_latex

    # 🆕 새 Tool
    - name: generate_video_explanation
      description: "개념 설명 동영상 스크립트 및 애니메이션 생성"
      dependencies:
        - manim
        - pydub
```

#### Step 3: 자동 반영

Orchestrator 재시작 시 자동으로 새 Tool 인식:

```bash
# Orchestrator restart
python orchestrator/main.py

# 로그 출력:
# ✓ Connected to gen-node-mcp: 4 tools  # 3개 → 4개로 증가
# ✓ New tool discovered: generate_video_explanation
```

---

## 3. 새 Flow 정의하기

### 시나리오: "개념_동영상_생성" Flow 추가

```yaml
# flows/개념_동영상_생성.yaml
flow:
  name: "개념_동영상_생성"
  description: "학생이 어려워하는 개념에 대한 맞춤형 설명 동영상 생성"
  version: "1.0"

  inputs:
    - name: student_id
      type: string
      required: true
    - name: concept_id
      type: string
      required: true
    - name: duration_seconds
      type: int
      default: 300

  steps:
    # Step 1: 학생의 현재 이해도 파악
    - id: check_understanding
      name: "학생 이해도 확인"
      mcp_call:
        server: lab-node-mcp
        tool: update_student_heatmap
        params:
          student_id: $inputs.student_id
      output: heatmap

    # Step 2: 선수 지식 확인
    - id: get_prerequisites
      name: "선수 지식 확인"
      mcp_call:
        server: logic-engine-mcp
        tool: get_prerequisites
        params:
          concept_id: $inputs.concept_id
          depth: 2
      output: prerequisites
      depends_on: [check_understanding]

    # Step 3: 동영상 스크립트 생성 (🆕 새 Tool 사용)
    - id: generate_video
      name: "동영상 생성"
      mcp_call:
        server: gen-node-mcp
        tool: generate_video_explanation
        params:
          concept_id: $inputs.concept_id
          duration_seconds: $inputs.duration_seconds
      output: video
      depends_on: [get_prerequisites]

    # Step 4: 학습 활동 로그 기록
    - id: log_video_view
      name: "시청 기록 저장"
      mcp_call:
        server: lab-node-mcp
        tool: log_activity
        params:
          student_id: $inputs.student_id
          activity_type: "video_watch"
          metadata:
            concept_id: $inputs.concept_id
            video_script: $video.script
      depends_on: [generate_video]

  outputs:
    video_script: $video.script
    animation_code: $video.animation_code
    prerequisites_covered: $prerequisites.prerequisite_tree
```

### Flow 자동 등록

```python
# orchestrator/flow_loader.py
import glob
import yaml

class FlowLoader:
    def __init__(self, flows_dir: str = "flows"):
        self.flows_dir = flows_dir
        self.flows = {}

    def load_all_flows(self):
        """
        flows/ 디렉토리의 모든 YAML 자동 로드
        """
        for yaml_path in glob.glob(f"{self.flows_dir}/*.yaml"):
            with open(yaml_path, 'r', encoding='utf-8') as f:
                flow_def = yaml.safe_load(f)

            flow_name = flow_def['flow']['name']
            self.flows[flow_name] = {
                "path": yaml_path,
                "definition": flow_def
            }

            logger.info(f"✓ Loaded flow: {flow_name}")

        return self.flows

# 사용
loader = FlowLoader()
flows = loader.load_all_flows()

# 자연어 명령 → Flow 매칭
user_command = "미적분 개념을 설명하는 동영상 만들어줘"
matched_flow = match_command_to_flow(user_command, flows)
# → "개념_동영상_생성" Flow 실행
```

---

## 4. 확장성 체크리스트

새 모듈 추가 시:

### ✅ 쉬운 것 (코드 변경 없음)
- [ ] 새 MCP Server (노드) 추가 → `module.yaml` 작성만
- [ ] 기존 노드에 Tool 추가 → 함수 작성 + `@app.tool()` 데코레이터
- [ ] 새 Flow 정의 → YAML 파일 추가
- [ ] 데이터베이스 스키마 추가 → `module.yaml`에 DDL 작성

### ⚠️ 약간의 수정 필요
- [ ] Orchestrator 재시작 (자동 발견)
- [ ] 의존성 패키지 설치 (`requirements.txt` 업데이트)

### ❌ 현재 설계에서 어려운 것
- [ ] MCP Tools 간 직접 통신 (현재는 Orchestrator 경유 필수)
- [ ] 실시간 Tool 핫 리로드 (서버 재시작 필요)
- [ ] Tool 버전 관리 (스키마 변경 시 호환성)

---

## 5. 실전 확장 예시

### 예시 1: "Gamification Node" 추가

**목적**: 학습 활동을 게임화 (포인트, 배지, 리더보드)

```yaml
# nodes/node8_gamification/module.yaml
module:
  name: "gamification-node"
  tools:
    - name: award_points
      description: "학생에게 포인트 부여"
    - name: unlock_badge
      description: "배지 해금"
    - name: get_leaderboard
      description: "리더보드 조회"
```

**Flow 통합**:
```yaml
# flows/문제_풀이_게임화.yaml
steps:
  - id: solve_problem
    mcp_call:
      server: gen-node-mcp
      tool: generate_picket_problem

  # 🆕 문제 풀면 포인트 지급
  - id: award_points
    mcp_call:
      server: gamification-node-mcp
      tool: award_points
      params:
        student_id: $inputs.student_id
        points: 100
        reason: "문제 해결"
```

### 예시 2: "Parental-Dashboard Node" 추가

**목적**: 학부모용 대시보드

```python
# nodes/node9_parental_dashboard/server.py

@app.tool()
async def get_child_progress(
    parent_id: str,
    period_days: int = 30
) -> dict:
    """
    자녀의 학습 진행 상황 요약
    """
    # 1. 부모와 연결된 학생 조회
    children = await db.fetch(
        "SELECT student_id FROM parent_child_relations WHERE parent_id = $1",
        parent_id
    )

    progress_reports = []
    for child in children:
        # 2. Lab Node에서 학습 데이터 가져오기
        heatmap = await mcp_call(
            server="lab-node-mcp",
            tool="update_student_heatmap",
            params={"student_id": child["student_id"]}
        )

        # 3. Report Node에서 간단한 리포트 생성
        report = await mcp_call(
            server="report-node-mcp",
            tool="generate_typst_report",
            params={
                "student_id": child["student_id"],
                "report_type": "weekly"
            }
        )

        progress_reports.append({
            "student_id": child["student_id"],
            "mastery_avg": sum(heatmap["heatmap_data"].values()) / len(heatmap["heatmap_data"]),
            "report_url": report["pdf_url"]
        })

    return {
        "parent_id": parent_id,
        "children_count": len(children),
        "reports": progress_reports
    }
```

---

## 6. 모듈 간 통신 패턴

### Pattern 1: Tool Chaining (권장)

```yaml
# Flow YAML에서 명시적 체이닝
steps:
  - id: step1
    mcp_call:
      server: node-a
      tool: tool_a
    output: result_a

  - id: step2
    mcp_call:
      server: node-b
      tool: tool_b
      params:
        input_from_a: $result_a.field  # 이전 결과 참조
```

### Pattern 2: 이벤트 기반 통신 (향후 확장)

```python
# 미래 개선: 이벤트 버스 추가
from orchestrator.event_bus import EventBus

@app.tool()
async def log_activity(student_id: str, activity_type: str):
    # 활동 로그 저장
    await db.execute(...)

    # 🆕 이벤트 발행
    await EventBus.publish("student.activity.logged", {
        "student_id": student_id,
        "activity_type": activity_type
    })

# 다른 노드에서 구독
@EventBus.subscribe("student.activity.logged")
async def on_activity_logged(event):
    # Gamification Node가 자동으로 포인트 지급
    if event["activity_type"] == "problem_solved":
        await award_points(event["student_id"], 50)
```

---

## 7. 확장성 로드맵

### Phase 1: 현재 (구현 중)
- ✅ MCP Protocol 기반 아키텍처
- ✅ 6개 코어 노드
- ✅ Flow YAML 시스템

### Phase 2: 단기 (1-2개월)
- 🔄 Module Registry 자동 발견
- 🔄 Hot Reload (서버 재시작 없이 Tool 추가)
- 🔄 Tool 버전 관리 시스템

### Phase 3: 중기 (3-6개월)
- 📅 이벤트 기반 통신
- 📅 분산 MCP Servers (Kubernetes)
- 📅 Tool Marketplace (커뮤니티 기여)

### Phase 4: 장기 (6개월+)
- 📅 AI가 Flow를 자동 생성
- 📅 Tool 조합 최적화 (강화학습)
- 📅 멀티모달 Tools (음성, 이미지, 비디오)

---

## 결론

**현재 설계는 높은 확장성을 가지고 있습니다:**

✅ **장점:**
- MCP Protocol 표준 준수 → 다른 시스템과 호환
- Flow YAML → 비개발자도 워크플로우 수정 가능
- 명확한 모듈 경계 → 독립 개발/배포
- mathesis-common → 코드 재사용

⚠️ **개선 필요:**
- Module Registry 시스템 추가
- Tool 스키마 자동 검증
- 의존성 그래프 시각화

**추천 다음 단계:**
1. Module Registry 구현 (이 문서의 코드 활용)
2. 첫 확장 모듈(Quiz-Node) 프로토타입
3. 확장성 테스트 (10개 노드까지 확장)

---

**생성 일시**: 2026-01-08
**문서 버전**: 1.0
**관련 문서**: 00_MCP_SYSTEM_DESIGN.md, LLM_ORCHESTRATOR.md
