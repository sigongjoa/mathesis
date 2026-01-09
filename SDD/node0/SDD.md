# Node 0: Student Hub - Software Design Document (SDD)

**Version**: 1.0.0
**Last Updated**: 2026-01-09
**Status**: Design Phase
**Author**: Mathesis Platform Team

---

## 📋 Table of Contents

1. [소프트웨어 개요](#1-소프트웨어-개요)
2. [클래스 설계](#2-클래스-설계)
3. [인터페이스 설계](#3-인터페이스-설계)
4. [시퀀스 다이어그램](#4-시퀀스-다이어그램)
5. [상태 다이어그램](#5-상태-다이어그램)
6. [데이터 흐름 다이어그램](#6-데이터-흐름-다이어그램)
7. [에러 처리](#7-에러-처리)
8. [트랜잭션 관리](#8-트랜잭션-관리)
9. [테스트 전략](#9-테스트-전략)
10. [배포 전략](#10-배포-전략)

---

## 1. 소프트웨어 개요

### 1.1 설계 원칙

#### 1.1.1 SOLID Principles
- **Single Responsibility**: 각 클래스는 단일 책임
- **Open/Closed**: 확장에 열려있고 수정에 닫혀있음
- **Liskov Substitution**: 인터페이스 기반 설계
- **Interface Segregation**: 작고 명확한 인터페이스
- **Dependency Inversion**: 의존성 주입 (FastAPI Depends)

#### 1.1.2 Clean Architecture
```
┌─────────────────────────────────────────────────┐
│           Presentation Layer (API)              │
│     (FastAPI Endpoints, MCP Server/Client)      │
├─────────────────────────────────────────────────┤
│          Application Layer (Services)           │
│    (Business Logic, Orchestration, Workflows)   │
├─────────────────────────────────────────────────┤
│          Domain Layer (Models, Schemas)         │
│        (Entities, Value Objects, DTOs)          │
├─────────────────────────────────────────────────┤
│      Infrastructure Layer (Repositories)        │
│   (Database, Cache, External MCP Calls, Tasks)  │
└─────────────────────────────────────────────────┘
```

#### 1.1.3 Design Patterns
| Pattern | 적용 위치 | 목적 |
|---------|----------|------|
| **Repository** | Data Access Layer | 데이터 접근 추상화 |
| **Factory** | Service Layer | 복잡한 객체 생성 |
| **Strategy** | Intervention Service | 개입 전략 다형성 |
| **Observer** | Event System | 이벤트 기반 통신 |
| **Circuit Breaker** | MCP Client | 장애 격리 (향후) |
| **Singleton** | Cache Service | 단일 인스턴스 |

### 1.2 의존성 관리

#### Dependency Injection (FastAPI Depends)
```python
# api/deps.py
from typing import AsyncGenerator
from sqlalchemy.ext.asyncio import AsyncSession
from app.db.session import SessionLocal
from app.cache.redis_client import get_redis
from app.services.mcp_client import MCPClient

async def get_db() -> AsyncGenerator[AsyncSession, None]:
    """Database 세션 의존성"""
    async with SessionLocal() as session:
        yield session

async def get_cache():
    """Redis 캐시 의존성"""
    return await get_redis()

def get_mcp_client() -> MCPClient:
    """MCP Client 의존성"""
    return MCPClient()
```

---

## 2. 클래스 설계

### 2.1 Domain Models (models/)

#### 2.1.1 Student (학생 엔티티)

```python
# models/student.py
from sqlalchemy import Column, String, Integer, TIMESTAMP, JSON
from sqlalchemy.dialects.postgresql import UUID
from sqlalchemy.orm import relationship
from app.db.base import Base
import uuid
from datetime import datetime

class Student(Base):
    """
    학생 마스터 데이터 엔티티

    책임:
    - 학생 기본 정보 관리
    - 학교 소속 정보 관리
    - 메타데이터 관리 (학습 스타일, 관심사 등)
    """
    __tablename__ = "students"

    # Primary Key
    id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)

    # Basic Info
    name = Column(String(100), nullable=False, comment="학생 이름 (암호화)")
    school_id = Column(String(50), nullable=False, index=True)
    grade = Column(Integer, nullable=False)
    class_name = Column(String(50), nullable=True)
    student_number = Column(String(20), nullable=True)

    # Contact
    email = Column(String(255), nullable=True, comment="이메일 (암호화)")
    parent_contact = Column(String(20), nullable=True, comment="학부모 연락처 (암호화)")

    # Metadata (JSONB)
    metadata = Column(JSON, default={}, nullable=False)

    # Timestamps
    created_at = Column(TIMESTAMP, default=datetime.utcnow, nullable=False)
    updated_at = Column(TIMESTAMP, default=datetime.utcnow, onupdate=datetime.utcnow, nullable=False)

    # Relationships
    learning_history = relationship("LearningHistory", back_populates="student", cascade="all, delete-orphan")
    interventions = relationship("Intervention", back_populates="student", cascade="all, delete-orphan")

    def __repr__(self):
        return f"<Student(id={self.id}, name={self.name}, grade={self.grade})>"

    @property
    def full_identifier(self) -> str:
        """학교-학년-반-번호 형태의 식별자"""
        return f"{self.school_id}-{self.grade}-{self.class_name}-{self.student_number}"

    def update_metadata(self, key: str, value: any):
        """메타데이터 업데이트"""
        if self.metadata is None:
            self.metadata = {}
        self.metadata[key] = value
```

#### 2.1.2 LearningHistory (학습 이력)

```python
# models/learning_history.py
from sqlalchemy import Column, String, Integer, TIMESTAMP, JSON, BigInteger, ForeignKey
from sqlalchemy.dialects.postgresql import UUID
from sqlalchemy.orm import relationship
from app.db.base import Base
from datetime import datetime

class LearningHistory(Base):
    """
    학습 이벤트 엔티티 (시계열 데이터)

    책임:
    - 학습 활동 기록 (문제 풀이, 시험, 개입 등)
    - Node 1-6에서 발생한 이벤트 통합 저장
    - 파티셔닝을 통한 효율적 조회
    """
    __tablename__ = "learning_history"
    __table_args__ = {
        'postgresql_partition_by': 'RANGE (occurred_at)'
    }

    # Composite Primary Key (id, occurred_at for partitioning)
    id = Column(BigInteger, primary_key=True, autoincrement=True)
    occurred_at = Column(TIMESTAMP, primary_key=True, nullable=False, index=True)

    # Foreign Key
    student_id = Column(UUID(as_uuid=True), ForeignKey("students.id", ondelete="CASCADE"), nullable=False, index=True)

    # Event Info
    event_type = Column(String(50), nullable=False, index=True, comment="study, test, intervention")
    source_node = Column(Integer, nullable=False, comment="이벤트 발생 노드 (1-6)")
    source_id = Column(String(255), nullable=True, comment="원본 노드의 데이터 ID")

    # Event Content (JSONB)
    content = Column(JSON, nullable=False, comment="이벤트 상세 데이터")

    # Timestamp
    created_at = Column(TIMESTAMP, default=datetime.utcnow, nullable=False)

    # Relationships
    student = relationship("Student", back_populates="learning_history")

    def __repr__(self):
        return f"<LearningHistory(id={self.id}, student_id={self.student_id}, event_type={self.event_type})>"

    @classmethod
    def from_node_event(cls, student_id: UUID, node: int, event_type: str, content: dict):
        """Node 이벤트로부터 LearningHistory 생성"""
        return cls(
            student_id=student_id,
            event_type=event_type,
            source_node=node,
            source_id=content.get("id"),
            content=content,
            occurred_at=content.get("occurred_at", datetime.utcnow())
        )
```

#### 2.1.3 Intervention (학습 개입)

```python
# models/intervention.py
from sqlalchemy import Column, String, TIMESTAMP, JSON, ForeignKey, CheckConstraint
from sqlalchemy.dialects.postgresql import UUID
from sqlalchemy.orm import relationship
from app.db.base import Base
from datetime import datetime
import uuid

class Intervention(Base):
    """
    학습 개입 엔티티

    책임:
    - 학습 개입 계획 관리
    - 진행 상태 추적
    - 학습 경로 저장
    """
    __tablename__ = "interventions"

    # Primary Key
    id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)

    # Foreign Key
    student_id = Column(UUID(as_uuid=True), ForeignKey("students.id", ondelete="CASCADE"), nullable=False, index=True)

    # Intervention Info
    type = Column(String(50), nullable=False, comment="auto, teacher_requested")
    weak_areas = Column(JSON, nullable=False, comment="약점 영역 분석 결과")
    learning_path = Column(JSON, nullable=False, comment="생성된 학습 경로")

    # Status
    status = Column(String(20), nullable=False, default="active", index=True)
    progress = Column(JSON, default={"completed": 0, "total": 0}, nullable=False)

    # Timestamps
    created_at = Column(TIMESTAMP, default=datetime.utcnow, nullable=False, index=True)
    completed_at = Column(TIMESTAMP, nullable=True)

    # Relationships
    student = relationship("Student", back_populates="interventions")

    # Constraints
    __table_args__ = (
        CheckConstraint("status IN ('active', 'paused', 'completed', 'cancelled')", name="check_status"),
    )

    def __repr__(self):
        return f"<Intervention(id={self.id}, student_id={self.student_id}, status={self.status})>"

    def update_progress(self, completed: int):
        """진행률 업데이트"""
        self.progress["completed"] = completed
        if completed >= self.progress["total"]:
            self.status = "completed"
            self.completed_at = datetime.utcnow()

    def pause(self):
        """개입 일시정지"""
        self.status = "paused"

    def resume(self):
        """개입 재개"""
        if self.status == "paused":
            self.status = "active"

    def cancel(self):
        """개입 취소"""
        self.status = "cancelled"
        self.completed_at = datetime.utcnow()
```

#### 2.1.4 ScheduledTask (예약 작업)

```python
# models/task.py
from sqlalchemy import Column, String, TIMESTAMP, JSON, ForeignKey, CheckConstraint
from sqlalchemy.dialects.postgresql import UUID
from app.db.base import Base
from datetime import datetime
import uuid

class ScheduledTask(Base):
    """
    예약 작업 엔티티

    책임:
    - 주기적/일회성 작업 관리
    - Celery Task 연동
    - 실행 이력 추적
    """
    __tablename__ = "scheduled_tasks"

    # Primary Key
    id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)

    # Foreign Key (nullable for global tasks)
    student_id = Column(UUID(as_uuid=True), ForeignKey("students.id", ondelete="CASCADE"), nullable=True, index=True)

    # Task Info
    task_type = Column(String(50), nullable=False, comment="daily_report, weekly_analytics, etc.")
    schedule_type = Column(String(20), nullable=False, comment="cron, interval, one_time")
    cron_expression = Column(String(100), nullable=True)
    config = Column(JSON, default={}, nullable=False)

    # Status
    status = Column(String(20), nullable=False, default="active")
    next_run_at = Column(TIMESTAMP, nullable=True, index=True)
    last_run_at = Column(TIMESTAMP, nullable=True)
    celery_task_id = Column(String(255), nullable=True)

    # Timestamp
    created_at = Column(TIMESTAMP, default=datetime.utcnow, nullable=False)

    # Constraints
    __table_args__ = (
        CheckConstraint("schedule_type IN ('cron', 'interval', 'one_time')", name="check_schedule_type"),
        CheckConstraint("status IN ('active', 'paused', 'completed')", name="check_task_status"),
    )

    def __repr__(self):
        return f"<ScheduledTask(id={self.id}, task_type={self.task_type}, status={self.status})>"

    def mark_executed(self, next_run: datetime = None):
        """작업 실행 완료 표시"""
        self.last_run_at = datetime.utcnow()
        if self.schedule_type == "one_time":
            self.status = "completed"
            self.next_run_at = None
        elif next_run:
            self.next_run_at = next_run
```

### 2.2 Schemas (Pydantic DTOs)

#### 2.2.1 Student Schemas

```python
# schemas/student.py
from pydantic import BaseModel, Field, EmailStr, UUID4
from typing import Optional, Dict, Any
from datetime import datetime

class StudentBase(BaseModel):
    """학생 기본 스키마"""
    name: str = Field(..., min_length=1, max_length=100)
    school_id: str = Field(..., min_length=1, max_length=50)
    grade: int = Field(..., ge=1, le=12)
    class_name: Optional[str] = Field(None, max_length=50)
    student_number: Optional[str] = Field(None, max_length=20)
    email: Optional[EmailStr] = None
    parent_contact: Optional[str] = Field(None, max_length=20)
    metadata: Dict[str, Any] = Field(default_factory=dict)

class StudentCreate(StudentBase):
    """학생 생성 요청"""
    pass

class StudentUpdate(BaseModel):
    """학생 정보 업데이트 요청"""
    name: Optional[str] = Field(None, min_length=1, max_length=100)
    email: Optional[EmailStr] = None
    parent_contact: Optional[str] = None
    metadata: Optional[Dict[str, Any]] = None

class StudentResponse(StudentBase):
    """학생 조회 응답"""
    id: UUID4
    created_at: datetime
    updated_at: datetime

    class Config:
        from_attributes = True  # Pydantic v2 (was orm_mode)

class UnifiedProfile(BaseModel):
    """통합 프로필 (MCP Tool 응답)"""
    student_id: UUID4
    basic_info: Dict[str, Any]
    knowledge_state: Dict[str, Any]
    mastery_levels: Dict[str, Any]
    recent_activities: list
    latest_reports: list
    cached: bool
    generated_at: datetime
```

#### 2.2.2 Intervention Schemas

```python
# schemas/intervention.py
from pydantic import BaseModel, Field, UUID4
from typing import Optional, Dict, List, Any
from datetime import datetime
from enum import Enum

class InterventionType(str, Enum):
    """개입 유형"""
    AUTO = "auto"
    TEACHER_REQUESTED = "teacher_requested"

class InterventionStatus(str, Enum):
    """개입 상태"""
    ACTIVE = "active"
    PAUSED = "paused"
    COMPLETED = "completed"
    CANCELLED = "cancelled"

class InterventionConfig(BaseModel):
    """개입 생성 설정"""
    student_id: UUID4
    type: InterventionType = InterventionType.AUTO
    target_level: float = Field(..., ge=0.0, le=1.0, description="목표 숙달도")
    duration_days: int = Field(default=14, ge=1, le=90, description="개입 기간 (일)")
    focus_areas: Optional[List[str]] = Field(None, description="집중 영역 (선택)")

class WeakArea(BaseModel):
    """약점 영역"""
    concept: str
    current_mastery: float = Field(..., ge=0.0, le=1.0)
    target_mastery: float = Field(..., ge=0.0, le=1.0)
    priority: int = Field(default=1, ge=1, le=3)

class LearningStep(BaseModel):
    """학습 경로 단계"""
    step: int
    activity: str  # review_concept, practice, assessment
    concept: Optional[str] = None
    problem_set_id: Optional[str] = None
    num_problems: Optional[int] = None
    estimated_duration: int = Field(..., description="예상 소요 시간 (초)")

class InterventionResult(BaseModel):
    """개입 생성 결과"""
    intervention_id: UUID4
    student_id: UUID4
    weak_areas: List[WeakArea]
    learning_path: List[LearningStep]
    scheduled_tasks: List[Dict[str, Any]]
    status: InterventionStatus
    created_at: datetime

class InterventionResponse(BaseModel):
    """개입 조회 응답"""
    id: UUID4
    student_id: UUID4
    type: InterventionType
    weak_areas: List[WeakArea]
    learning_path: List[LearningStep]
    status: InterventionStatus
    progress: Dict[str, int]
    created_at: datetime
    completed_at: Optional[datetime] = None

    class Config:
        from_attributes = True
```

### 2.3 Service Layer

#### 2.3.1 ProfileService 클래스 다이어그램

```
┌───────────────────────────────────────────────────┐
│              ProfileService                       │
├───────────────────────────────────────────────────┤
│ - student_repo: StudentRepository                 │
│ - cache_service: CacheService                     │
│ - mcp_client: MCPClient                           │
│ - logger: Logger                                  │
├───────────────────────────────────────────────────┤
│ + get_unified_profile(student_id, ...)           │
│   → Dict (통합 프로필)                            │
│                                                   │
│ - _get_from_cache(cache_key)                     │
│   → Optional[Dict]                                │
│                                                   │
│ - _fetch_master_data(student_id)                 │
│   → Student                                       │
│                                                   │
│ - _aggregate_from_nodes(student_id, days)        │
│   → Dict (병렬 호출 결과)                         │
│                                                   │
│ - _get_knowledge_state(student_id)               │
│   → Dict (Node 1)                                 │
│                                                   │
│ - _get_mastery_levels(student_id)                │
│   → Dict (Node 2)                                 │
│                                                   │
│ - _get_recent_activities(student_id, days)       │
│   → List (Node 4)                                 │
│                                                   │
│ - _get_latest_reports(student_id)                │
│   → List (Node 5)                                 │
│                                                   │
│ - _merge_results(master_data, node_results)      │
│   → Dict                                          │
│                                                   │
│ - _cache_profile(cache_key, profile, ttl)        │
│   → None                                          │
└───────────────────────────────────────────────────┘
        │                    │                  │
        │ uses               │ uses             │ uses
        ↓                    ↓                  ↓
┌──────────────┐  ┌──────────────────┐  ┌──────────────┐
│StudentRepo   │  │  CacheService    │  │  MCPClient   │
│              │  │                  │  │              │
│ + get_by_id  │  │ + get(key)       │  │ + call(...)  │
│ + create     │  │ + set(key, val)  │  │              │
│ + update     │  │ + delete(key)    │  │              │
└──────────────┘  └──────────────────┘  └──────────────┘
```

#### 2.3.2 InterventionService 클래스 다이어그램

```
┌────────────────────────────────────────────────────┐
│          InterventionService                       │
├────────────────────────────────────────────────────┤
│ - intervention_repo: InterventionRepository        │
│ - student_repo: StudentRepository                  │
│ - mcp_client: MCPClient                            │
│ - task_queue: CeleryApp                            │
│ - strategy_factory: InterventionStrategyFactory    │
├────────────────────────────────────────────────────┤
│ + create_intervention(student_id, config)          │
│   → InterventionResult                             │
│                                                    │
│ + get_intervention(intervention_id)                │
│   → InterventionResponse                           │
│                                                    │
│ + update_progress(intervention_id, completed)      │
│   → None                                           │
│                                                    │
│ + pause_intervention(intervention_id)              │
│   → None                                           │
│                                                    │
│ + resume_intervention(intervention_id)             │
│   → None                                           │
│                                                    │
│ - _validate_student(student_id)                    │
│   → Student                                        │
│                                                    │
│ - _analyze_weak_areas(student_id)                  │
│   → List[WeakArea] (Node 2 호출)                   │
│                                                    │
│ - _generate_learning_path(student_id, weak_areas)  │
│   → List[LearningStep] (Node 3 호출)               │
│                                                    │
│ - _create_intervention_record(...)                 │
│   → Intervention                                   │
│                                                    │
│ - _schedule_background_workflow(intervention_id)   │
│   → None                                           │
└────────────────────────────────────────────────────┘
        │                         │
        │ uses                    │ uses
        ↓                         ↓
┌──────────────────────┐  ┌──────────────────────────┐
│InterventionRepository│  │InterventionStrategyFactory│
│                      │  │                          │
│ + create(...)        │  │ + create_strategy(type)  │
│ + get_by_id(id)      │  │   → InterventionStrategy │
│ + update(id, data)   │  └──────────────────────────┘
└──────────────────────┘           │
                                   │ creates
                                   ↓
                          ┌────────────────────┐
                          │InterventionStrategy│ (Interface)
                          ├────────────────────┤
                          │ + analyze(...)     │
                          │ + generate_path()  │
                          └────────────────────┘
                                   △
                                   │ implements
                   ┌───────────────┴───────────────┐
                   │                               │
            ┌──────────────┐              ┌──────────────┐
            │AutoStrategy  │              │TeacherStrategy│
            │              │              │              │
            │ + analyze()  │              │ + analyze()  │
            │ + generate() │              │ + generate() │
            └──────────────┘              └──────────────┘
```

#### 2.3.3 MCPClient 클래스 다이어그램

```
┌──────────────────────────────────────────────────┐
│                 MCPClient                        │
├──────────────────────────────────────────────────┤
│ - client: httpx.AsyncClient                      │
│ - endpoints: Dict[str, str]                      │
│ - logger: Logger                                 │
│ - metrics: MetricsCollector                      │
├──────────────────────────────────────────────────┤
│ + call(node, tool, params, timeout)              │
│   → Dict                                         │
│                                                  │
│ + batch_call(calls: List[CallSpec])              │
│   → List[Dict] (병렬 호출)                        │
│                                                  │
│ + call_with_retry(node, tool, params, retries)  │
│   → Dict                                         │
│                                                  │
│ - _build_request_payload(tool, params)           │
│   → Dict (JSON-RPC 2.0)                          │
│                                                  │
│ - _parse_response(response)                      │
│   → Dict                                         │
│                                                  │
│ - _handle_error(error)                           │
│   → None (raises MCPError)                       │
│                                                  │
│ + close()                                        │
│   → None (cleanup)                               │
└──────────────────────────────────────────────────┘
        │
        │ uses
        ↓
┌──────────────────┐
│ httpx.AsyncClient│
│                  │
│ + post(...)      │
│ + get(...)       │
└──────────────────┘
```

### 2.4 Repository Layer

#### 2.4.1 Base Repository (추상 클래스)

```python
# repositories/base.py
from typing import TypeVar, Generic, List, Optional, Dict, Any
from sqlalchemy import select, update, delete
from sqlalchemy.ext.asyncio import AsyncSession
from app.db.base import Base

ModelType = TypeVar("ModelType", bound=Base)

class BaseRepository(Generic[ModelType]):
    """
    Base Repository (추상 클래스)

    책임:
    - CRUD 기본 operations 제공
    - 타입 안전성 보장
    - 공통 쿼리 패턴 추상화
    """

    def __init__(self, model: type[ModelType], db: AsyncSession):
        self.model = model
        self.db = db

    async def get_by_id(self, id: Any) -> Optional[ModelType]:
        """ID로 조회"""
        result = await self.db.execute(
            select(self.model).where(self.model.id == id)
        )
        return result.scalar_one_or_none()

    async def get_all(self, skip: int = 0, limit: int = 100) -> List[ModelType]:
        """전체 조회 (페이징)"""
        result = await self.db.execute(
            select(self.model).offset(skip).limit(limit)
        )
        return result.scalars().all()

    async def create(self, obj_in: Dict[str, Any]) -> ModelType:
        """생성"""
        db_obj = self.model(**obj_in)
        self.db.add(db_obj)
        await self.db.commit()
        await self.db.refresh(db_obj)
        return db_obj

    async def update(self, id: Any, obj_in: Dict[str, Any]) -> Optional[ModelType]:
        """업데이트"""
        await self.db.execute(
            update(self.model).where(self.model.id == id).values(**obj_in)
        )
        await self.db.commit()
        return await self.get_by_id(id)

    async def delete(self, id: Any) -> bool:
        """삭제"""
        result = await self.db.execute(
            delete(self.model).where(self.model.id == id)
        )
        await self.db.commit()
        return result.rowcount > 0
```

#### 2.4.2 StudentRepository

```python
# repositories/student_repo.py
from typing import List, Optional
from sqlalchemy import select
from app.models.student import Student
from app.repositories.base import BaseRepository

class StudentRepository(BaseRepository[Student]):
    """
    Student Repository

    책임:
    - 학생 데이터 CRUD
    - 학교/학년/반별 조회
    - 복잡한 검색 쿼리
    """

    async def get_by_school(self, school_id: str, grade: Optional[int] = None) -> List[Student]:
        """학교별 학생 조회 (선택적 학년 필터)"""
        query = select(Student).where(Student.school_id == school_id)
        if grade:
            query = query.where(Student.grade == grade)
        result = await self.db.execute(query)
        return result.scalars().all()

    async def get_by_class(self, school_id: str, grade: int, class_name: str) -> List[Student]:
        """학급별 학생 조회"""
        result = await self.db.execute(
            select(Student)
            .where(Student.school_id == school_id)
            .where(Student.grade == grade)
            .where(Student.class_name == class_name)
        )
        return result.scalars().all()

    async def search_by_name(self, name: str) -> List[Student]:
        """이름으로 검색 (부분 일치)"""
        result = await self.db.execute(
            select(Student).where(Student.name.ilike(f"%{name}%"))
        )
        return result.scalars().all()

    async def exists(self, school_id: str, grade: int, class_name: str, student_number: str) -> bool:
        """학생 존재 여부 확인 (중복 체크)"""
        result = await self.db.execute(
            select(Student.id)
            .where(Student.school_id == school_id)
            .where(Student.grade == grade)
            .where(Student.class_name == class_name)
            .where(Student.student_number == student_number)
        )
        return result.scalar_one_or_none() is not None
```

---

## 3. 인터페이스 설계

### 3.1 MCP Server Interface

#### 3.1.1 Tool Interface (Protocol)

```python
# api/mcp/protocols.py
from typing import Protocol, TypeVar, Generic
from pydantic import BaseModel

InputT = TypeVar("InputT", bound=BaseModel)
OutputT = TypeVar("OutputT", bound=BaseModel)

class MCPTool(Protocol, Generic[InputT, OutputT]):
    """
    MCP Tool Interface

    모든 MCP Tool은 이 프로토콜을 구현해야 함
    """

    @property
    def name(self) -> str:
        """Tool 이름"""
        ...

    @property
    def description(self) -> str:
        """Tool 설명"""
        ...

    @property
    def input_schema(self) -> type[InputT]:
        """입력 스키마 (Pydantic Model)"""
        ...

    @property
    def output_schema(self) -> type[OutputT]:
        """출력 스키마 (Pydantic Model)"""
        ...

    async def execute(self, input: InputT) -> OutputT:
        """Tool 실행"""
        ...
```

#### 3.1.2 Tool 구현 예시

```python
# api/mcp/tools/profile.py
from app.api.mcp.protocols import MCPTool
from app.schemas.student import UnifiedProfile
from app.schemas.profile import GetUnifiedProfileInput
from app.services.profile_service import ProfileService

class GetUnifiedProfileTool(MCPTool[GetUnifiedProfileInput, UnifiedProfile]):
    """통합 프로필 조회 Tool"""

    def __init__(self, profile_service: ProfileService):
        self._profile_service = profile_service

    @property
    def name(self) -> str:
        return "get_unified_profile"

    @property
    def description(self) -> str:
        return "학생의 통합 프로필 조회 (마스터 데이터 + Node 1-6 데이터 통합)"

    @property
    def input_schema(self) -> type[GetUnifiedProfileInput]:
        return GetUnifiedProfileInput

    @property
    def output_schema(self) -> type[UnifiedProfile]:
        return UnifiedProfile

    async def execute(self, input: GetUnifiedProfileInput) -> UnifiedProfile:
        profile = await self._profile_service.get_unified_profile(
            student_id=str(input.student_id),
            include_history=input.include_history,
            days=input.days
        )
        return UnifiedProfile(**profile)
```

### 3.2 Service Interface

#### 3.2.1 InterventionStrategy (전략 패턴)

```python
# services/intervention/strategy.py
from abc import ABC, abstractmethod
from typing import List
from app.schemas.intervention import WeakArea, LearningStep

class InterventionStrategy(ABC):
    """
    개입 전략 인터페이스 (Strategy Pattern)

    구현체:
    - AutoStrategy: 자동 개입 (BKT/IRT 기반)
    - TeacherRequestedStrategy: 교사 요청 개입
    """

    @abstractmethod
    async def analyze_weak_areas(self, student_id: str) -> List[WeakArea]:
        """약점 영역 분석"""
        pass

    @abstractmethod
    async def generate_learning_path(
        self,
        student_id: str,
        weak_areas: List[WeakArea],
        target_level: float,
        duration_days: int
    ) -> List[LearningStep]:
        """학습 경로 생성"""
        pass

# 구현 1: 자동 개입
class AutoInterventionStrategy(InterventionStrategy):
    def __init__(self, mcp_client):
        self.mcp = mcp_client

    async def analyze_weak_areas(self, student_id: str) -> List[WeakArea]:
        # Node 2 (Q-DNA) 호출하여 BKT 기반 분석
        result = await self.mcp.call("q-dna", "analyze_weak_areas", {"student_id": student_id})
        return [WeakArea(**area) for area in result["weak_areas"]]

    async def generate_learning_path(self, student_id, weak_areas, target_level, duration_days):
        # Node 3 (Gen Node) 호출하여 학습 경로 생성
        result = await self.mcp.call(
            "gen-node",
            "generate_adaptive_path",
            {
                "student_id": student_id,
                "weak_areas": [a.dict() for a in weak_areas],
                "target_level": target_level,
                "duration_days": duration_days
            }
        )
        return [LearningStep(**step) for step in result["learning_path"]]

# 구현 2: 교사 요청 개입
class TeacherRequestedInterventionStrategy(InterventionStrategy):
    def __init__(self, mcp_client):
        self.mcp = mcp_client

    async def analyze_weak_areas(self, student_id: str) -> List[WeakArea]:
        # 교사가 지정한 영역 (config에서 가져옴)
        # 구현 생략
        pass

    async def generate_learning_path(self, student_id, weak_areas, target_level, duration_days):
        # 교사가 커스터마이징한 경로 생성
        # 구현 생략
        pass
```

### 3.3 Repository Interface

```python
# repositories/interfaces.py
from typing import Protocol, TypeVar, List, Optional, Dict, Any

T = TypeVar("T")

class IRepository(Protocol[T]):
    """Repository 인터페이스"""

    async def get_by_id(self, id: Any) -> Optional[T]:
        """ID로 조회"""
        ...

    async def get_all(self, skip: int = 0, limit: int = 100) -> List[T]:
        """전체 조회"""
        ...

    async def create(self, obj_in: Dict[str, Any]) -> T:
        """생성"""
        ...

    async def update(self, id: Any, obj_in: Dict[str, Any]) -> Optional[T]:
        """업데이트"""
        ...

    async def delete(self, id: Any) -> bool:
        """삭제"""
        ...
```

---

## 4. 시퀀스 다이어그램

### 4.1 통합 프로필 조회 (Cache Hit)

```
Client              MCP Server       ProfileService    CacheService
  │                     │                   │                │
  │ get_unified_profile │                   │                │
  ├────────────────────>│                   │                │
  │                     │ get_unified_profile│               │
  │                     ├──────────────────>│                │
  │                     │                   │ get(cache_key) │
  │                     │                   ├───────────────>│
  │                     │                   │                │
  │                     │                   │ Cache Hit! ✓   │
  │                     │                   │<───────────────┤
  │                     │                   │                │
  │                     │ UnifiedProfile    │                │
  │                     │<──────────────────┤                │
  │ UnifiedProfile      │                   │                │
  │<────────────────────┤                   │                │
  │ (< 100ms)           │                   │                │
```

### 4.2 통합 프로필 조회 (Cache Miss - 병렬 호출)

```
Client      MCP Server   ProfileService   StudentRepo   MCPClient   Node1  Node2  Node4  Node5   Cache
  │             │              │               │            │          │      │      │      │       │
  │ get_profile │              │               │            │          │      │      │      │       │
  ├────────────>│              │               │            │          │      │      │      │       │
  │             │ get_profile  │               │            │          │      │      │      │       │
  │             ├─────────────>│               │            │          │      │      │      │       │
  │             │              │ get(key)      │            │          │      │      │      │       │
  │             │              ├───────────────┼───────────────────────┼──────┼──────┼──────┼──────>│
  │             │              │               │            │          │      │      │      │ Miss  │
  │             │              │<──────────────┼───────────────────────┼──────┼──────┼──────┼───────┤
  │             │              │               │            │          │      │      │      │       │
  │             │              │ get_by_id(student_id)      │          │      │      │      │       │
  │             │              ├──────────────>│            │          │      │      │      │       │
  │             │              │ Student       │            │          │      │      │      │       │
  │             │              │<──────────────┤            │          │      │      │      │       │
  │             │              │               │            │          │      │      │      │       │
  │             │              │ Parallel MCP Calls (asyncio.gather)   │      │      │      │       │
  │             │              ├───────────────┼───────────>│          │      │      │      │       │
  │             │              │               │            │ get_knowledge_state      │      │       │
  │             │              │               │            ├─────────>│      │      │      │       │
  │             │              │               │            │ get_mastery_level│      │      │       │
  │             │              │               │            ├─────────────────>│      │      │       │
  │             │              │               │            │ get_recent_activities    │      │       │
  │             │              │               │            ├──────────────────────────>│      │       │
  │             │              │               │            │ get_latest_reports       │      │       │
  │             │              │               │            ├────────────────────────────────>│       │
  │             │              │               │            │          │      │      │      │       │
  │             │              │               │            │ Result 1 │      │      │      │       │
  │             │              │               │            │<─────────┤      │      │      │       │
  │             │              │               │            │ Result 2 │      │      │      │       │
  │             │              │               │            │<─────────────────┤      │      │       │
  │             │              │               │            │ Result 3 │      │      │      │       │
  │             │              │               │            │<──────────────────────────┤      │       │
  │             │              │               │            │ Result 4 │      │      │      │       │
  │             │              │               │            │<────────────────────────────────┤       │
  │             │              │               │            │          │      │      │      │       │
  │             │              │ Node Results  │            │          │      │      │      │       │
  │             │              │<──────────────┼────────────┤          │      │      │      │       │
  │             │              │               │            │          │      │      │      │       │
  │             │              │ merge(master_data, node_results)     │      │      │      │       │
  │             │              │ ──────────────────────────────────── │      │      │      │       │
  │             │              │               │            │          │      │      │      │       │
  │             │              │ set(key, profile, ttl=300) │          │      │      │      │       │
  │             │              ├───────────────┼───────────────────────┼──────┼──────┼──────┼──────>│
  │             │              │               │            │          │      │      │      │ OK    │
  │             │              │<──────────────┼───────────────────────┼──────┼──────┼──────┼───────┤
  │             │              │               │            │          │      │      │      │       │
  │             │ UnifiedProfile│              │            │          │      │      │      │       │
  │             │<─────────────┤               │            │          │      │      │      │       │
  │ Profile     │              │               │            │          │      │      │      │       │
  │<────────────┤              │               │            │          │      │      │      │       │
  │ (< 2000ms)  │              │               │            │          │      │      │      │       │
```

### 4.3 학습 개입 생성 (순차 + 비동기)

```
Client      MCP Server   InterventionService   Node2   Node3   InterventionRepo   CeleryQueue
  │             │                │                │       │            │               │
  │ create_intervention           │                │       │            │               │
  ├────────────>│                │                │       │            │               │
  │             │ create_intervention              │       │            │               │
  │             ├───────────────>│                │       │            │               │
  │             │                │ validate_student│       │            │               │
  │             │                │ ─────────────── │       │            │               │
  │             │                │                │       │            │               │
  │             │                │ analyze_weak_areas      │            │               │
  │             │                ├───────────────>│       │            │               │
  │             │                │ weak_areas     │       │            │               │
  │             │                │<───────────────┤       │            │               │
  │             │                │                │       │            │               │
  │             │                │ generate_learning_path  │            │               │
  │             │                ├───────────────────────>│            │               │
  │             │                │ learning_path  │       │            │               │
  │             │                │<───────────────────────┤            │               │
  │             │                │                │       │            │               │
  │             │                │ create(intervention_data)           │               │
  │             │                ├────────────────┼───────┼───────────>│               │
  │             │                │ Intervention   │       │            │               │
  │             │                │<───────────────┼───────┼────────────┤               │
  │             │                │                │       │            │               │
  │             │                │ schedule_workflow(intervention_id)  │               │
  │             │                ├────────────────┼───────┼────────────┼──────────────>│
  │             │                │                │       │            │ Task Queued   │
  │             │                │<───────────────┼───────┼────────────┼───────────────┤
  │             │                │                │       │            │               │
  │             │ InterventionResult              │       │            │               │
  │             │<───────────────┤                │       │            │               │
  │ Result      │                │                │       │            │               │
  │<────────────┤                │                │       │            │               │
  │ (< 3000ms)  │                │                │       │            │               │
  │             │                │                │       │            │               │
  │             │                │           (Background - Celery Worker)              │
  │             │                │                │       │            │      Worker   │
  │             │                │                │       │            │       │       │
  │             │                │                │       │            │ pick_task     │
  │             │                │                │       │            │<──────┤       │
  │             │                │                │       │            │       │       │
  │             │                │                │       │     send_notifications     │
  │             │                │                │       │            │       │       │
  │             │                │                │       │     update_analytics       │
  │             │                │                │       │            │       │       │
  │             │                │                │       │  schedule_progress_check   │
  │             │                │                │       │            │ (+7 days)     │
```

### 4.4 주기적 작업 실행 (Celery Beat)

```
Celery Beat      CeleryQueue      Worker       ProfileService    Node5 (Report)   StudentRepo
     │                │               │                │               │               │
     │ (매일 09:00)   │               │                │               │               │
     │ daily_reports  │               │                │               │               │
     ├───────────────>│               │                │               │               │
     │                │ task queued   │                │               │               │
     │                │──────────────>│                │               │               │
     │                │               │ get_active_students            │               │
     │                │               ├────────────────┼───────────────┼──────────────>│
     │                │               │ students[]     │               │               │
     │                │               │<───────────────┼───────────────┼───────────────┤
     │                │               │                │               │               │
     │                │               │ for each student:              │               │
     │                │               │                │               │               │
     │                │               │ generate_daily_report(student) │               │
     │                │               ├───────────────>│               │               │
     │                │               │                │ call(report_node)             │
     │                │               │                ├──────────────>│               │
     │                │               │                │ PDF Report    │               │
     │                │               │                │<──────────────┤               │
     │                │               │ Report         │               │               │
     │                │               │<───────────────┤               │               │
     │                │               │                │               │               │
     │                │ task complete │                │               │               │
     │                │<──────────────┤                │               │               │
```

---

## 5. 상태 다이어그램

### 5.1 Intervention 상태 전이

```
                       ┌─────────────┐
                       │   ACTIVE    │ (초기 상태)
                       └──────┬──────┘
                              │
              ┌───────────────┼───────────────┐
              │               │               │
      pause() │        complete()       cancel()
              │               │               │
              ↓               ↓               ↓
        ┌──────────┐    ┌──────────┐    ┌──────────┐
        │  PAUSED  │    │COMPLETED │    │CANCELLED │
        └────┬─────┘    └──────────┘    └──────────┘
             │               (종료)          (종료)
      resume()
             │
             ↓
        ┌──────────┐
        │  ACTIVE  │
        └──────────┘

상태 전이 조건:
- ACTIVE → PAUSED: 사용자 또는 시스템 요청으로 일시정지
- ACTIVE → COMPLETED: progress.completed >= progress.total
- ACTIVE → CANCELLED: 사용자 또는 교사가 개입 취소
- PAUSED → ACTIVE: 사용자가 재개 요청
```

### 5.2 ScheduledTask 상태 전이

```
                       ┌─────────────┐
                       │   ACTIVE    │ (초기 상태)
                       └──────┬──────┘
                              │
              ┌───────────────┼───────────────┐
              │               │               │
      pause() │      (schedule_type='one_time'
              │        && executed)           │
              │               │               │
              ↓               ↓               │
        ┌──────────┐    ┌──────────┐         │
        │  PAUSED  │    │COMPLETED │         │
        └────┬─────┘    └──────────┘         │
             │               (종료)           │
      resume()                               │
             │                                │
             ↓                                │
        ┌──────────┐                         │
        │  ACTIVE  │<────────────────────────┘
        └──────────┘        (cron/interval 작업은 계속 실행)

상태 전이 조건:
- ACTIVE → PAUSED: 작업 일시정지
- ACTIVE → COMPLETED: one_time 작업 실행 완료
- PAUSED → ACTIVE: 작업 재개
- cron/interval 작업은 ACTIVE 상태 유지
```

---

## 6. 데이터 흐름 다이어그램

### 6.1 Level 0: System Context DFD

```
┌────────────────┐
│  LLM           │──────┐
│  Orchestrator  │      │
└────────────────┘      │
                        │ MCP Requests
┌────────────────┐      │ (JSON-RPC 2.0)
│  Frontend      │──────┤
│  Application   │      │
└────────────────┘      │
                        ↓
                ┌──────────────────┐
                │   Node 0:        │
                │   Student Hub    │────────┐
                └──────────────────┘        │ MCP Calls
                        ↑                   │
                        │ Student Data      ↓
                        │                ┌───────────────┐
                ┌───────┴───────┐        │  Node 1-6:    │
                │  PostgreSQL   │        │  Domain       │
                │  (Master DB)  │        │  Services     │
                └───────────────┘        └───────────────┘
```

### 6.2 Level 1: Student Hub DFD

```
External              ┌──────────────────────────────────┐              External
Clients               │        Node 0: Student Hub       │              Services
  │                   │                                  │                 │
  │ 1. Profile Request│                                  │                 │
  ├──────────────────>│  ┌────────────────────────┐     │                 │
  │                   │  │  MCP Server (Tools)    │     │                 │
  │                   │  └───────────┬────────────┘     │                 │
  │                   │              │                  │                 │
  │                   │              ↓                  │                 │
  │                   │  ┌────────────────────────┐     │                 │
  │                   │  │  Service Layer         │     │                 │
  │                   │  │  - ProfileService      │     │                 │
  │                   │  │  - InterventionService │     │                 │
  │                   │  └───────┬────────────────┘     │                 │
  │                   │          │                      │                 │
  │                   │          ↓                      │                 │
  │                   │  ┌───────────────┐              │                 │
  │                   │  │  Cache        │              │                 │
  │ 2. Cached Profile │  │  (Redis)      │              │                 │
  │<──────────────────├──┤  (Hit)        │              │                 │
  │   (< 100ms)       │  └───────────────┘              │                 │
  │                   │          │                      │                 │
  │                   │          │ (Miss)               │                 │
  │                   │          ↓                      │                 │
  │                   │  ┌───────────────┐              │  3. MCP Calls   │
  │                   │  │  Database     │              │  (Parallel)     │
  │                   │  │  (PostgreSQL) │              ├────────────────>│
  │                   │  └───────────────┘              │                 │
  │                   │          +                      │  4. Node Results│
  │                   │  ┌───────────────┐              │<────────────────┤
  │                   │  │  MCP Client   │              │                 │
  │                   │  │  (Node 1-6)   │──────────────┤                 │
  │                   │  └───────────────┘              │                 │
  │                   │          │                      │                 │
  │                   │          ↓                      │                 │
  │ 5. Unified Profile│  ┌───────────────┐              │                 │
  │<──────────────────├──┤  Merge &      │              │                 │
  │   (< 2s)          │  │  Cache        │              │                 │
  │                   │  └───────────────┘              │                 │
  │                   └──────────────────────────────────┘                 │
```

### 6.3 데이터 저장소 (Data Stores)

| ID | 이름 | 타입 | 용도 | TTL/보관기간 |
|----|------|------|------|--------------|
| **DS1** | students | PostgreSQL | 학생 마스터 데이터 | 영구 |
| **DS2** | learning_history | PostgreSQL (Partitioned) | 학습 이벤트 로그 | 2년 (월별 파티션) |
| **DS3** | interventions | PostgreSQL | 학습 개입 기록 | 영구 |
| **DS4** | scheduled_tasks | PostgreSQL | 예약 작업 | 영구 |
| **DS5** | profile_cache | Redis | 통합 프로필 캐시 | 300s |
| **DS6** | node_response_cache | Redis | 노드 응답 캐시 | 60s |
| **DS7** | celery_tasks | Redis | Celery 작업 큐 | 7일 (결과) |

---

## 7. 에러 처리

### 7.1 에러 계층 구조

```python
# utils/exceptions.py
class StudentHubError(Exception):
    """Base exception for Student Hub"""
    def __init__(self, message: str, code: str, details: dict = None):
        self.message = message
        self.code = code
        self.details = details or {}
        super().__init__(self.message)

class ValidationError(StudentHubError):
    """입력 검증 오류"""
    def __init__(self, message: str, field: str = None):
        super().__init__(
            message=message,
            code="VALIDATION_ERROR",
            details={"field": field} if field else {}
        )

class NotFoundError(StudentHubError):
    """리소스 없음"""
    def __init__(self, resource: str, id: str):
        super().__init__(
            message=f"{resource} not found: {id}",
            code="NOT_FOUND",
            details={"resource": resource, "id": id}
        )

class MCPError(StudentHubError):
    """MCP 통신 오류"""
    def __init__(self, node: str, tool: str, reason: str):
        super().__init__(
            message=f"MCP call failed: {node}.{tool}",
            code="MCP_ERROR",
            details={"node": node, "tool": tool, "reason": reason}
        )

class CacheError(StudentHubError):
    """캐시 오류"""
    pass

class DatabaseError(StudentHubError):
    """데이터베이스 오류"""
    pass
```

### 7.2 에러 처리 전략

#### 7.2.1 MCP Call Error Handling (Graceful Degradation)

```python
# services/profile_service.py
async def _aggregate_from_nodes(self, student_id: str, days: int) -> Dict:
    """
    Node 병렬 호출 with Graceful Degradation

    전략:
    - 각 노드 호출은 독립적
    - 일부 노드 실패 시에도 가능한 데이터 반환
    - 실패한 노드는 에러 정보 포함
    """
    tasks = {
        "knowledge": self._call_with_fallback("logic-engine", "get_knowledge_state", {"student_id": student_id}),
        "mastery": self._call_with_fallback("q-dna", "get_mastery_level", {"student_id": student_id}),
        "activities": self._call_with_fallback("lab-node", "get_recent_activities", {"student_id": student_id, "days": days}),
        "reports": self._call_with_fallback("report-node", "get_latest_reports", {"student_id": student_id})
    }

    results = {}
    for key, task in tasks.items():
        try:
            results[key] = await asyncio.wait_for(task, timeout=1.0)
        except asyncio.TimeoutError:
            logger.warning(f"Node call timeout: {key}", student_id=student_id)
            results[key] = {"error": "timeout", "available": False}
        except MCPError as e:
            logger.error(f"MCP error: {key}", error=str(e), student_id=student_id)
            results[key] = {"error": str(e), "available": False}

    return results

async def _call_with_fallback(self, node: str, tool: str, params: dict):
    """Fallback 포함 MCP 호출"""
    try:
        return await self.mcp.call(node, tool, params)
    except Exception:
        # 실패 시 빈 결과 반환 (Graceful Degradation)
        return {"error": "fallback", "data": None}
```

#### 7.2.2 Database Transaction Error Handling

```python
# services/intervention_service.py
from sqlalchemy.exc import IntegrityError

async def create_intervention(self, student_id: str, config: InterventionConfig):
    """트랜잭션 기반 개입 생성"""
    try:
        async with self.db.begin():  # 트랜잭션 시작
            # 1. 학생 검증
            student = await self.student_repo.get_by_id(student_id)
            if not student:
                raise NotFoundError("Student", student_id)

            # 2. 약점 분석
            weak_areas = await self._analyze_weak_areas(student_id)

            # 3. 학습 경로 생성
            learning_path = await self._generate_learning_path(student_id, weak_areas, ...)

            # 4. Intervention 저장
            intervention = await self.intervention_repo.create({
                "student_id": student_id,
                "type": config.type,
                "weak_areas": [w.dict() for w in weak_areas],
                "learning_path": [l.dict() for l in learning_path],
                "status": "active"
            })

            # 트랜잭션 커밋 (자동)
            return intervention

    except IntegrityError as e:
        logger.error("Database integrity error", error=str(e))
        raise DatabaseError("Failed to create intervention", code="DB_INTEGRITY_ERROR")
    except Exception as e:
        logger.error("Unexpected error", error=str(e))
        raise
```

### 7.3 FastAPI Exception Handlers

```python
# main.py
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse
from app.utils.exceptions import StudentHubError, NotFoundError, ValidationError

app = FastAPI()

@app.exception_handler(NotFoundError)
async def not_found_handler(request: Request, exc: NotFoundError):
    return JSONResponse(
        status_code=404,
        content={
            "error": {
                "code": exc.code,
                "message": exc.message,
                "details": exc.details
            }
        }
    )

@app.exception_handler(ValidationError)
async def validation_error_handler(request: Request, exc: ValidationError):
    return JSONResponse(
        status_code=422,
        content={
            "error": {
                "code": exc.code,
                "message": exc.message,
                "details": exc.details
            }
        }
    )

@app.exception_handler(StudentHubError)
async def student_hub_error_handler(request: Request, exc: StudentHubError):
    return JSONResponse(
        status_code=500,
        content={
            "error": {
                "code": exc.code,
                "message": exc.message,
                "details": exc.details
            }
        }
    )
```

---

## 8. 트랜잭션 관리

### 8.1 Database Transaction Scope

#### 8.1.1 Service-Level Transactions
```python
# repositories/base.py
from contextlib import asynccontextmanager

class BaseRepository:
    @asynccontextmanager
    async def transaction(self):
        """트랜잭션 컨텍스트 매니저"""
        async with self.db.begin():
            try:
                yield
                # 정상 종료 시 자동 커밋
            except Exception:
                # 예외 발생 시 자동 롤백
                raise
```

#### 8.1.2 Complex Workflow Transaction
```python
# services/intervention_service.py
async def create_intervention_with_history(self, student_id: str, config: InterventionConfig):
    """
    개입 생성 + 학습 히스토리 기록 (트랜잭션)

    ACID 보장:
    - Atomicity: 모두 성공 또는 모두 실패
    - Consistency: 제약 조건 유지
    - Isolation: 다른 트랜잭션 격리
    - Durability: 커밋 후 영구 저장
    """
    async with self.intervention_repo.transaction():
        # 1. Intervention 생성
        intervention = await self.intervention_repo.create({...})

        # 2. Learning History 기록
        await self.learning_history_repo.create({
            "student_id": student_id,
            "event_type": "intervention",
            "source_node": 0,
            "source_id": str(intervention.id),
            "content": {
                "type": "intervention_created",
                "intervention_id": str(intervention.id)
            },
            "occurred_at": datetime.utcnow()
        })

        # 3. Student metadata 업데이트
        await self.student_repo.update(student_id, {
            "metadata": {
                **student.metadata,
                "last_intervention_at": datetime.utcnow().isoformat()
            }
        })

        # 트랜잭션 커밋 (자동)
        return intervention
```

### 8.2 Distributed Transaction (Saga Pattern - 향후)

```python
# tasks/sagas.py (계획)
from celery import chain, group

@celery_app.task
def intervention_saga(student_id: str, config: dict):
    """
    분산 트랜잭션 (Saga Pattern)

    Flow:
    1. Create Intervention (Node 0)
    2. Generate Problem Set (Node 3)
    3. Schedule Tasks (Node 0)

    Compensation (실패 시 롤백):
    - Step 3 실패 → Delete Problem Set → Delete Intervention
    """
    workflow = chain(
        create_intervention_step.s(student_id, config),
        generate_problem_set_step.s(),
        schedule_tasks_step.s()
    ).on_error(
        compensate_intervention_creation.s()
    )

    return workflow.apply_async()

@celery_app.task
def compensate_intervention_creation(intervention_id: str):
    """보상 트랜잭션 (Compensation)"""
    # 1. Delete scheduled tasks
    # 2. Delete problem set
    # 3. Mark intervention as cancelled
    pass
```

---

## 9. 테스트 전략

### 9.1 테스트 피라미드

```
        ┌────────────┐
        │    E2E     │ (10%)
        │   Tests    │ - Full workflow tests
        └────────────┘
      ┌──────────────────┐
      │  Integration     │ (30%)
      │   Tests          │ - API + DB + Cache
      └──────────────────┘
  ┌────────────────────────────┐
  │      Unit Tests            │ (60%)
  │  - Services, Repos, Utils  │
  └────────────────────────────┘
```

### 9.2 Unit Tests

#### 9.2.1 Service Layer Tests

```python
# tests/unit/services/test_profile_service.py
import pytest
from unittest.mock import AsyncMock, MagicMock
from app.services.profile_service import ProfileService

@pytest.fixture
def mock_student_repo():
    repo = AsyncMock()
    repo.get_by_id = AsyncMock(return_value=MagicMock(
        id="student-123",
        name="김철수",
        grade=10
    ))
    return repo

@pytest.fixture
def mock_cache_service():
    cache = AsyncMock()
    cache.get = AsyncMock(return_value=None)  # Cache miss
    cache.set = AsyncMock()
    return cache

@pytest.fixture
def mock_mcp_client():
    client = AsyncMock()
    client.call = AsyncMock(side_effect=[
        {"concepts": []},  # Node 1
        {"overall": 0.75}, # Node 2
        [],                # Node 4
        []                 # Node 5
    ])
    return client

@pytest.mark.asyncio
async def test_get_unified_profile_cache_miss(
    mock_student_repo,
    mock_cache_service,
    mock_mcp_client
):
    """통합 프로필 조회 (Cache Miss) 테스트"""
    # Given
    service = ProfileService(
        student_repo=mock_student_repo,
        cache_service=mock_cache_service,
        mcp_client=mock_mcp_client
    )

    # When
    profile = await service.get_unified_profile("student-123")

    # Then
    assert profile["student_id"] == "student-123"
    assert profile["basic_info"]["name"] == "김철수"
    assert profile["cached"] is False
    mock_cache_service.set.assert_called_once()
    assert mock_mcp_client.call.call_count == 4  # 4개 노드 호출
```

#### 9.2.2 Repository Layer Tests

```python
# tests/unit/repositories/test_student_repo.py
import pytest
from sqlalchemy.ext.asyncio import AsyncSession
from app.repositories.student_repo import StudentRepository
from app.models.student import Student

@pytest.mark.asyncio
async def test_create_student(db_session: AsyncSession):
    """학생 생성 테스트"""
    # Given
    repo = StudentRepository(Student, db_session)
    student_data = {
        "name": "김철수",
        "school_id": "SCH001",
        "grade": 10,
        "class_name": "1반",
        "student_number": "01"
    }

    # When
    student = await repo.create(student_data)

    # Then
    assert student.id is not None
    assert student.name == "김철수"
    assert student.grade == 10

@pytest.mark.asyncio
async def test_get_by_class(db_session: AsyncSession):
    """학급별 학생 조회 테스트"""
    # Given
    repo = StudentRepository(Student, db_session)
    # ... 테스트 데이터 생성

    # When
    students = await repo.get_by_class("SCH001", 10, "1반")

    # Then
    assert len(students) > 0
    assert all(s.grade == 10 for s in students)
```

### 9.3 Integration Tests

```python
# tests/integration/api/test_mcp_tools.py
import pytest
from httpx import AsyncClient
from app.main import app

@pytest.mark.asyncio
async def test_get_unified_profile_integration(async_client: AsyncClient):
    """통합 프로필 조회 통합 테스트 (API + DB + Cache)"""
    # Given: 테스트 학생 생성
    student_response = await async_client.post("/students", json={
        "name": "김철수",
        "school_id": "SCH001",
        "grade": 10
    })
    student_id = student_response.json()["id"]

    # When: MCP Tool 호출
    response = await async_client.post("/mcp/tools/call", json={
        "jsonrpc": "2.0",
        "id": 1,
        "method": "tools/call",
        "params": {
            "name": "get_unified_profile",
            "arguments": {
                "student_id": student_id,
                "include_history": True,
                "days": 30
            }
        }
    })

    # Then
    assert response.status_code == 200
    result = response.json()["result"]
    assert result["student_id"] == student_id
    assert result["basic_info"]["name"] == "김철수"
    assert "knowledge_state" in result
    assert "mastery_levels" in result
```

### 9.4 E2E Tests (End-to-End)

```python
# tests/e2e/test_intervention_workflow.py
import pytest
from httpx import AsyncClient

@pytest.mark.asyncio
@pytest.mark.e2e
async def test_full_intervention_workflow(async_client: AsyncClient):
    """
    개입 생성 → 진행 → 완료 전체 워크플로우 테스트
    """
    # 1. 학생 생성
    student_resp = await async_client.post("/students", json={...})
    student_id = student_resp.json()["id"]

    # 2. 개입 생성
    intervention_resp = await async_client.post("/mcp/tools/call", json={
        "jsonrpc": "2.0",
        "id": 1,
        "method": "tools/call",
        "params": {
            "name": "create_learning_intervention",
            "arguments": {
                "student_id": student_id,
                "type": "auto",
                "target_level": 0.85,
                "duration_days": 14
            }
        }
    })
    intervention_id = intervention_resp.json()["result"]["intervention_id"]

    # 3. 진행 상태 업데이트
    await async_client.put(f"/interventions/{intervention_id}/progress", json={
        "completed": 5
    })

    # 4. 개입 조회
    get_resp = await async_client.get(f"/interventions/{intervention_id}")
    intervention = get_resp.json()

    # 5. 검증
    assert intervention["status"] == "active"
    assert intervention["progress"]["completed"] == 5

    # 6. 완료 처리
    await async_client.put(f"/interventions/{intervention_id}/progress", json={
        "completed": intervention["progress"]["total"]
    })

    # 7. 최종 검증
    final_resp = await async_client.get(f"/interventions/{intervention_id}")
    final_intervention = final_resp.json()
    assert final_intervention["status"] == "completed"
    assert final_intervention["completed_at"] is not None
```

### 9.5 테스트 커버리지 목표

| 레이어 | 목표 커버리지 | 측정 도구 |
|--------|--------------|-----------|
| **Services** | > 90% | pytest-cov |
| **Repositories** | > 85% | pytest-cov |
| **Models** | > 80% | pytest-cov |
| **API Endpoints** | > 75% | pytest-cov |
| **Overall** | > 80% | pytest-cov |

---

## 10. 배포 전략

### 10.1 Docker 구성

#### 10.1.1 Dockerfile

```dockerfile
# docker/Dockerfile
FROM python:3.11-slim

WORKDIR /app

# Install system dependencies
RUN apt-get update && apt-get install -y \
    build-essential \
    && rm -rf /var/lib/apt/lists/*

# Install Python dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application
COPY app/ app/

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=40s --retries=3 \
    CMD python -c "import httpx; httpx.get('http://localhost:8000/health')"

# Run application
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000", "--workers", "4"]
```

#### 10.1.2 docker-compose.yml

```yaml
version: '3.8'

services:
  student-hub-api:
    build:
      context: .
      dockerfile: docker/Dockerfile
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=postgresql+asyncpg://postgres:password@postgres:5432/mathesis_hub
      - REDIS_URL=redis://redis:6379/0
    depends_on:
      - postgres
      - redis
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
    networks:
      - mathesis-network

  student-hub-worker:
    build:
      context: .
      dockerfile: docker/Dockerfile
    command: celery -A app.tasks.celery_app worker -Q default,high_priority,low_priority -c 4
    environment:
      - DATABASE_URL=postgresql+asyncpg://postgres:password@postgres:5432/mathesis_hub
      - CELERY_BROKER_URL=redis://redis:6379/1
      - CELERY_RESULT_BACKEND=redis://redis:6379/2
    depends_on:
      - postgres
      - redis
    networks:
      - mathesis-network

  student-hub-beat:
    build:
      context: .
      dockerfile: docker/Dockerfile
    command: celery -A app.tasks.celery_app beat --loglevel=info
    environment:
      - CELERY_BROKER_URL=redis://redis:6379/1
      - CELERY_RESULT_BACKEND=redis://redis:6379/2
    depends_on:
      - redis
    networks:
      - mathesis-network

  postgres:
    image: postgres:14
    environment:
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=password
      - POSTGRES_DB=mathesis_hub
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    networks:
      - mathesis-network

  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes
    volumes:
      - redis_data:/data
    ports:
      - "6379:6379"
    networks:
      - mathesis-network

volumes:
  postgres_data:
  redis_data:

networks:
  mathesis-network:
    driver: bridge
```

### 10.2 배포 단계

#### Phase 1: Development
```bash
# Local development
docker-compose -f docker-compose.dev.yml up -d

# Run migrations
docker-compose exec student-hub-api alembic upgrade head

# Run tests
docker-compose exec student-hub-api pytest
```

#### Phase 2: Staging
```bash
# Build images
docker-compose -f docker-compose.staging.yml build

# Deploy to staging
docker-compose -f docker-compose.staging.yml up -d

# Health check
curl http://staging.mathesis.io:8000/health
```

#### Phase 3: Production (Kubernetes - 향후)
```yaml
# k8s/deployment.yaml (계획)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: student-hub-api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: student-hub-api
  template:
    metadata:
      labels:
        app: student-hub-api
    spec:
      containers:
      - name: api
        image: mathesis/student-hub:1.0.0
        ports:
        - containerPort: 8000
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: student-hub-secrets
              key: database-url
```

### 10.3 CI/CD Pipeline (계획)

```yaml
# .github/workflows/deploy.yml
name: Deploy Student Hub

on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Run tests
        run: |
          docker-compose -f docker-compose.test.yml up --abort-on-container-exit
          docker-compose -f docker-compose.test.yml down

  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Build Docker image
        run: docker build -t mathesis/student-hub:${{ github.sha }} .
      - name: Push to registry
        run: docker push mathesis/student-hub:${{ github.sha }}

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to production
        run: |
          # kubectl apply -f k8s/
          # kubectl rollout status deployment/student-hub-api
```

---

## 부록

### A. 코드 품질 도구

| 도구 | 목적 | 설정 |
|------|------|------|
| **black** | Code formatting | line-length = 120 |
| **isort** | Import sorting | profile = "black" |
| **mypy** | Type checking | strict = true |
| **flake8** | Linting | max-line-length = 120 |
| **pytest** | Testing | testpaths = ["tests"] |
| **pytest-cov** | Coverage | min_coverage = 80% |

### B. 개발 환경 설정

```toml
# pyproject.toml
[tool.poetry]
name = "node0-student-hub"
version = "1.0.0"
description = "Mathesis Student Hub (Node 0)"
authors = ["Mathesis Team"]

[tool.poetry.dependencies]
python = "^3.11"
fastapi = "^0.104.0"
uvicorn = {extras = ["standard"], version = "^0.24.0"}
sqlalchemy = "^2.0.0"
asyncpg = "^0.29.0"
redis = "^5.0.0"
celery = "^5.3.0"
pydantic = "^2.5.0"
httpx = "^0.25.0"
structlog = "^23.2.0"
prometheus-client = "^0.18.0"

[tool.poetry.dev-dependencies]
pytest = "^7.4.0"
pytest-asyncio = "^0.21.0"
pytest-cov = "^4.1.0"
black = "^23.11.0"
isort = "^5.12.0"
mypy = "^1.7.0"
flake8 = "^6.1.0"

[tool.black]
line-length = 120

[tool.isort]
profile = "black"
line_length = 120

[tool.mypy]
python_version = "3.11"
strict = true
```

### C. 참고 문서

- [FastAPI Documentation](https://fastapi.tiangolo.com/)
- [SQLAlchemy 2.0 Documentation](https://docs.sqlalchemy.org/)
- [Celery Documentation](https://docs.celeryq.dev/)
- [Redis Documentation](https://redis.io/documentation)
- [MCP Protocol Specification](../architecture/MCP_PROTOCOL.md)

---

**Document Version**: 1.0.0
**Last Updated**: 2026-01-09
**Review Date**: 2026-02-09
**Approved By**: [Pending]
