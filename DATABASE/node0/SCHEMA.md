# Node 0: Student Hub - Database Schema

**Version**: 1.0.0
**Last Updated**: 2026-01-09
**Database**: PostgreSQL 14+
**Migration Tool**: Alembic

---

## 📋 Table of Contents

1. [데이터베이스 개요](#1-데이터베이스-개요)
2. [ERD (Entity Relationship Diagram)](#2-erd-entity-relationship-diagram)
3. [테이블 상세 스키마](#3-테이블-상세-스키마)
4. [인덱스 전략](#4-인덱스-전략)
5. [파티셔닝 전략](#5-파티셔닝-전략)
6. [제약 조건](#6-제약-조건)
7. [마이그레이션 전략](#7-마이그레이션-전략)
8. [데이터 보존 정책](#8-데이터-보존-정책)
9. [백업 및 복구](#9-백업-및-복구)
10. [성능 튜닝](#10-성능-튜닝)

---

## 1. 데이터베이스 개요

### 1.1 기본 정보

| 항목 | 값 |
|------|-----|
| **DBMS** | PostgreSQL 14+ |
| **Character Set** | UTF-8 |
| **Collation** | ko_KR.UTF-8 |
| **Timezone** | UTC |
| **Connection Pool** | 20 (min), 30 (max) |

### 1.2 데이터베이스 생성

```sql
-- Database 생성
CREATE DATABASE mathesis_hub
    WITH
    OWNER = postgres
    ENCODING = 'UTF8'
    LC_COLLATE = 'ko_KR.UTF-8'
    LC_CTYPE = 'ko_KR.UTF-8'
    TABLESPACE = pg_default
    CONNECTION LIMIT = -1;

-- Extensions 설치
\c mathesis_hub

CREATE EXTENSION IF NOT EXISTS "uuid-ossp";      -- UUID 생성
CREATE EXTENSION IF NOT EXISTS "pg_trgm";        -- 유사도 검색
CREATE EXTENSION IF NOT EXISTS "btree_gin";      -- GIN 인덱스 최적화
```

### 1.3 스키마 구조

```
mathesis_hub
├── public (기본 스키마)
│   ├── students (학생 마스터 데이터)
│   ├── learning_history (학습 이벤트 - 파티션)
│   │   ├── learning_history_2026_01
│   │   ├── learning_history_2026_02
│   │   └── ...
│   ├── interventions (학습 개입)
│   └── scheduled_tasks (예약 작업)
│
└── audit (감사 로그 - 향후)
    └── audit_logs
```

---

## 2. ERD (Entity Relationship Diagram)

### 2.1 개념적 ERD

```
┌─────────────────────────┐
│       STUDENTS          │ (마스터 데이터)
│─────────────────────────│
│ PK: id (UUID)           │
│     name                │
│     school_id           │
│     grade               │
│     class_name          │
│     student_number      │
│     email               │
│     parent_contact      │
│     metadata (JSONB)    │
│     created_at          │
│     updated_at          │
└────────┬────────────────┘
         │ 1
         │
         │ N
         ↓
┌─────────────────────────────────────────┐
│       LEARNING_HISTORY                  │ (시계열 데이터, 파티션)
│─────────────────────────────────────────│
│ PK: (id, occurred_at)                   │
│ FK: student_id → students.id            │
│     event_type (study, test, ...)      │
│     source_node (1-6)                   │
│     source_id                           │
│     content (JSONB)                     │
│     occurred_at (파티션 키)             │
│     created_at                          │
└─────────────────────────────────────────┘

         │
         │
         ↓
┌─────────────────────────────────────────┐
│       INTERVENTIONS                     │ (학습 개입)
│─────────────────────────────────────────│
│ PK: id (UUID)                           │
│ FK: student_id → students.id            │
│     type (auto, teacher_requested)      │
│     weak_areas (JSONB)                  │
│     learning_path (JSONB)               │
│     status (active, paused, ...)        │
│     progress (JSONB)                    │
│     created_at                          │
│     completed_at                        │
└─────────────────────────────────────────┘

         │
         │
         ↓
┌─────────────────────────────────────────┐
│       SCHEDULED_TASKS                   │ (주기적 작업)
│─────────────────────────────────────────│
│ PK: id (UUID)                           │
│ FK: student_id → students.id (nullable) │
│     task_type (daily_report, ...)       │
│     schedule_type (cron, interval, ...) │
│     cron_expression                     │
│     config (JSONB)                      │
│     status (active, paused, ...)        │
│     next_run_at                         │
│     last_run_at                         │
│     celery_task_id                      │
│     created_at                          │
└─────────────────────────────────────────┘
```

### 2.2 관계 정리

| 부모 테이블 | 자식 테이블 | 관계 | ON DELETE |
|-------------|-------------|------|-----------|
| `students` | `learning_history` | 1:N | CASCADE |
| `students` | `interventions` | 1:N | CASCADE |
| `students` | `scheduled_tasks` | 1:N | CASCADE (student_id는 nullable) |

---

## 3. 테이블 상세 스키마

### 3.1 students (학생 마스터 데이터)

**목적**: 학생의 기본 정보 및 마스터 데이터 관리

**DDL**:
```sql
CREATE TABLE students (
    -- Primary Key
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    -- Basic Info
    name VARCHAR(100) NOT NULL,
    school_id VARCHAR(50) NOT NULL,
    grade INTEGER NOT NULL,
    class_name VARCHAR(50),
    student_number VARCHAR(20),

    -- Contact (암호화 권장)
    email VARCHAR(255),
    parent_contact VARCHAR(20),

    -- Metadata (확장 가능한 JSON 필드)
    metadata JSONB DEFAULT '{}'::JSONB NOT NULL,

    -- Timestamps
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW(),

    -- Constraints
    CONSTRAINT students_grade_check CHECK (grade >= 1 AND grade <= 12),
    CONSTRAINT students_unique_identifier UNIQUE (school_id, grade, class_name, student_number)
);

-- Indexes
CREATE INDEX idx_students_school_id ON students(school_id);
CREATE INDEX idx_students_grade_class ON students(grade, class_name);
CREATE INDEX idx_students_created_at ON students(created_at DESC);
CREATE INDEX idx_students_metadata_gin ON students USING GIN (metadata);

-- Comments
COMMENT ON TABLE students IS '학생 마스터 데이터 (Single Source of Truth)';
COMMENT ON COLUMN students.id IS '학생 고유 ID (UUID)';
COMMENT ON COLUMN students.name IS '학생 이름 (암호화 권장)';
COMMENT ON COLUMN students.metadata IS '확장 가능한 메타데이터 (학습 스타일, 관심사 등)';
```

**예시 데이터**:
```sql
INSERT INTO students (name, school_id, grade, class_name, student_number, email, parent_contact, metadata)
VALUES (
    '김철수',
    'SCH001',
    10,
    '1반',
    '01',
    'student@example.com',
    '010-1234-5678',
    '{
        "learning_style": "visual",
        "special_needs": false,
        "interests": ["math", "science"],
        "parent_preferences": {
            "notification_method": "email",
            "report_frequency": "weekly"
        }
    }'::JSONB
);
```

### 3.2 learning_history (학습 이벤트 - 시계열 데이터)

**목적**: 학생의 학습 활동 이벤트 저장 (문제 풀이, 시험, 개입 등)

**DDL**:
```sql
-- Parent Table (파티션 부모)
CREATE TABLE learning_history (
    -- Primary Key (Composite for Partitioning)
    id BIGSERIAL,
    occurred_at TIMESTAMP NOT NULL,

    -- Foreign Key
    student_id UUID NOT NULL,

    -- Event Info
    event_type VARCHAR(50) NOT NULL,
    source_node INTEGER NOT NULL,
    source_id VARCHAR(255),

    -- Event Content (JSONB)
    content JSONB NOT NULL,

    -- Timestamp
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),

    -- Primary Key (id, occurred_at for partitioning)
    PRIMARY KEY (id, occurred_at),

    -- Foreign Key Constraint
    FOREIGN KEY (student_id) REFERENCES students(id) ON DELETE CASCADE,

    -- Constraints
    CONSTRAINT learning_history_source_node_check CHECK (source_node >= 1 AND source_node <= 6),
    CONSTRAINT learning_history_event_type_check CHECK (
        event_type IN ('study', 'test', 'intervention', 'review', 'assessment')
    )
) PARTITION BY RANGE (occurred_at);

-- Indexes (on parent)
CREATE INDEX idx_lh_student_id ON learning_history(student_id);
CREATE INDEX idx_lh_event_type ON learning_history(event_type);
CREATE INDEX idx_lh_occurred_at ON learning_history(occurred_at DESC);
CREATE INDEX idx_lh_content_gin ON learning_history USING GIN (content);

-- Comments
COMMENT ON TABLE learning_history IS '학습 이벤트 로그 (시계열 데이터, 월별 파티션)';
COMMENT ON COLUMN learning_history.occurred_at IS '이벤트 발생 시각 (파티션 키)';
COMMENT ON COLUMN learning_history.source_node IS '이벤트 발생 노드 (1-6)';
COMMENT ON COLUMN learning_history.content IS '이벤트 상세 데이터 (JSONB)';
```

**월별 파티션 생성**:
```sql
-- 2026년 1월 파티션
CREATE TABLE learning_history_2026_01 PARTITION OF learning_history
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');

-- 2026년 2월 파티션
CREATE TABLE learning_history_2026_02 PARTITION OF learning_history
    FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');

-- 2026년 3월 파티션
CREATE TABLE learning_history_2026_03 PARTITION OF learning_history
    FOR VALUES FROM ('2026-03-01') TO ('2026-04-01');

-- ... (나머지 파티션은 자동화 스크립트로 생성)
```

**예시 데이터**:
```sql
INSERT INTO learning_history (student_id, event_type, source_node, source_id, content, occurred_at)
VALUES (
    '550e8400-e29b-41d4-a716-446655440000',
    'study',
    4,  -- Lab Node
    'LAB_SESSION_123',
    '{
        "action": "solved_problem",
        "problem_id": "PROB_456",
        "result": "correct",
        "time_spent": 180,
        "difficulty": 0.65,
        "hints_used": 1
    }'::JSONB,
    '2026-01-08 14:30:00'
);
```

### 3.3 interventions (학습 개입)

**목적**: 학생에 대한 학습 개입 계획 및 진행 상태 관리

**DDL**:
```sql
CREATE TABLE interventions (
    -- Primary Key
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    -- Foreign Key
    student_id UUID NOT NULL,

    -- Intervention Info
    type VARCHAR(50) NOT NULL,
    weak_areas JSONB NOT NULL,
    learning_path JSONB NOT NULL,

    -- Status
    status VARCHAR(20) NOT NULL DEFAULT 'active',
    progress JSONB DEFAULT '{"completed": 0, "total": 0}'::JSONB NOT NULL,

    -- Timestamps
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    completed_at TIMESTAMP,

    -- Foreign Key Constraint
    FOREIGN KEY (student_id) REFERENCES students(id) ON DELETE CASCADE,

    -- Constraints
    CONSTRAINT interventions_type_check CHECK (type IN ('auto', 'teacher_requested')),
    CONSTRAINT interventions_status_check CHECK (status IN ('active', 'paused', 'completed', 'cancelled'))
);

-- Indexes
CREATE INDEX idx_interventions_student_id ON interventions(student_id);
CREATE INDEX idx_interventions_status ON interventions(status);
CREATE INDEX idx_interventions_created_at ON interventions(created_at DESC);
CREATE INDEX idx_interventions_student_status ON interventions(student_id, status);

-- Comments
COMMENT ON TABLE interventions IS '학습 개입 (자동/교사 요청)';
COMMENT ON COLUMN interventions.weak_areas IS '약점 영역 분석 결과 (JSONB)';
COMMENT ON COLUMN interventions.learning_path IS '생성된 학습 경로 (JSONB)';
COMMENT ON COLUMN interventions.progress IS '진행 상태 {completed: N, total: M}';
```

**예시 데이터**:
```sql
INSERT INTO interventions (student_id, type, weak_areas, learning_path, status, progress)
VALUES (
    '550e8400-e29b-41d4-a716-446655440000',
    'auto',
    '[
        {
            "concept": "이차함수",
            "current_mastery": 0.65,
            "target_mastery": 0.85,
            "priority": 1
        },
        {
            "concept": "삼각함수",
            "current_mastery": 0.70,
            "target_mastery": 0.85,
            "priority": 2
        }
    ]'::JSONB,
    '[
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
            "num_problems": 10,
            "estimated_duration": 2400
        }
    ]'::JSONB,
    'active',
    '{"completed": 0, "total": 2}'::JSONB
);
```

### 3.4 scheduled_tasks (예약 작업)

**목적**: 주기적/일회성 작업 스케줄링 및 실행 이력 관리

**DDL**:
```sql
CREATE TABLE scheduled_tasks (
    -- Primary Key
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    -- Foreign Key (nullable for global tasks)
    student_id UUID,

    -- Task Info
    task_type VARCHAR(50) NOT NULL,
    schedule_type VARCHAR(20) NOT NULL,
    cron_expression VARCHAR(100),
    config JSONB DEFAULT '{}'::JSONB NOT NULL,

    -- Status
    status VARCHAR(20) NOT NULL DEFAULT 'active',
    next_run_at TIMESTAMP,
    last_run_at TIMESTAMP,
    celery_task_id VARCHAR(255),

    -- Timestamp
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),

    -- Foreign Key Constraint
    FOREIGN KEY (student_id) REFERENCES students(id) ON DELETE CASCADE,

    -- Constraints
    CONSTRAINT scheduled_tasks_schedule_type_check CHECK (
        schedule_type IN ('cron', 'interval', 'one_time')
    ),
    CONSTRAINT scheduled_tasks_status_check CHECK (
        status IN ('active', 'paused', 'completed')
    )
);

-- Indexes
CREATE INDEX idx_scheduled_tasks_student_id ON scheduled_tasks(student_id);
CREATE INDEX idx_scheduled_tasks_next_run ON scheduled_tasks(next_run_at) WHERE status = 'active';
CREATE INDEX idx_scheduled_tasks_type_status ON scheduled_tasks(task_type, status);
CREATE INDEX idx_scheduled_tasks_created_at ON scheduled_tasks(created_at DESC);

-- Comments
COMMENT ON TABLE scheduled_tasks IS '주기적/일회성 작업 스케줄';
COMMENT ON COLUMN scheduled_tasks.student_id IS '학생 ID (NULL이면 전역 작업)';
COMMENT ON COLUMN scheduled_tasks.cron_expression IS 'Cron 표현식 (schedule_type=cron)';
COMMENT ON COLUMN scheduled_tasks.celery_task_id IS 'Celery Task ID';
```

**예시 데이터**:
```sql
-- 학생별 일일 리포트 (매일 오전 9시)
INSERT INTO scheduled_tasks (student_id, task_type, schedule_type, cron_expression, config, next_run_at)
VALUES (
    '550e8400-e29b-41d4-a716-446655440000',
    'daily_report',
    'cron',
    '0 9 * * *',
    '{
        "report_format": "pdf",
        "include_sections": ["progress", "recommendations"]
    }'::JSONB,
    '2026-01-10 09:00:00'
);

-- 전역 주간 분석 (매주 월요일 오전 10시)
INSERT INTO scheduled_tasks (student_id, task_type, schedule_type, cron_expression, config, next_run_at)
VALUES (
    NULL,  -- 전역 작업
    'weekly_analytics',
    'cron',
    '0 10 * * 1',
    '{
        "scope": "all_schools",
        "metrics": ["engagement", "performance"]
    }'::JSONB,
    '2026-01-13 10:00:00'
);
```

---

## 4. 인덱스 전략

### 4.1 인덱스 유형별 사용

| 인덱스 타입 | 사용 예시 | 장점 |
|-------------|----------|------|
| **B-Tree** (기본) | Primary Key, Foreign Key, 범위 검색 | 범용적, 정렬 가능 |
| **GIN** (Generalized Inverted Index) | JSONB 필드, 텍스트 검색 | JSON 쿼리 최적화 |
| **Partial Index** | `status = 'active'` 조건 | 인덱스 크기 감소 |
| **Composite Index** | (student_id, status) | 다중 컬럼 쿼리 최적화 |

### 4.2 인덱스 목록

#### students 테이블
```sql
-- Primary Key (자동 생성)
-- UNIQUE: id (UUID)

-- Foreign Key 조회
CREATE INDEX idx_students_school_id ON students(school_id);

-- 학년/반별 조회
CREATE INDEX idx_students_grade_class ON students(grade, class_name);

-- 시간순 조회
CREATE INDEX idx_students_created_at ON students(created_at DESC);

-- JSONB 검색 (메타데이터)
CREATE INDEX idx_students_metadata_gin ON students USING GIN (metadata);

-- 이름 검색 (LIKE 쿼리 최적화)
CREATE INDEX idx_students_name_trgm ON students USING GIN (name gin_trgm_ops);
```

#### learning_history 테이블
```sql
-- 학생별 이벤트 조회 (최신순)
CREATE INDEX idx_lh_student_id ON learning_history(student_id);
CREATE INDEX idx_lh_student_occurred ON learning_history(student_id, occurred_at DESC);

-- 이벤트 타입별 조회
CREATE INDEX idx_lh_event_type ON learning_history(event_type);

-- 시간 범위 조회
CREATE INDEX idx_lh_occurred_at ON learning_history(occurred_at DESC);

-- JSONB 검색 (content)
CREATE INDEX idx_lh_content_gin ON learning_history USING GIN (content);

-- 복합 인덱스 (학생 + 이벤트 타입)
CREATE INDEX idx_lh_student_event ON learning_history(student_id, event_type, occurred_at DESC);
```

#### interventions 테이블
```sql
-- 학생별 조회
CREATE INDEX idx_interventions_student_id ON interventions(student_id);

-- 상태별 조회
CREATE INDEX idx_interventions_status ON interventions(status);

-- 최신순 조회
CREATE INDEX idx_interventions_created_at ON interventions(created_at DESC);

-- 복합 인덱스 (학생 + 상태)
CREATE INDEX idx_interventions_student_status ON interventions(student_id, status);

-- 활성 개입만 조회 (Partial Index)
CREATE INDEX idx_interventions_active ON interventions(student_id, created_at DESC) WHERE status = 'active';
```

#### scheduled_tasks 테이블
```sql
-- 학생별 조회
CREATE INDEX idx_scheduled_tasks_student_id ON scheduled_tasks(student_id);

-- 다음 실행 시각 조회 (active만)
CREATE INDEX idx_scheduled_tasks_next_run ON scheduled_tasks(next_run_at) WHERE status = 'active';

-- 작업 타입 + 상태
CREATE INDEX idx_scheduled_tasks_type_status ON scheduled_tasks(task_type, status);

-- 최신순 조회
CREATE INDEX idx_scheduled_tasks_created_at ON scheduled_tasks(created_at DESC);
```

### 4.3 인덱스 모니터링

```sql
-- 인덱스 사용률 확인
SELECT
    schemaname,
    tablename,
    indexname,
    idx_scan AS index_scans,
    idx_tup_read AS tuples_read,
    idx_tup_fetch AS tuples_fetched
FROM pg_stat_user_indexes
WHERE schemaname = 'public'
ORDER BY idx_scan ASC;

-- 사용되지 않는 인덱스 찾기
SELECT
    schemaname,
    tablename,
    indexname
FROM pg_stat_user_indexes
WHERE idx_scan = 0
    AND indexrelname NOT LIKE '%_pkey';
```

---

## 5. 파티셔닝 전략

### 5.1 learning_history 파티션 (Range by occurred_at)

**전략**: 월별 파티션 (RANGE partitioning)

**이유**:
- 시계열 데이터 특성상 최근 데이터 조회 빈도가 높음
- 오래된 데이터는 별도 파티션으로 분리하여 아카이빙 가능
- 파티션 단위 백업/복구 용이

#### 자동 파티션 생성 함수

```sql
-- 월별 파티션 자동 생성 함수
CREATE OR REPLACE FUNCTION create_learning_history_partition(partition_date DATE)
RETURNS VOID AS $$
DECLARE
    partition_name TEXT;
    start_date DATE;
    end_date DATE;
BEGIN
    partition_name := 'learning_history_' || TO_CHAR(partition_date, 'YYYY_MM');
    start_date := DATE_TRUNC('month', partition_date);
    end_date := start_date + INTERVAL '1 month';

    -- 파티션이 이미 존재하는지 확인
    IF NOT EXISTS (
        SELECT 1 FROM pg_class WHERE relname = partition_name
    ) THEN
        EXECUTE FORMAT(
            'CREATE TABLE %I PARTITION OF learning_history FOR VALUES FROM (%L) TO (%L)',
            partition_name,
            start_date,
            end_date
        );

        RAISE NOTICE 'Partition % created for range % to %', partition_name, start_date, end_date;
    ELSE
        RAISE NOTICE 'Partition % already exists', partition_name;
    END IF;
END;
$$ LANGUAGE plpgsql;

-- 향후 6개월치 파티션 미리 생성
DO $$
DECLARE
    month_offset INTEGER;
BEGIN
    FOR month_offset IN 0..5 LOOP
        PERFORM create_learning_history_partition(CURRENT_DATE + (month_offset || ' months')::INTERVAL);
    END LOOP;
END $$;
```

#### 파티션 자동 정리 (아카이빙)

```sql
-- 2년 이상 오래된 파티션을 아카이브 테이블로 이동
CREATE OR REPLACE FUNCTION archive_old_learning_history_partitions()
RETURNS VOID AS $$
DECLARE
    partition_record RECORD;
    archive_date DATE;
BEGIN
    archive_date := CURRENT_DATE - INTERVAL '2 years';

    FOR partition_record IN
        SELECT tablename
        FROM pg_tables
        WHERE schemaname = 'public'
            AND tablename LIKE 'learning_history_20%'
            AND tablename < 'learning_history_' || TO_CHAR(archive_date, 'YYYY_MM')
    LOOP
        -- 아카이브 스키마로 이동 (또는 외부 저장소로 백업)
        EXECUTE FORMAT('ALTER TABLE %I SET SCHEMA archive', partition_record.tablename);
        RAISE NOTICE 'Partition % archived', partition_record.tablename;
    END LOOP;
END;
$$ LANGUAGE plpgsql;
```

### 5.2 파티션 유지보수

```sql
-- Cron으로 매월 1일에 실행
-- 1. 새 파티션 생성
SELECT create_learning_history_partition(CURRENT_DATE + INTERVAL '6 months');

-- 2. 오래된 파티션 아카이빙 (2년 이상)
SELECT archive_old_learning_history_partitions();
```

---

## 6. 제약 조건

### 6.1 Primary Key Constraints

```sql
-- students
ALTER TABLE students ADD CONSTRAINT students_pkey PRIMARY KEY (id);

-- learning_history (composite)
ALTER TABLE learning_history ADD CONSTRAINT learning_history_pkey PRIMARY KEY (id, occurred_at);

-- interventions
ALTER TABLE interventions ADD CONSTRAINT interventions_pkey PRIMARY KEY (id);

-- scheduled_tasks
ALTER TABLE scheduled_tasks ADD CONSTRAINT scheduled_tasks_pkey PRIMARY KEY (id);
```

### 6.2 Foreign Key Constraints

```sql
-- learning_history → students
ALTER TABLE learning_history
ADD CONSTRAINT fk_learning_history_student
FOREIGN KEY (student_id) REFERENCES students(id) ON DELETE CASCADE;

-- interventions → students
ALTER TABLE interventions
ADD CONSTRAINT fk_interventions_student
FOREIGN KEY (student_id) REFERENCES students(id) ON DELETE CASCADE;

-- scheduled_tasks → students
ALTER TABLE scheduled_tasks
ADD CONSTRAINT fk_scheduled_tasks_student
FOREIGN KEY (student_id) REFERENCES students(id) ON DELETE CASCADE;
```

### 6.3 Unique Constraints

```sql
-- students: school_id + grade + class_name + student_number는 유일해야 함
ALTER TABLE students
ADD CONSTRAINT students_unique_identifier
UNIQUE (school_id, grade, class_name, student_number);
```

### 6.4 Check Constraints

```sql
-- students: grade는 1-12 사이
ALTER TABLE students
ADD CONSTRAINT students_grade_check
CHECK (grade >= 1 AND grade <= 12);

-- learning_history: source_node는 1-6 사이
ALTER TABLE learning_history
ADD CONSTRAINT learning_history_source_node_check
CHECK (source_node >= 1 AND source_node <= 6);

-- learning_history: event_type 제한
ALTER TABLE learning_history
ADD CONSTRAINT learning_history_event_type_check
CHECK (event_type IN ('study', 'test', 'intervention', 'review', 'assessment'));

-- interventions: type 제한
ALTER TABLE interventions
ADD CONSTRAINT interventions_type_check
CHECK (type IN ('auto', 'teacher_requested'));

-- interventions: status 제한
ALTER TABLE interventions
ADD CONSTRAINT interventions_status_check
CHECK (status IN ('active', 'paused', 'completed', 'cancelled'));

-- scheduled_tasks: schedule_type 제한
ALTER TABLE scheduled_tasks
ADD CONSTRAINT scheduled_tasks_schedule_type_check
CHECK (schedule_type IN ('cron', 'interval', 'one_time'));

-- scheduled_tasks: status 제한
ALTER TABLE scheduled_tasks
ADD CONSTRAINT scheduled_tasks_status_check
CHECK (status IN ('active', 'paused', 'completed'));
```

### 6.5 NOT NULL Constraints

```sql
-- 모든 테이블에서 필수 필드는 NOT NULL
-- (DDL에서 이미 정의됨)
```

---

## 7. 마이그레이션 전략

### 7.1 Alembic 설정

**초기화**:
```bash
# Alembic 초기화
alembic init alembic

# alembic.ini 수정
sqlalchemy.url = postgresql+asyncpg://user:pass@localhost:5432/mathesis_hub
```

**env.py 설정**:
```python
# alembic/env.py
from app.db.base import Base
from app.models.student import Student
from app.models.learning_history import LearningHistory
from app.models.intervention import Intervention
from app.models.task import ScheduledTask

target_metadata = Base.metadata
```

### 7.2 Migration 생성

```bash
# 자동 생성
alembic revision --autogenerate -m "Initial schema"

# 수동 생성
alembic revision -m "Add index on students.name"
```

### 7.3 Migration 파일 예시

```python
# alembic/versions/001_initial_schema.py
"""Initial schema

Revision ID: 001
Revises:
Create Date: 2026-01-09 05:30:00.000000
"""

from alembic import op
import sqlalchemy as sa
from sqlalchemy.dialects import postgresql

# revision identifiers
revision = '001'
down_revision = None
branch_labels = None
depends_on = None

def upgrade():
    # students 테이블 생성
    op.create_table(
        'students',
        sa.Column('id', postgresql.UUID(as_uuid=True), primary_key=True, server_default=sa.text('gen_random_uuid()')),
        sa.Column('name', sa.String(100), nullable=False),
        sa.Column('school_id', sa.String(50), nullable=False),
        sa.Column('grade', sa.Integer, nullable=False),
        sa.Column('class_name', sa.String(50)),
        sa.Column('student_number', sa.String(20)),
        sa.Column('email', sa.String(255)),
        sa.Column('parent_contact', sa.String(20)),
        sa.Column('metadata', postgresql.JSONB, nullable=False, server_default='{}'),
        sa.Column('created_at', sa.TIMESTAMP, nullable=False, server_default=sa.text('NOW()')),
        sa.Column('updated_at', sa.TIMESTAMP, nullable=False, server_default=sa.text('NOW()')),
        sa.CheckConstraint('grade >= 1 AND grade <= 12', name='students_grade_check'),
        sa.UniqueConstraint('school_id', 'grade', 'class_name', 'student_number', name='students_unique_identifier')
    )

    # 인덱스 생성
    op.create_index('idx_students_school_id', 'students', ['school_id'])
    op.create_index('idx_students_grade_class', 'students', ['grade', 'class_name'])
    op.create_index('idx_students_created_at', 'students', ['created_at'], postgresql_using='btree', postgresql_ops={'created_at': 'DESC'})

    # learning_history 테이블 생성 (파티션)
    op.execute("""
        CREATE TABLE learning_history (
            id BIGSERIAL,
            occurred_at TIMESTAMP NOT NULL,
            student_id UUID NOT NULL,
            event_type VARCHAR(50) NOT NULL,
            source_node INTEGER NOT NULL,
            source_id VARCHAR(255),
            content JSONB NOT NULL,
            created_at TIMESTAMP NOT NULL DEFAULT NOW(),
            PRIMARY KEY (id, occurred_at),
            FOREIGN KEY (student_id) REFERENCES students(id) ON DELETE CASCADE,
            CHECK (source_node >= 1 AND source_node <= 6),
            CHECK (event_type IN ('study', 'test', 'intervention', 'review', 'assessment'))
        ) PARTITION BY RANGE (occurred_at);
    """)

    # 초기 파티션 생성 (2026년 1-6월)
    op.execute("""
        CREATE TABLE learning_history_2026_01 PARTITION OF learning_history
            FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
    """)
    # ... (나머지 파티션)

    # interventions 테이블 생성
    # ...

    # scheduled_tasks 테이블 생성
    # ...

def downgrade():
    op.drop_table('scheduled_tasks')
    op.drop_table('interventions')
    op.execute('DROP TABLE learning_history CASCADE')
    op.drop_table('students')
```

### 7.4 Migration 실행

```bash
# 현재 상태 확인
alembic current

# 마이그레이션 실행 (최신 버전으로)
alembic upgrade head

# 특정 버전으로 업그레이드
alembic upgrade 001

# 다운그레이드
alembic downgrade -1

# 히스토리 확인
alembic history
```

---

## 8. 데이터 보존 정책

### 8.1 보존 기간

| 테이블 | 보존 기간 | 정책 |
|--------|----------|------|
| **students** | 영구 | 졸업 후 5년까지 보관, 이후 익명화 |
| **learning_history** | 2년 | 2년 이상 데이터는 아카이브 스키마로 이동 |
| **interventions** | 영구 | 완료된 개입도 분석을 위해 보관 |
| **scheduled_tasks** | 1년 | 완료된 작업은 1년 후 삭제 |

### 8.2 아카이빙 전략

```sql
-- 아카이브 스키마 생성
CREATE SCHEMA IF NOT EXISTS archive;

-- 오래된 파티션 이동
ALTER TABLE learning_history_2024_01 SET SCHEMA archive;

-- 아카이브 데이터 조회 (필요 시)
SELECT * FROM archive.learning_history_2024_01 WHERE student_id = '...';
```

### 8.3 데이터 익명화 (GDPR 준수)

```sql
-- 졸업 후 5년이 지난 학생 데이터 익명화
CREATE OR REPLACE FUNCTION anonymize_old_students()
RETURNS VOID AS $$
BEGIN
    UPDATE students
    SET
        name = 'ANONYMIZED_' || id,
        email = NULL,
        parent_contact = NULL,
        metadata = '{}'::JSONB
    WHERE created_at < CURRENT_DATE - INTERVAL '5 years';
END;
$$ LANGUAGE plpgsql;
```

---

## 9. 백업 및 복구

### 9.1 백업 전략

#### 전체 백업 (Daily)
```bash
# 전체 데이터베이스 백업
pg_dump -U postgres -Fc mathesis_hub > mathesis_hub_$(date +%Y%m%d).dump

# 압축 백업
pg_dump -U postgres -Fc -Z9 mathesis_hub > mathesis_hub_$(date +%Y%m%d).dump.gz
```

#### 테이블별 백업
```bash
# students 테이블만 백업
pg_dump -U postgres -t students mathesis_hub > students_$(date +%Y%m%d).sql

# 파티션별 백업
pg_dump -U postgres -t learning_history_2026_01 mathesis_hub > lh_2026_01.sql
```

#### Point-in-Time Recovery (PITR)
```bash
# WAL 아카이빙 활성화 (postgresql.conf)
wal_level = replica
archive_mode = on
archive_command = 'cp %p /var/lib/postgresql/archive/%f'
```

### 9.2 복구 전략

#### 전체 복구
```bash
# 데이터베이스 삭제 (주의!)
dropdb -U postgres mathesis_hub

# 데이터베이스 생성
createdb -U postgres mathesis_hub

# 백업 복구
pg_restore -U postgres -d mathesis_hub mathesis_hub_20260109.dump
```

#### 테이블별 복구
```bash
# 특정 테이블만 복구
pg_restore -U postgres -d mathesis_hub -t students mathesis_hub_20260109.dump
```

### 9.3 백업 자동화 (Cron)

```bash
# /etc/cron.d/postgres-backup
0 2 * * * postgres /usr/local/bin/backup_mathesis.sh

# /usr/local/bin/backup_mathesis.sh
#!/bin/bash
BACKUP_DIR="/var/backups/postgres"
DATE=$(date +%Y%m%d)

# 전체 백업
pg_dump -U postgres -Fc mathesis_hub > $BACKUP_DIR/mathesis_hub_$DATE.dump

# 7일 이상 된 백업 삭제
find $BACKUP_DIR -name "mathesis_hub_*.dump" -mtime +7 -delete
```

---

## 10. 성능 튜닝

### 10.1 PostgreSQL 설정 (postgresql.conf)

```ini
# 메모리 설정 (서버 메모리 16GB 기준)
shared_buffers = 4GB                # 총 메모리의 25%
effective_cache_size = 12GB         # 총 메모리의 75%
work_mem = 64MB                     # 정렬/해시 작업 메모리
maintenance_work_mem = 512MB        # VACUUM, CREATE INDEX 메모리

# 쿼리 플래너
random_page_cost = 1.1              # SSD 기준
effective_io_concurrency = 200      # SSD 동시성

# WAL 설정
wal_buffers = 16MB
min_wal_size = 1GB
max_wal_size = 4GB

# Checkpoint 설정
checkpoint_completion_target = 0.9
checkpoint_timeout = 15min

# 연결 풀
max_connections = 100
```

### 10.2 VACUUM 전략

```sql
-- Auto-vacuum 설정 (postgresql.conf)
autovacuum = on
autovacuum_naptime = 1min           -- 1분마다 실행
autovacuum_vacuum_threshold = 50    -- 최소 50행 변경 시
autovacuum_analyze_threshold = 50

-- 수동 VACUUM (야간 배치)
VACUUM ANALYZE students;
VACUUM ANALYZE interventions;

-- Full VACUUM (월 1회, 서비스 중단 시)
VACUUM FULL learning_history;
```

### 10.3 쿼리 최적화

#### EXPLAIN 분석
```sql
EXPLAIN ANALYZE
SELECT * FROM students
WHERE school_id = 'SCH001' AND grade = 10;

-- 결과 예시:
-- Index Scan using idx_students_school_id on students  (cost=0.15..8.17 rows=1 width=100)
--   Index Cond: ((school_id)::text = 'SCH001'::text)
--   Filter: (grade = 10)
```

#### Slow Query 로깅
```ini
# postgresql.conf
log_min_duration_statement = 1000   -- 1초 이상 쿼리 로깅
log_line_prefix = '%t [%p]: [%l-1] user=%u,db=%d '
```

### 10.4 Connection Pooling (PgBouncer)

```ini
# pgbouncer.ini
[databases]
mathesis_hub = host=localhost port=5432 dbname=mathesis_hub

[pgbouncer]
pool_mode = transaction
max_client_conn = 100
default_pool_size = 20
```

---

## 부록

### A. 데이터베이스 통계 조회

```sql
-- 테이블 크기
SELECT
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS size
FROM pg_tables
WHERE schemaname = 'public'
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;

-- 인덱스 크기
SELECT
    indexname,
    pg_size_pretty(pg_relation_size(schemaname||'.'||indexname)) AS size
FROM pg_indexes
WHERE schemaname = 'public'
ORDER BY pg_relation_size(schemaname||'.'||indexname) DESC;

-- 파티션별 행 수
SELECT
    tablename,
    n_live_tup AS row_count
FROM pg_stat_user_tables
WHERE tablename LIKE 'learning_history_%'
ORDER BY tablename;
```

### B. 데이터베이스 헬스 체크

```sql
-- 연결 수
SELECT count(*) FROM pg_stat_activity;

-- 대기 중인 쿼리
SELECT * FROM pg_stat_activity WHERE state = 'active' AND wait_event IS NOT NULL;

-- 데드락
SELECT * FROM pg_stat_database WHERE datname = 'mathesis_hub';

-- 캐시 히트율 (95% 이상 권장)
SELECT
    sum(heap_blks_hit) / nullif(sum(heap_blks_hit) + sum(heap_blks_read), 0) * 100 AS cache_hit_ratio
FROM pg_statio_user_tables;
```

### C. 마이그레이션 체크리스트

- [ ] Alembic 버전 확인
- [ ] 백업 생성
- [ ] 개발 환경에서 테스트
- [ ] Staging 환경에서 테스트
- [ ] 다운타임 공지
- [ ] Production 마이그레이션 실행
- [ ] 데이터 검증
- [ ] 롤백 계획 준비

---

**Document Version**: 1.0.0
**Last Updated**: 2026-01-09
**Review Date**: 2026-02-09
**Approved By**: [Pending]
