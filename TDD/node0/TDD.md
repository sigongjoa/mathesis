# Node 0: Student Hub - Technical Design Document (TDD)

**Version**: 1.0.0
**Last Updated**: 2026-01-09
**Status**: Design Phase
**Author**: Mathesis Platform Team

---

## 📋 Table of Contents

1. [시스템 개요](#1-시스템-개요)
2. [기술 스택](#2-기술-스택)
3. [시스템 아키텍처](#3-시스템-아키텍처)
4. [컴포넌트 설계](#4-컴포넌트-설계)
5. [데이터베이스 설계](#5-데이터베이스-설계)
6. [MCP 통신 설계](#6-mcp-통신-설계)
7. [워크플로우 엔진](#7-워크플로우-엔진)
8. [성능 최적화](#8-성능-최적화)
9. [보안 설계](#9-보안-설계)
10. [모니터링 및 로깅](#10-모니터링-및-로깅)

---

## 1. 시스템 개요

### 1.1 목적
Node 0 (Student Hub)는 Mathesis Platform의 **마스터 노드**로서 다음 역할을 수행:
- **Single Source of Truth**: 모든 학생 데이터의 단일 진실 공급원
- **Data Aggregator**: Node 1-6의 데이터를 통합하여 360도 학생 프로필 제공
- **Workflow Orchestrator**: 교육 워크플로우 자동화 및 오케스트레이션

### 1.2 핵심 특징
- **이중 역할**: MCP Server (외부 제공) + MCP Client (내부 호출)
- **실시간 처리**: Redis 기반 캐싱 및 실시간 데이터 동기화
- **비동기 처리**: Celery 기반 백그라운드 작업 처리
- **확장성**: 수평 확장 가능한 마이크로서비스 설계

### 1.3 제약사항
| 구분 | 제약사항 | 근거 |
|------|----------|------|
| **응답 시간** | 통합 프로필 조회 < 2초 | PRD KPI |
| **동시 사용자** | 1,000명 동시 접속 | Phase 1 목표 |
| **데이터 정합성** | Node 간 데이터 동기화 < 5초 | UX 요구사항 |
| **가용성** | 99.5% 이상 | 교육 서비스 특성 |

---

## 2. 기술 스택

### 2.1 Backend Framework
```yaml
Framework: FastAPI 0.104+
  - Async/Await 지원
  - Pydantic 기반 데이터 검증
  - OpenAPI 자동 생성
  - 높은 성능 (Starlette + Uvicorn)

Python Version: 3.11+
  - Type Hints 완전 지원
  - Performance 개선 (10-60% faster)
  - Better error messages
```

### 2.2 Database Stack
```yaml
Primary DB: PostgreSQL 14+
  - 학생 마스터 데이터
  - 학습 히스토리 (시계열 데이터)
  - JSONB 필드 (유연한 메타데이터)
  - Partitioning (학습 이벤트 테이블)

Cache: Redis 7+
  - 통합 프로필 캐시 (TTL: 300s)
  - 세션 관리
  - Rate Limiting
  - Pub/Sub (실시간 알림)

Task Queue: Redis (Celery Backend)
  - 비동기 작업 큐
  - 결과 저장소
```

### 2.3 Message Queue & Task Processing
```yaml
Task Queue: Celery 5.3+
  - 비동기 작업 처리
  - 주기적 작업 스케줄링 (Celery Beat)
  - 재시도 메커니즘
  - Task 우선순위 관리

Broker: Redis
  - 메시지 브로커
  - 결과 백엔드
```

### 2.4 MCP Communication
```yaml
MCP Protocol:
  - Server: 4개 Tools 제공
  - Client: Node 1-6 호출
  - Transport: HTTP/JSON-RPC
  - Timeout: 30초 (기본), 300초 (리포트 생성)
```

### 2.5 Monitoring & Logging
```yaml
Logging:
  - structlog (구조화 로깅)
  - JSON 형식
  - 중앙 집중식 (향후 ELK Stack)

Metrics:
  - Prometheus client
  - Custom metrics (학생 활동, API 호출)

Tracing:
  - OpenTelemetry (향후 통합)
```

---

## 3. 시스템 아키텍처

### 3.1 High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    External Clients                          │
│          (LLM Orchestrator, Frontend, CLI Tools)            │
└────────────────────┬────────────────────────────────────────┘
                     │ MCP Protocol
                     ↓
┌─────────────────────────────────────────────────────────────┐
│                 Node 0: Student Hub (Port 8000)             │
├─────────────────────────────────────────────────────────────┤
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │ MCP Server   │  │ REST API     │  │ Admin API    │     │
│  │ (4 Tools)    │  │ (Internal)   │  │ (Management) │     │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘     │
│         │                  │                  │              │
│         └──────────────────┼──────────────────┘              │
│                            ↓                                 │
│  ┌──────────────────────────────────────────────────────┐  │
│  │           Service Layer (Business Logic)             │  │
│  ├──────────────────────────────────────────────────────┤  │
│  │ • ProfileService   • InterventionService             │  │
│  │ • WorkflowService  • AnalyticsService                │  │
│  └────────┬─────────────────────────────────────┬────────┘  │
│           │                                     │            │
│           ↓                                     ↓            │
│  ┌─────────────────┐                  ┌──────────────────┐ │
│  │ MCP Client      │                  │ Cache Layer      │ │
│  │ (Call Node 1-6) │                  │ (Redis)          │ │
│  └─────────────────┘                  └──────────────────┘ │
│           │                                     │            │
│           ↓                                     ↓            │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              Data Access Layer (DAL)                  │  │
│  ├──────────────────────────────────────────────────────┤  │
│  │ • StudentRepository    • LearningHistoryRepository   │  │
│  │ • InterventionRepository • TaskRepository            │  │
│  └────────┬──────────────────────────────────────────────┘  │
│           │                                                  │
│           ↓                                                  │
│  ┌──────────────────┐         ┌──────────────────┐         │
│  │  PostgreSQL      │         │  Celery Worker   │         │
│  │  (Master Data)   │         │  (Background)    │         │
│  └──────────────────┘         └──────────────────┘         │
└─────────────────────────────────────────────────────────────┘
                     │
                     ↓ MCP Protocol
┌─────────────────────────────────────────────────────────────┐
│              Downstream Services (Node 1-6)                 │
├──────────┬──────────┬──────────┬──────────┬──────────┬──────┤
│ Node 1   │ Node 2   │ Node 3   │ Node 4   │ Node 5   │Node 6│
│ Logic    │ Q-DNA    │ Gen      │ Lab      │ Report   │School│
│ Engine   │ 문제은행  │ AI생성   │ 학습추적 │ 리포트   │ 정보 │
└──────────┴──────────┴──────────┴──────────┴──────────┴──────┘
```

### 3.2 Deployment Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      Docker Compose                          │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  student-hub-api (Port 8000)                         │   │
│  │  • FastAPI Application                               │   │
│  │  • Uvicorn Server (4 workers)                        │   │
│  │  • Health Check: /health                             │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                               │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  student-hub-worker                                  │   │
│  │  • Celery Worker (Concurrency: 4)                    │   │
│  │  • Queues: default, high_priority, low_priority      │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                               │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  student-hub-beat                                    │   │
│  │  • Celery Beat Scheduler                             │   │
│  │  • Periodic Tasks (Daily reports, Weekly analytics)  │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                               │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  postgres (Port 5432)                                │   │
│  │  • PostgreSQL 14                                     │   │
│  │  • Volume: ./data/postgres                           │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                               │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  redis (Port 6379)                                   │   │
│  │  • Redis 7                                           │   │
│  │  • Persistence: AOF enabled                          │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                               │
└─────────────────────────────────────────────────────────────┘

Network: mathesis-network (Bridge)
```

### 3.3 Data Flow

#### 3.3.1 통합 프로필 조회 Flow
```
Client (LLM Orchestrator)
  │
  │ 1. MCP: get_unified_profile(student_id)
  ↓
MCP Server (FastAPI Endpoint)
  │
  │ 2. Check Cache
  ↓
Redis Cache
  │
  ├─ Cache Hit → Return Cached Profile (< 100ms)
  │
  └─ Cache Miss
      │
      │ 3. Fetch Master Data
      ↓
    PostgreSQL (students table)
      │
      │ 4. Aggregate Data (Parallel)
      ↓
    MCP Client (Concurrent Calls)
      ├─→ Node 1: get_knowledge_state(student_id)
      ├─→ Node 2: get_mastery_level(student_id)
      ├─→ Node 4: get_recent_activities(student_id)
      └─→ Node 5: get_latest_reports(student_id)
      │
      │ 5. Merge Results
      ↓
    ProfileService.merge()
      │
      │ 6. Cache Result (TTL: 300s)
      ↓
    Redis Cache
      │
      │ 7. Return Unified Profile
      ↓
    Client
```

**Performance Budget**:
- Cache Hit: < 100ms
- Cache Miss: < 2000ms (4개 노드 병렬 호출)
- Node 호출 Timeout: 500ms/node

#### 3.3.2 학습 개입 생성 Flow
```
Client
  │
  │ 1. MCP: create_learning_intervention(student_id, config)
  ↓
MCP Server
  │
  │ 2. Validate Student
  ↓
PostgreSQL
  │
  │ 3. Analyze Performance
  ↓
MCP Client → Node 2: analyze_weak_areas(student_id)
  │
  │ 4. Generate Learning Path
  ↓
MCP Client → Node 3: generate_problem_set(weak_areas)
  │
  │ 5. Create Intervention (DB)
  ↓
PostgreSQL (interventions table)
  │
  │ 6. Schedule Follow-up (Async)
  ↓
Celery Task Queue
  │
  │ 7. Return Intervention ID
  ↓
Client
  │
  │ (Background)
  ↓
Celery Worker
  ├─→ Send Notification
  ├─→ Update Analytics
  └─→ Schedule Progress Check (7 days later)
```

---

## 4. 컴포넌트 설계

### 4.1 Directory Structure

```
node0_student_hub/
├── app/
│   ├── __init__.py
│   ├── main.py                    # FastAPI 앱 진입점
│   ├── config.py                  # 설정 관리 (Pydantic Settings)
│   │
│   ├── api/                       # API Layer
│   │   ├── __init__.py
│   │   ├── deps.py                # 의존성 주입
│   │   ├── mcp/                   # MCP Server Endpoints
│   │   │   ├── __init__.py
│   │   │   ├── server.py          # MCP 서버 설정
│   │   │   └── tools/             # MCP Tools 구현
│   │   │       ├── profile.py     # get_unified_profile
│   │   │       ├── intervention.py # create_learning_intervention
│   │   │       ├── scheduler.py   # schedule_periodic_task
│   │   │       └── analytics.py   # get_class_analytics
│   │   │
│   │   └── rest/                  # REST API (Internal)
│   │       ├── students.py        # 학생 CRUD
│   │       ├── interventions.py   # 개입 관리
│   │       └── health.py          # Health Check
│   │
│   ├── services/                  # Business Logic Layer
│   │   ├── __init__.py
│   │   ├── profile_service.py     # 프로필 통합
│   │   ├── intervention_service.py # 개입 로직
│   │   ├── workflow_service.py    # 워크플로우 오케스트레이션
│   │   ├── analytics_service.py   # 분석 로직
│   │   └── mcp_client.py          # MCP Client (Node 1-6 호출)
│   │
│   ├── models/                    # Domain Models
│   │   ├── __init__.py
│   │   ├── student.py             # Student 엔티티
│   │   ├── intervention.py        # Intervention 엔티티
│   │   ├── learning_history.py    # LearningHistory 엔티티
│   │   └── task.py                # ScheduledTask 엔티티
│   │
│   ├── schemas/                   # Pydantic Schemas (DTO)
│   │   ├── __init__.py
│   │   ├── student.py             # StudentCreate, StudentResponse
│   │   ├── profile.py             # UnifiedProfile
│   │   ├── intervention.py        # InterventionConfig, InterventionResult
│   │   └── analytics.py           # ClassAnalytics
│   │
│   ├── repositories/              # Data Access Layer
│   │   ├── __init__.py
│   │   ├── student_repo.py
│   │   ├── intervention_repo.py
│   │   ├── learning_history_repo.py
│   │   └── task_repo.py
│   │
│   ├── tasks/                     # Celery Tasks
│   │   ├── __init__.py
│   │   ├── celery_app.py          # Celery 앱 설정
│   │   ├── periodic.py            # 주기적 작업 (Beat)
│   │   └── workflows.py           # 워크플로우 작업
│   │
│   ├── db/                        # Database
│   │   ├── __init__.py
│   │   ├── session.py             # DB 세션 관리
│   │   ├── base.py                # SQLAlchemy Base
│   │   └── migrations/            # Alembic migrations
│   │       └── versions/
│   │
│   ├── cache/                     # Cache Layer
│   │   ├── __init__.py
│   │   ├── redis_client.py        # Redis 클라이언트
│   │   └── cache_service.py       # 캐싱 로직
│   │
│   └── utils/                     # Utilities
│       ├── __init__.py
│       ├── logger.py              # 구조화 로깅
│       ├── metrics.py             # Prometheus 메트릭
│       └── validators.py          # 공통 검증 로직
│
├── tests/                         # 테스트
│   ├── unit/
│   ├── integration/
│   └── conftest.py
│
├── docker/
│   ├── Dockerfile
│   └── docker-compose.yml
│
├── requirements.txt
├── pyproject.toml                 # Poetry or setuptools
└── README.md
```

### 4.2 핵심 컴포넌트 상세 설계

#### 4.2.1 MCP Server (api/mcp/)

**설계 원칙**:
- Tool 단위 분리 (각 Tool은 독립적 모듈)
- Pydantic 기반 입력 검증
- 구조화된 에러 응답

**구현 예시**:
```python
# api/mcp/tools/profile.py
from pydantic import BaseModel, Field
from typing import Optional
from app.services.profile_service import ProfileService

class GetUnifiedProfileInput(BaseModel):
    student_id: str = Field(..., description="학생 ID (UUID)")
    include_history: bool = Field(default=True, description="학습 히스토리 포함 여부")
    days: int = Field(default=30, description="히스토리 조회 기간 (일)")

class GetUnifiedProfileOutput(BaseModel):
    student_id: str
    basic_info: dict
    knowledge_state: dict
    mastery_levels: dict
    recent_activities: list
    latest_reports: list
    cached: bool
    generated_at: str

async def get_unified_profile(
    input: GetUnifiedProfileInput,
    profile_service: ProfileService
) -> GetUnifiedProfileOutput:
    """
    통합 프로필 조회
    - Cache-First 전략
    - Parallel Node 호출
    - Graceful Degradation (일부 노드 실패 시에도 가능한 데이터 반환)
    """
    profile = await profile_service.get_unified_profile(
        student_id=input.student_id,
        include_history=input.include_history,
        days=input.days
    )
    return GetUnifiedProfileOutput(**profile)
```

#### 4.2.2 Service Layer (services/)

**ProfileService 설계**:
```python
# services/profile_service.py
import asyncio
from typing import Dict, Optional
from app.cache.cache_service import CacheService
from app.services.mcp_client import MCPClient
from app.repositories.student_repo import StudentRepository

class ProfileService:
    def __init__(
        self,
        student_repo: StudentRepository,
        cache_service: CacheService,
        mcp_client: MCPClient
    ):
        self.student_repo = student_repo
        self.cache = cache_service
        self.mcp = mcp_client

    async def get_unified_profile(
        self,
        student_id: str,
        include_history: bool = True,
        days: int = 30
    ) -> Dict:
        """
        통합 프로필 조회

        Flow:
        1. Cache 확인
        2. Cache Miss → DB + Node 병렬 호출
        3. 결과 병합 및 캐싱
        """
        # 1. Cache 확인
        cache_key = f"profile:{student_id}:{days}"
        cached = await self.cache.get(cache_key)
        if cached:
            cached["cached"] = True
            return cached

        # 2. Master Data 조회
        student = await self.student_repo.get_by_id(student_id)
        if not student:
            raise ValueError(f"Student {student_id} not found")

        # 3. Node 병렬 호출 (Graceful Degradation)
        tasks = {
            "knowledge": self._get_knowledge_state(student_id),
            "mastery": self._get_mastery_levels(student_id),
            "activities": self._get_recent_activities(student_id, days) if include_history else None,
            "reports": self._get_latest_reports(student_id)
        }

        results = {}
        for key, task in tasks.items():
            if task is None:
                results[key] = None
                continue
            try:
                results[key] = await asyncio.wait_for(task, timeout=1.0)
            except asyncio.TimeoutError:
                results[key] = {"error": "timeout"}
            except Exception as e:
                results[key] = {"error": str(e)}

        # 4. 결과 병합
        profile = {
            "student_id": student_id,
            "basic_info": {
                "name": student.name,
                "school_id": student.school_id,
                "grade": student.grade,
                "class_name": student.class_name
            },
            "knowledge_state": results.get("knowledge", {}),
            "mastery_levels": results.get("mastery", {}),
            "recent_activities": results.get("activities", []),
            "latest_reports": results.get("reports", []),
            "cached": False,
            "generated_at": datetime.utcnow().isoformat()
        }

        # 5. Cache 저장 (TTL: 300s)
        await self.cache.set(cache_key, profile, ttl=300)

        return profile

    async def _get_knowledge_state(self, student_id: str) -> Dict:
        """Node 1: Logic Engine 호출"""
        return await self.mcp.call(
            node="logic-engine",
            tool="get_knowledge_state",
            params={"student_id": student_id}
        )

    # ... 다른 Node 호출 메서드
```

**InterventionService 설계**:
```python
# services/intervention_service.py
from app.tasks.workflows import create_intervention_workflow

class InterventionService:
    async def create_intervention(
        self,
        student_id: str,
        config: InterventionConfig
    ) -> str:
        """
        학습 개입 생성

        Flow:
        1. 학생 성과 분석 (Node 2)
        2. 약점 영역 식별
        3. 학습 경로 생성 (Node 3)
        4. DB 저장
        5. Background Task 스케줄링
        """
        # 1. 성과 분석
        weak_areas = await self._analyze_weak_areas(student_id)

        # 2. 학습 경로 생성
        learning_path = await self._generate_learning_path(
            student_id=student_id,
            weak_areas=weak_areas,
            target_level=config.target_level
        )

        # 3. Intervention 저장
        intervention = await self.intervention_repo.create({
            "student_id": student_id,
            "type": config.type,
            "weak_areas": weak_areas,
            "learning_path": learning_path,
            "status": "active",
            "created_at": datetime.utcnow()
        })

        # 4. Background Workflow 시작
        create_intervention_workflow.delay(
            intervention_id=intervention.id,
            student_id=student_id
        )

        return intervention.id
```

#### 4.2.3 MCP Client (services/mcp_client.py)

**설계**:
```python
# services/mcp_client.py
import httpx
from typing import Dict, Any
from app.config import settings

class MCPClient:
    """
    MCP Client for calling Node 1-6

    Features:
    - Connection Pool (재사용)
    - Timeout 관리
    - Retry with Exponential Backoff
    - Circuit Breaker (향후)
    """

    NODE_ENDPOINTS = {
        "logic-engine": "http://localhost:8001/mcp",
        "q-dna": "http://localhost:8002/mcp",
        "gen-node": "http://localhost:8003/mcp",
        "lab-node": "http://localhost:8004/mcp",
        "report-node": "http://localhost:8005/mcp",
        "school-info": "http://localhost:8006/mcp"
    }

    def __init__(self):
        self.client = httpx.AsyncClient(
            timeout=httpx.Timeout(30.0),
            limits=httpx.Limits(max_connections=100)
        )

    async def call(
        self,
        node: str,
        tool: str,
        params: Dict[str, Any],
        timeout: float = 30.0
    ) -> Dict:
        """
        MCP Tool 호출

        Args:
            node: Node 이름 (e.g., "q-dna")
            tool: Tool 이름 (e.g., "get_mastery_level")
            params: Tool 파라미터
            timeout: 타임아웃 (초)

        Returns:
            Tool 실행 결과

        Raises:
            MCPError: MCP 호출 실패
        """
        endpoint = self.NODE_ENDPOINTS.get(node)
        if not endpoint:
            raise ValueError(f"Unknown node: {node}")

        payload = {
            "jsonrpc": "2.0",
            "id": 1,
            "method": "tools/call",
            "params": {
                "name": tool,
                "arguments": params
            }
        }

        try:
            response = await self.client.post(
                endpoint,
                json=payload,
                timeout=timeout
            )
            response.raise_for_status()

            result = response.json()
            if "error" in result:
                raise MCPError(f"{node}.{tool} failed: {result['error']}")

            return result["result"]

        except httpx.TimeoutException:
            raise MCPError(f"{node}.{tool} timeout after {timeout}s")
        except httpx.HTTPError as e:
            raise MCPError(f"{node}.{tool} HTTP error: {e}")
```

#### 4.2.4 Cache Layer (cache/)

**설계**:
```python
# cache/cache_service.py
import json
from typing import Optional, Any
from redis.asyncio import Redis

class CacheService:
    def __init__(self, redis: Redis):
        self.redis = redis

    async def get(self, key: str) -> Optional[Any]:
        """캐시에서 데이터 조회"""
        data = await self.redis.get(key)
        if data:
            return json.loads(data)
        return None

    async def set(self, key: str, value: Any, ttl: int = 300):
        """캐시에 데이터 저장"""
        await self.redis.setex(
            key,
            ttl,
            json.dumps(value, ensure_ascii=False)
        )

    async def delete(self, key: str):
        """캐시 삭제"""
        await self.redis.delete(key)

    async def invalidate_pattern(self, pattern: str):
        """패턴 매칭으로 캐시 무효화"""
        keys = await self.redis.keys(pattern)
        if keys:
            await self.redis.delete(*keys)
```

#### 4.2.5 Celery Tasks (tasks/)

**설계**:
```python
# tasks/periodic.py
from celery import Celery
from celery.schedules import crontab

celery_app = Celery("student_hub")

# 주기적 작업 스케줄
celery_app.conf.beat_schedule = {
    # 매일 오전 9시: 일일 학습 리포트 생성
    "daily-learning-reports": {
        "task": "app.tasks.periodic.generate_daily_reports",
        "schedule": crontab(hour=9, minute=0),
    },
    # 매주 월요일 오전 10시: 주간 분석
    "weekly-analytics": {
        "task": "app.tasks.periodic.generate_weekly_analytics",
        "schedule": crontab(day_of_week=1, hour=10, minute=0),
    },
    # 매시간: 개입 진행 상태 확인
    "check-interventions": {
        "task": "app.tasks.periodic.check_intervention_progress",
        "schedule": crontab(minute=0),  # 매시 정각
    }
}

@celery_app.task
def generate_daily_reports():
    """일일 학습 리포트 생성 (모든 활성 학생)"""
    # 구현 생략
    pass

@celery_app.task
def generate_weekly_analytics():
    """주간 학급/학년별 분석"""
    # 구현 생략
    pass
```

---

## 5. 데이터베이스 설계

### 5.1 ERD (Entity Relationship Diagram)

```
┌─────────────────────────┐
│       students          │ (마스터 데이터)
├─────────────────────────┤
│ PK  id (UUID)           │
│     name                │
│     school_id           │
│     grade               │
│     class_name          │
│     student_number      │
│     email (nullable)    │
│     parent_contact      │
│     metadata (JSONB)    │
│     created_at          │
│     updated_at          │
└────────┬────────────────┘
         │ 1
         │
         │ N
         ↓
┌─────────────────────────┐
│  learning_history       │ (학습 이벤트 - 시계열)
├─────────────────────────┤
│ PK  id (BIGSERIAL)      │
│ FK  student_id (UUID)   │
│     event_type          │ (study, test, intervention)
│     source_node         │ (1-6)
│     source_id           │ (원본 데이터 ID)
│     content (JSONB)     │ (이벤트 상세)
│     occurred_at         │ (파티션 키)
│     created_at          │
└────────┬────────────────┘
         │
         │
         │
┌────────┴────────────────┐
│                         │
│                         │
↓                         ↓
┌──────────────────┐   ┌──────────────────────┐
│ interventions    │   │  scheduled_tasks     │
├──────────────────┤   ├──────────────────────┤
│ PK id (UUID)     │   │ PK id (UUID)         │
│ FK student_id    │   │ FK student_id        │
│    type          │   │    task_type         │
│    weak_areas    │   │    schedule_type     │
│    learning_path │   │    cron_expression   │
│    status        │   │    config (JSONB)    │
│    progress      │   │    next_run_at       │
│    created_at    │   │    last_run_at       │
│    completed_at  │   │    status            │
└──────────────────┘   │    celery_task_id    │
                       │    created_at        │
                       └──────────────────────┘
```

### 5.2 테이블 상세 스키마

#### 5.2.1 students (학생 마스터 데이터)
```sql
CREATE TABLE students (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(100) NOT NULL,
    school_id VARCHAR(50) NOT NULL,
    grade INTEGER NOT NULL CHECK (grade >= 1 AND grade <= 12),
    class_name VARCHAR(50),
    student_number VARCHAR(20),
    email VARCHAR(255),
    parent_contact VARCHAR(20),
    metadata JSONB DEFAULT '{}',  -- 확장 가능한 메타데이터
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW(),

    -- 인덱스
    CONSTRAINT students_school_grade_class_number_unique
        UNIQUE (school_id, grade, class_name, student_number)
);

CREATE INDEX idx_students_school_id ON students(school_id);
CREATE INDEX idx_students_grade_class ON students(grade, class_name);
CREATE INDEX idx_students_metadata ON students USING GIN (metadata);
```

**metadata 예시**:
```json
{
  "learning_style": "visual",
  "special_needs": false,
  "interests": ["math", "science"],
  "parent_preferences": {
    "notification_method": "email",
    "report_frequency": "weekly"
  }
}
```

#### 5.2.2 learning_history (학습 이벤트 - 시계열 데이터)
```sql
CREATE TABLE learning_history (
    id BIGSERIAL,
    student_id UUID NOT NULL REFERENCES students(id) ON DELETE CASCADE,
    event_type VARCHAR(50) NOT NULL,  -- 'study', 'test', 'intervention'
    source_node INTEGER NOT NULL CHECK (source_node >= 1 AND source_node <= 6),
    source_id VARCHAR(255),  -- 원본 노드의 데이터 ID
    content JSONB NOT NULL,
    occurred_at TIMESTAMP NOT NULL,  -- 파티션 키
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),

    PRIMARY KEY (id, occurred_at)
) PARTITION BY RANGE (occurred_at);

-- 월별 파티션 (예시: 2026년 1월)
CREATE TABLE learning_history_2026_01 PARTITION OF learning_history
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');

-- 인덱스
CREATE INDEX idx_lh_student_occurred ON learning_history(student_id, occurred_at DESC);
CREATE INDEX idx_lh_event_type ON learning_history(event_type);
CREATE INDEX idx_lh_content ON learning_history USING GIN (content);
```

**content 예시**:
```json
{
  "node": 2,
  "action": "solved_problem",
  "problem_id": "prob_12345",
  "result": "correct",
  "time_spent": 180,
  "difficulty": 0.65
}
```

#### 5.2.3 interventions (학습 개입)
```sql
CREATE TABLE interventions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    student_id UUID NOT NULL REFERENCES students(id) ON DELETE CASCADE,
    type VARCHAR(50) NOT NULL,  -- 'auto', 'teacher_requested'
    weak_areas JSONB NOT NULL,  -- 약점 영역 분석 결과
    learning_path JSONB NOT NULL,  -- 생성된 학습 경로
    status VARCHAR(20) NOT NULL DEFAULT 'active',  -- 'active', 'paused', 'completed', 'cancelled'
    progress JSONB DEFAULT '{"completed": 0, "total": 0}',
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    completed_at TIMESTAMP,

    CHECK (status IN ('active', 'paused', 'completed', 'cancelled'))
);

CREATE INDEX idx_interventions_student ON interventions(student_id);
CREATE INDEX idx_interventions_status ON interventions(status);
CREATE INDEX idx_interventions_created ON interventions(created_at DESC);
```

#### 5.2.4 scheduled_tasks (주기적 작업)
```sql
CREATE TABLE scheduled_tasks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    student_id UUID REFERENCES students(id) ON DELETE CASCADE,  -- NULL for global tasks
    task_type VARCHAR(50) NOT NULL,  -- 'daily_report', 'weekly_analytics', etc.
    schedule_type VARCHAR(20) NOT NULL,  -- 'cron', 'interval', 'one_time'
    cron_expression VARCHAR(100),  -- cron 표현식 (schedule_type='cron')
    config JSONB NOT NULL DEFAULT '{}',
    next_run_at TIMESTAMP,
    last_run_at TIMESTAMP,
    status VARCHAR(20) NOT NULL DEFAULT 'active',
    celery_task_id VARCHAR(255),  -- Celery Task ID
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),

    CHECK (schedule_type IN ('cron', 'interval', 'one_time')),
    CHECK (status IN ('active', 'paused', 'completed'))
);

CREATE INDEX idx_tasks_next_run ON scheduled_tasks(next_run_at) WHERE status = 'active';
CREATE INDEX idx_tasks_student ON scheduled_tasks(student_id);
```

### 5.3 Database Migration Strategy

**Tool**: Alembic

**예시 Migration**:
```python
# migrations/versions/001_initial_schema.py
def upgrade():
    op.create_table(
        'students',
        sa.Column('id', postgresql.UUID(), nullable=False),
        sa.Column('name', sa.String(100), nullable=False),
        # ... 생략
        sa.PrimaryKeyConstraint('id')
    )

def downgrade():
    op.drop_table('students')
```

---

## 6. MCP 통신 설계

### 6.1 MCP Server Tools 상세 명세

#### Tool 1: get_unified_profile

**Request**:
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "get_unified_profile",
    "arguments": {
      "student_id": "550e8400-e29b-41d4-a716-446655440000",
      "include_history": true,
      "days": 30
    }
  }
}
```

**Response**:
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "student_id": "550e8400-e29b-41d4-a716-446655440000",
    "basic_info": {
      "name": "김철수",
      "school_id": "SCH001",
      "grade": 10,
      "class_name": "1반"
    },
    "knowledge_state": {
      "node": 1,
      "concepts": [
        {"name": "이차방정식", "mastery": 0.85},
        {"name": "함수", "mastery": 0.72}
      ]
    },
    "mastery_levels": {
      "node": 2,
      "overall": 0.78,
      "by_subject": {
        "math": 0.82,
        "science": 0.74
      }
    },
    "recent_activities": [
      {
        "date": "2026-01-08",
        "type": "study",
        "duration": 3600,
        "problems_solved": 15
      }
    ],
    "latest_reports": [
      {
        "report_id": "REP123",
        "type": "weekly",
        "generated_at": "2026-01-05T10:00:00Z"
      }
    ],
    "cached": false,
    "generated_at": "2026-01-09T05:30:00Z"
  }
}
```

#### Tool 2: create_learning_intervention

**Request**:
```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "tools/call",
  "params": {
    "name": "create_learning_intervention",
    "arguments": {
      "student_id": "550e8400-e29b-41d4-a716-446655440000",
      "type": "auto",
      "target_level": 0.85,
      "duration_days": 14
    }
  }
}
```

**Response**:
```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "result": {
    "intervention_id": "INT_abc123",
    "student_id": "550e8400-e29b-41d4-a716-446655440000",
    "weak_areas": [
      {
        "concept": "이차함수",
        "current_mastery": 0.65,
        "target_mastery": 0.85
      }
    ],
    "learning_path": [
      {
        "step": 1,
        "activity": "review_concept",
        "concept": "이차함수 기본",
        "estimated_duration": 1800
      },
      {
        "step": 2,
        "activity": "practice",
        "problem_set_id": "PS_456",
        "num_problems": 10
      }
    ],
    "scheduled_tasks": [
      {
        "task_id": "TASK_789",
        "type": "progress_check",
        "scheduled_at": "2026-01-16T10:00:00Z"
      }
    ],
    "status": "active",
    "created_at": "2026-01-09T05:30:00Z"
  }
}
```

### 6.2 MCP Client 호출 패턴

#### 6.2.1 병렬 호출 (Fan-Out Pattern)
```python
async def get_unified_profile(student_id: str):
    # 여러 노드에 동시 요청
    tasks = [
        mcp_client.call("logic-engine", "get_knowledge_state", {"student_id": student_id}),
        mcp_client.call("q-dna", "get_mastery_level", {"student_id": student_id}),
        mcp_client.call("lab-node", "get_recent_activities", {"student_id": student_id, "days": 30}),
        mcp_client.call("report-node", "get_latest_reports", {"student_id": student_id})
    ]

    # 모두 완료 대기 (최대 2초)
    results = await asyncio.gather(*tasks, return_exceptions=True)

    # 실패한 호출 처리 (Graceful Degradation)
    return merge_results(results)
```

#### 6.2.2 순차 호출 (Sequential Pattern)
```python
async def create_intervention(student_id: str):
    # 1. 약점 분석 (Node 2)
    weak_areas = await mcp_client.call(
        "q-dna",
        "analyze_weak_areas",
        {"student_id": student_id}
    )

    # 2. 학습 경로 생성 (Node 3) - 약점 분석 결과 필요
    learning_path = await mcp_client.call(
        "gen-node",
        "generate_learning_path",
        {
            "student_id": student_id,
            "weak_areas": weak_areas
        }
    )

    return learning_path
```

### 6.3 Error Handling & Retry

**Retry 전략**:
```python
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=1, max=10)
)
async def call_with_retry(node: str, tool: str, params: dict):
    return await mcp_client.call(node, tool, params)
```

**에러 응답**:
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "error": {
    "code": -32001,
    "message": "Node communication failed",
    "data": {
      "node": "q-dna",
      "tool": "get_mastery_level",
      "reason": "timeout after 30s"
    }
  }
}
```

---

## 7. 워크플로우 엔진

### 7.1 Celery 워크플로우 설계

#### 7.1.1 개입 생성 워크플로우
```python
# tasks/workflows.py
from celery import chain, group

@celery_app.task
def create_intervention_workflow(intervention_id: str, student_id: str):
    """
    개입 생성 후 실행되는 워크플로우

    1. 알림 전송 (학생, 학부모, 교사)
    2. 분석 데이터 업데이트
    3. 7일 후 진행 상태 확인 스케줄링
    """
    workflow = chain(
        send_notifications.s(intervention_id, student_id),
        update_analytics.s(student_id),
        schedule_progress_check.s(intervention_id, days=7)
    )

    return workflow.apply_async()

@celery_app.task
def send_notifications(intervention_id: str, student_id: str):
    """알림 전송 (학생, 학부모, 교사)"""
    # 구현 생략
    pass

@celery_app.task
def update_analytics(student_id: str):
    """분석 데이터 업데이트"""
    # 구현 생략
    pass

@celery_app.task
def schedule_progress_check(intervention_id: str, days: int):
    """진행 상태 확인 스케줄링"""
    # 구현 생략
    pass
```

#### 7.1.2 주기적 리포트 생성
```python
@celery_app.task
def generate_daily_reports():
    """
    일일 학습 리포트 생성

    1. 활성 학생 조회
    2. 병렬로 리포트 생성 (그룹 태스크)
    3. 결과 집계 및 저장
    """
    active_students = get_active_students()

    # 병렬 처리
    job = group([
        generate_student_report.s(student_id)
        for student_id in active_students
    ])

    result = job.apply_async()
    return result.get()

@celery_app.task
def generate_student_report(student_id: str):
    """개별 학생 리포트 생성"""
    # Node 5 (Report Node) 호출
    report = await mcp_client.call(
        "report-node",
        "generate_daily_report",
        {"student_id": student_id}
    )
    return report
```

### 7.2 Task Priority

**Queue 구조**:
```python
# config.py
CELERY_TASK_ROUTES = {
    'app.tasks.workflows.create_intervention_workflow': {
        'queue': 'high_priority'
    },
    'app.tasks.periodic.generate_daily_reports': {
        'queue': 'low_priority'
    },
    'app.tasks.workflows.send_notifications': {
        'queue': 'default'
    }
}
```

**Worker 구성**:
```bash
# High-priority worker
celery -A app.tasks.celery_app worker -Q high_priority -c 4

# Default worker
celery -A app.tasks.celery_app worker -Q default -c 8

# Low-priority worker (batch jobs)
celery -A app.tasks.celery_app worker -Q low_priority -c 2
```

---

## 8. 성능 최적화

### 8.1 Caching Strategy

#### 8.1.1 Cache Layers

| Layer | TTL | 용도 | Key 패턴 |
|-------|-----|------|----------|
| **L1: Profile Cache** | 300s (5분) | 통합 프로필 | `profile:{student_id}:{days}` |
| **L2: Node Response Cache** | 60s (1분) | 개별 노드 응답 | `node:{node_id}:{tool}:{hash}` |
| **L3: Analytics Cache** | 3600s (1시간) | 학급/학년 분석 | `analytics:{class_id}:{date}` |

#### 8.1.2 Cache Invalidation

**전략**:
- **Time-based**: TTL 기반 자동 만료
- **Event-based**: 학생 데이터 변경 시 패턴 매칭 무효화

**구현**:
```python
async def invalidate_student_cache(student_id: str):
    """학생 관련 캐시 무효화"""
    patterns = [
        f"profile:{student_id}:*",
        f"node:*:*:*{student_id}*",
        f"analytics:*"  # 학급 분석도 무효화
    ]

    for pattern in patterns:
        await cache_service.invalidate_pattern(pattern)
```

### 8.2 Database Optimization

#### 8.2.1 Index Strategy
```sql
-- 자주 조회되는 컬럼
CREATE INDEX idx_students_school_id ON students(school_id);
CREATE INDEX idx_students_grade_class ON students(grade, class_name);

-- 시계열 쿼리
CREATE INDEX idx_lh_student_occurred ON learning_history(student_id, occurred_at DESC);

-- JSONB 필드
CREATE INDEX idx_students_metadata ON students USING GIN (metadata);
CREATE INDEX idx_lh_content ON learning_history USING GIN (content);
```

#### 8.2.2 Partitioning
```sql
-- learning_history: 월별 파티션
CREATE TABLE learning_history_2026_01 PARTITION OF learning_history
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');

-- 자동 파티션 생성 (pg_partman 사용)
```

#### 8.2.3 Connection Pooling
```python
# db/session.py
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession

engine = create_async_engine(
    DATABASE_URL,
    echo=False,
    pool_size=20,           # 기본 풀 크기
    max_overflow=10,        # 최대 추가 연결
    pool_pre_ping=True,     # 연결 유효성 검사
    pool_recycle=3600       # 1시간마다 연결 재생성
)
```

### 8.3 MCP Call Optimization

#### 8.3.1 Parallel Calls with Timeout
```python
async def parallel_mcp_calls(calls: List[Dict]):
    """병렬 MCP 호출 with 개별 Timeout"""
    tasks = []
    for call in calls:
        task = asyncio.create_task(
            mcp_client.call(
                call["node"],
                call["tool"],
                call["params"],
                timeout=call.get("timeout", 30.0)
            )
        )
        tasks.append(task)

    # Graceful Degradation
    results = []
    for task in asyncio.as_completed(tasks):
        try:
            result = await task
            results.append(result)
        except Exception as e:
            results.append({"error": str(e)})

    return results
```

#### 8.3.2 Request Batching (향후)
```python
async def batch_get_profiles(student_ids: List[str]):
    """여러 프로필 배치 조회 (1개 요청으로 최적화)"""
    # 구현 예정
    pass
```

---

## 9. 보안 설계

### 9.1 인증 & 인가

**Phase 1 (현재)**: API Key 기반
```python
# api/deps.py
from fastapi import Security, HTTPException
from fastapi.security import APIKeyHeader

api_key_header = APIKeyHeader(name="X-API-Key")

async def verify_api_key(api_key: str = Security(api_key_header)):
    if api_key not in settings.ALLOWED_API_KEYS:
        raise HTTPException(status_code=403, detail="Invalid API key")
    return api_key
```

**Phase 2 (계획)**: OAuth 2.0 + JWT
- Keycloak 통합
- Role-based Access Control (RBAC)

### 9.2 데이터 보안

#### 9.2.1 개인정보 암호화
```python
# utils/crypto.py
from cryptography.fernet import Fernet

class EncryptionService:
    def __init__(self, key: bytes):
        self.cipher = Fernet(key)

    def encrypt(self, data: str) -> str:
        """민감 정보 암호화 (이름, 연락처 등)"""
        return self.cipher.encrypt(data.encode()).decode()

    def decrypt(self, encrypted: str) -> str:
        return self.cipher.decrypt(encrypted.encode()).decode()
```

**적용 필드**:
- `students.name`: 암호화 저장
- `students.parent_contact`: 암호화 저장
- `students.email`: 암호화 저장

#### 9.2.2 접근 제어
```python
# services/profile_service.py
async def get_unified_profile(self, student_id: str, requester_id: str):
    # 권한 확인
    if not await self.can_access(requester_id, student_id):
        raise PermissionError("Access denied")

    # ... 프로필 조회
```

### 9.3 Rate Limiting

```python
# middleware/rate_limit.py
from slowapi import Limiter
from slowapi.util import get_remote_address

limiter = Limiter(key_func=get_remote_address)

@app.get("/mcp/tools/get_unified_profile")
@limiter.limit("100/minute")
async def get_profile(...):
    pass
```

---

## 10. 모니터링 및 로깅

### 10.1 Structured Logging

```python
# utils/logger.py
import structlog

logger = structlog.get_logger()

# 사용 예시
logger.info(
    "profile_fetched",
    student_id=student_id,
    cached=cached,
    duration_ms=duration,
    nodes_called=["q-dna", "logic-engine"]
)
```

**Log 형식** (JSON):
```json
{
  "event": "profile_fetched",
  "student_id": "550e8400-e29b-41d4-a716-446655440000",
  "cached": false,
  "duration_ms": 1823,
  "nodes_called": ["q-dna", "logic-engine"],
  "timestamp": "2026-01-09T05:30:15Z",
  "level": "info"
}
```

### 10.2 Metrics (Prometheus)

```python
# utils/metrics.py
from prometheus_client import Counter, Histogram, Gauge

# Counters
mcp_calls_total = Counter(
    "mcp_calls_total",
    "Total MCP calls",
    ["node", "tool", "status"]
)

# Histograms
profile_fetch_duration = Histogram(
    "profile_fetch_duration_seconds",
    "Profile fetch duration",
    buckets=[0.1, 0.5, 1.0, 2.0, 5.0]
)

# Gauges
active_interventions = Gauge(
    "active_interventions",
    "Number of active interventions"
)
```

**사용**:
```python
with profile_fetch_duration.time():
    profile = await get_unified_profile(student_id)

mcp_calls_total.labels(node="q-dna", tool="get_mastery", status="success").inc()
```

### 10.3 Health Check

```python
# api/rest/health.py
from fastapi import APIRouter

router = APIRouter()

@router.get("/health")
async def health_check():
    """
    헬스 체크
    - Database 연결
    - Redis 연결
    - MCP Client 상태
    """
    checks = {
        "database": await check_db(),
        "redis": await check_redis(),
        "mcp_clients": await check_mcp_clients()
    }

    healthy = all(checks.values())

    return {
        "status": "healthy" if healthy else "unhealthy",
        "checks": checks,
        "timestamp": datetime.utcnow().isoformat()
    }
```

### 10.4 Distributed Tracing (향후)

**OpenTelemetry 통합**:
```python
# 계획
from opentelemetry import trace

tracer = trace.get_tracer(__name__)

@tracer.start_as_current_span("get_unified_profile")
async def get_unified_profile(student_id: str):
    span = trace.get_current_span()
    span.set_attribute("student_id", student_id)
    # ...
```

---

## 부록

### A. 기술 스택 버전 명세

| Component | Version | 비고 |
|-----------|---------|------|
| Python | 3.11+ | Type Hints, Performance |
| FastAPI | 0.104+ | Async, Pydantic v2 |
| PostgreSQL | 14+ | Partitioning, JSONB |
| Redis | 7+ | Async client |
| Celery | 5.3+ | Task queue |
| SQLAlchemy | 2.0+ | Async ORM |
| Pydantic | 2.0+ | 성능 개선 |
| httpx | 0.24+ | Async HTTP client |
| structlog | 23.0+ | 구조화 로깅 |
| prometheus-client | 0.18+ | Metrics |

### B. 환경 변수

```bash
# Database
DATABASE_URL=postgresql+asyncpg://user:pass@localhost:5432/mathesis_hub

# Redis
REDIS_URL=redis://localhost:6379/0

# Celery
CELERY_BROKER_URL=redis://localhost:6379/1
CELERY_RESULT_BACKEND=redis://localhost:6379/2

# Security
API_KEY=your-secret-api-key
ENCRYPTION_KEY=your-encryption-key

# MCP Nodes
NODE1_URL=http://localhost:8001/mcp
NODE2_URL=http://localhost:8002/mcp
# ... Node 3-6

# Logging
LOG_LEVEL=INFO
LOG_FORMAT=json
```

### C. Performance Benchmarks (목표)

| Metric | Target | 측정 방법 |
|--------|--------|-----------|
| Profile fetch (cached) | < 100ms | p95 |
| Profile fetch (uncached) | < 2000ms | p95 |
| Intervention creation | < 3000ms | p95 |
| MCP call (single) | < 500ms | p95 |
| DB query (indexed) | < 50ms | p95 |
| Cache hit rate | > 80% | Profile fetches |

---

**Document Version**: 1.0.0
**Last Updated**: 2026-01-09
**Review Date**: 2026-02-09
**Approved By**: [Pending]
