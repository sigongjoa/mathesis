# Mathesis Platform - System Integration Overview

> 노드 간 통합 패턴 및 데이터 플로우 가이드

**작성일**: 2026-01-10
**버전**: 1.0
**상태**: Draft

---

## 📋 목차

1. [시스템 아키텍처](#시스템-아키텍처)
2. [노드별 역할과 책임](#노드별-역할과-책임)
3. [통신 프로토콜](#통신-프로토콜)
4. [데이터 플로우 패턴](#데이터-플로우-패턴)
5. [통합 시나리오](#통합-시나리오)

---

## 시스템 아키텍처

### Overall Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    Mathesis Platform                             │
│                                                                   │
│  ┌────────────────────────────────────────────────────────┐    │
│  │         Node 0: Student Hub (Master Orchestrator)      │    │
│  │         - SSOT for student data                        │    │
│  │         - MCP Client for all nodes                     │    │
│  │         - Workflow orchestration                       │    │
│  │         - Profile aggregation                          │    │
│  └──────────────┬──────────────────────────────────────────┘    │
│                 │ MCP Protocol                                   │
│                 ├─────────────────┬──────────────┬──────────────┤
│                 ▼                 ▼              ▼              ▼
│  ┌──────────────────┐  ┌──────────────┐  ┌────────────┐  ┌──────────┐
│  │   Node 1         │  │   Node 2     │  │  Node 4    │  │ Node 5   │
│  │   Logic Engine   │  │   Q-DNA      │  │  Lab Node  │  │ Q-Metrics│
│  │   (Neo4j)        │  │   (BKT/IRT)  │  │ (Tracking) │  │ (Reports)│
│  └──────────────────┘  └──────────────┘  └────────────┘  └──────────┘
│                                                                   │
│  ┌──────────────────┐  ┌──────────────┐                         │
│  │   Node 6         │  │   Node 7     │                         │
│  │   School Info    │  │   Error Note │                         │
│  │   (RAG)          │  │   (Anki)     │                         │
│  └──────────────────┘  └──────────────┘                         │
│                                                                   │
│  ┌────────────────────────────────────────────────────────┐    │
│  │         mathesis-common (Shared Modules)                │    │
│  │  - Vision (OCR)                                         │    │
│  │  - Analysis (DNA)                                       │    │
│  │  - Generation (Problem Generator)                       │    │
│  │  - Prompts (Templates)                                  │    │
│  └────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
```

---

## 노드별 역할과 책임

### Node 0: Student Hub (Master Orchestrator)

**Primary Role**: 중앙 조정자 및 SSOT

**Responsibilities**:
- 학생 기본 정보 관리 (이름, 학년, 학교)
- 다른 노드에서 데이터 수집 및 통합
- 워크플로우 실행 및 조정
- 프로필 캐싱 (Redis)

**API Endpoints**:
- `POST /api/v1/students/` - 학생 생성
- `GET /api/v1/students/{id}/profile` - 통합 프로필 조회
- `POST /api/v1/workflows/diagnostic` - 진단 워크플로우 실행
- `POST /api/v1/workflows/learning-path` - 학습 경로 생성

**Integration Points**:
- MCP Client로 모든 노드와 통신
- Redis 캐싱으로 성능 최적화

---

### Node 1: Logic Engine (교육 이론)

**Primary Role**: 교육 이론 지식 베이스

**Responsibilities**:
- Bloom's Taxonomy 분류
- 교육과정 계층 구조 (Neo4j)
- 개념 간 관계 그래프
- 학습 경로 추천

**API Endpoints**:
- `GET /api/v1/curriculum/tree` - 교육과정 트리
- `POST /api/v1/concepts/relate` - 개념 관계 생성
- `GET /api/v1/learning-path/{concept}` - 학습 경로 추천

**Integration Points**:
- gRPC (port 50051) - 고속 통신
- MCP - 표준 인터페이스
- Neo4j - 지식 그래프

---

### Node 2: Q-DNA (문제 은행)

**Primary Role**: 문제 관리 및 적응형 추천

**Responsibilities**:
- 문제 DNA 분석 (난이도, 개념, 태그)
- BKT로 학생 숙련도 추적
- IRT로 문제 난이도 매칭
- Twin 문제 생성 (mathesis_core)

**API Endpoints**:
- `POST /api/v1/questions/` - 문제 생성 (자동 DNA 추출)
- `GET /api/v1/questions/recommend/{student_id}` - 문제 추천
- `POST /api/v1/questions/{id}/twin` - Twin 문제 생성
- `POST /api/v1/analytics/attempt` - 학습 시도 기록 (BKT 업데이트)

**Integration Points**:
- mathesis_core.analysis.DNAAnalyzer
- mathesis_core.generation.ProblemGenerator
- PostgreSQL (ltree, JSONB)

---

### Node 4: Lab Node (학습 활동 추적)

**Primary Role**: 학습 활동 로깅 및 분석

**Responsibilities**:
- 모든 학습 활동 기록
- 시간 기반 분석 (TimescaleDB)
- 히트맵 생성 (교육과정, 시간, 개념)
- 약점 탐지

**API Endpoints**:
- `POST /api/v1/activities/` - 활동 로깅
- `GET /api/v1/heatmaps/curriculum/{student_id}` - 교육과정 히트맵
- `GET /api/v1/progress/weak-points/{student_id}` - 약점 분석
- `GET /api/v1/patterns/{student_id}` - 학습 패턴 분석

**Integration Points**:
- TimescaleDB - 시계열 데이터
- Redis - 실시간 캐싱
- Node 2 연동 - BKT 업데이트 트리거

---

### Node 5: Q-Metrics (시험 분석)

**Primary Role**: 시험 분석 및 리포트 생성

**Responsibilities**:
- PDF 시험지 OCR
- 문제 난이도 분석
- 교육과정 정렬
- Typst 기반 PDF 리포트

**API Endpoints**:
- `POST /api/v1/exams/analyze` - 시험 분석
- `POST /api/v1/textbooks/analyze` - 교과서 분석
- `POST /api/v1/reports/generate` - 리포트 생성

**Integration Points**:
- gRPC (port 50052)
- PaddleOCR - 텍스트 추출
- Typst - PDF 생성

---

### Node 6: School Info (학교 정보)

**Primary Role**: 학교 정보 RAG 시스템

**Responsibilities**:
- 학교 정보 크롤링
- ChromaDB 벡터 저장
- HyDE 기반 검색
- 의미론적 질의 응답

**API Endpoints**:
- `POST /api/v1/crawl/school/{code}` - 학교 크롤링
- `POST /api/v1/query` - RAG 질의
- `GET /api/v1/export/json` - JSON 내보내기

**Integration Points**:
- Playwright - 웹 크롤링
- ChromaDB - 벡터 DB
- mathesis_core.crawlers

---

### Node 7: Error Note (오답노트)

**Primary Role**: 오답 분석 및 복습 관리

**Responsibilities**:
- 메타인지 5단계 프레임워크
- Anki SM-2 복습 스케줄링
- LLM 기반 오답 분석 (DNA 통합)
- 변형 문제 생성

**API Endpoints**:
- `POST /api/v1/error-notes/` - 오답노트 생성
- `POST /api/v1/reviews/` - 복습 결과 제출
- `GET /api/v1/reviews/due` - 오늘 복습할 항목
- `GET /api/v1/analytics/patterns` - 오답 패턴 분석

**Integration Points**:
- mathesis_core.analysis.DNAAnalyzer - 개념 추출
- mathesis_core.generation.ProblemGenerator - 변형 생성
- Celery + Redis - 알림 스케줄링
- PostgreSQL - 오답노트 저장

---

## 통신 프로토콜

### MCP (Model Context Protocol)

**용도**: 표준 노드 간 통신

**특징**:
- Tool-based communication
- JSON-RPC 2.0
- Stdio 기반 (개발), TCP/HTTP (프로덕션)

**예시**:
```python
# Node 0에서 Node 2 호출
from mcp_client import MCPClientManager

mcp = MCPClientManager()

# BKT 숙련도 조회
mastery = await mcp.call("q-dna", "get_student_mastery", {
    "student_id": "student_123",
    "skill_id": "derivative"
})

# 문제 추천
questions = await mcp.call("q-dna", "recommend_questions", {
    "student_id": "student_123",
    "count": 10,
    "curriculum_path": "고등수학.미적분"
})
```

### gRPC

**용도**: 고속 통신이 필요한 경우

**노드**:
- Node 1 (Logic Engine): port 50051
- Node 5 (Q-Metrics): port 50052

**예시**:
```python
# Node 0에서 Node 1 gRPC 호출
import grpc
from proto import logic_pb2, logic_pb2_grpc

channel = grpc.insecure_channel('localhost:50051')
stub = logic_pb2_grpc.LogicServiceStub(channel)

response = stub.GetLearningPath(logic_pb2.ConceptRequest(
    concept="도함수",
    depth=3
))
```

### REST API

**용도**: 외부 클라이언트 (프론트엔드, 모바일)

**특징**:
- FastAPI
- OpenAPI/Swagger 문서
- JWT 인증 (계획)

---

## 데이터 플로우 패턴

### Pattern 1: Orchestrated Workflow

Node 0이 여러 노드를 순차적으로 호출

```
Frontend → Node 0 (Orchestrator)
                ├→ Node 2 (Q-DNA): 문제 추천
                ├→ Node 4 (Lab): 활동 로깅
                ├→ Node 7 (Error): 오답 기록
                └→ Node 5 (Metrics): 리포트 생성
```

### Pattern 2: Event-Driven

Node 4의 활동 로깅이 다른 노드 업데이트 트리거

```
Node 4 (Activity logged)
    └→ Event: "attempt_completed"
        ├→ Node 2: BKT 업데이트
        ├→ Node 7: 오답 자동 생성 (오답 시)
        └→ Node 0: 프로필 캐시 무효화
```

### Pattern 3: Aggregation

Node 0이 여러 노드에서 데이터 수집 후 통합

```
Frontend → Node 0: GET /students/{id}/profile
               ├→ Node 2: 숙련도 데이터
               ├→ Node 4: 활동 통계
               ├→ Node 7: 오답 패턴
               └→ Aggregate → Response
```

---

## 통합 시나리오

### Scenario 1: 주간 학습 진단

**상세 문서**: [01_WEEKLY_DIAGNOSTIC.md](./01_WEEKLY_DIAGNOSTIC.md)

매주 월요일, 학생의 현재 숙련도를 파악하고 맞춤형 진단 문제를 제공하여 학습 방향을 설정합니다.

```
1. Frontend → Node 0: POST /workflows/diagnostic
2. Node 0 → Node 2: 현재 숙련도 조회 (BKT)
3. Node 0 → Node 2: 문제 10개 추천 (IRT)
4. Frontend → Node 4: 각 문제 풀이 활동 로깅
5. Node 4 → Node 2: BKT 업데이트 (자동)
6. Node 0 → Node 5: 진단 리포트 생성
7. Frontend ← Node 0: 리포트 PDF 반환
```

**관련 노드**: Node 0, Node 2, Node 4, Node 5

---

### Scenario 2: 오답 복습 플로우

**상세 문서**: [02_ERROR_REVIEW_FLOW.md](./02_ERROR_REVIEW_FLOW.md)

학생이 문제를 틀렸을 때부터 메타인지 분석, Anki 스케줄링, 복습 알림, 변형 문제 생성까지 전체 오답 관리 프로세스를 자동화합니다.

```
1. Student 문제 틀림 → Node 4: 활동 로깅 (is_correct=false)
2. Node 4 → Node 7: 오답노트 자동 생성 트리거
3. Node 7: LLM 분석 (DNAAnalyzer로 개념 추출)
4. Node 7: Anki 스케줄링 (다음 복습 시점 계산)
5. Celery → Node 7: 복습 알림 발송 (6일 후)
6. Teacher 복습 → Node 7: 복습 결과 제출
7. Node 7 → Node 2: 변형 문제 요청 (ProblemGenerator)
8. Node 7: Anki 간격 재계산
```

**관련 노드**: Node 0, Node 2, Node 4, Node 7

---

### Scenario 3: 개인화 학습 경로

**상세 문서**: [03_PERSONALIZED_LEARNING_PATH.md](./03_PERSONALIZED_LEARNING_PATH.md)

학생의 약점을 히트맵으로 분석하고, 교육 이론 기반 선수 학습 관계를 고려하여 맞춤형 학습 경로를 자동 생성합니다.

```
1. Student → Node 0: 학습 경로 생성 요청
2. Node 0 → Node 4: 히트맵 조회 (약점 탐지)
3. Node 0 → Node 2: BKT 숙련도 조회
4. Node 0 → Node 1: 선수 학습 관계 조회 (Neo4j)
5. Node 0: 위상 정렬로 최적 경로 생성
6. Node 0 → Node 2: 각 단계별 문제 추천
7. Student ← Node 0: 학습 경로 반환
```

**관련 노드**: Node 0, Node 1, Node 2, Node 4

---

### Scenario 4: 클래스 관리 (선생님 관점)

**상세 문서**: [04_CLASS_MANAGEMENT.md](./04_CLASS_MANAGEMENT.md)

선생님이 반 전체 학생의 학습 현황을 실시간으로 파악하고, 공통 약점 기반 수업 계획을 수립합니다.

```
1. Teacher → Node 0: GET /class/{class_id}/analytics
2. Node 0 → Node 4: 클래스 전체 활동 히트맵
3. Node 0 → Node 2: 클래스 평균 숙련도
4. Node 0 → Node 7: 클래스 공통 오답 패턴
5. Node 0 → Node 5: 클래스 리포트 생성
6. Teacher ← Node 0: 통합 대시보드 데이터
```

**관련 노드**: Node 0, Node 2, Node 4, Node 7

---

### Scenario 5: 시험 준비

**상세 문서**: [05_EXAM_PREPARATION.md](./05_EXAM_PREPARATION.md)

학교 시험 2주 전, 학생의 약점 분석 기반 복습 계획 자동 생성 및 AI 예상 문제, 모의고사 제공으로 효율적인 시험 대비를 지원합니다.

```
1. Student → Node 0: 시험 준비 모드 활성화
2. Node 0 → Node 1: 시험 범위 개념 트리 조회
3. Node 0 → Node 2: BKT 숙련도 조회
4. Node 0 → Node 7: 오답노트 조회 및 복습 스케줄 조정
5. Node 0: 2주 복습 계획 생성
6. Node 0 → Node 2: AI 예상 문제 생성
7. Node 0 → Node 5: 모의고사 PDF 생성
8. Student ← Node 0: 시험 준비 플랜 반환
```

**관련 노드**: Node 0, Node 1, Node 2, Node 5, Node 7

---

## 다음 문서

- [Use Case 01: 주간 학습 진단](./01_WEEKLY_DIAGNOSTIC.md)
- [Use Case 02: 오답 복습 플로우](./02_ERROR_REVIEW_FLOW.md)
- [Use Case 03: 개인화 학습 경로](./03_PERSONALIZED_LEARNING_PATH.md)
- [Use Case 04: 클래스 관리](./04_CLASS_MANAGEMENT.md)
- [Use Case 05: 시험 준비](./05_EXAM_PREPARATION.md)

---

**Last Updated**: 2026-01-10
**Contributors**: Claude Sonnet 4.5
