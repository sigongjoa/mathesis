# Node 0: Student Hub - API Specification

**Version**: 1.0.0
**Last Updated**: 2026-01-09
**Base URL**: `http://localhost:8000`
**Protocol**: HTTP/JSON, MCP (JSON-RPC 2.0)

---

## 📋 Table of Contents

1. [개요](#1-개요)
2. [MCP Server API](#2-mcp-server-api)
3. [REST API (Internal)](#3-rest-api-internal)
4. [인증](#4-인증)
5. [에러 처리](#5-에러-처리)
6. [Rate Limiting](#6-rate-limiting)
7. [OpenAPI Specification](#7-openapi-specification)

---

## 1. 개요

### 1.1 API 종류

Node 0 (Student Hub)는 두 가지 API를 제공:

| API 타입 | 프로토콜 | 용도 | 엔드포인트 |
|----------|----------|------|------------|
| **MCP Server** | JSON-RPC 2.0 | 외부 클라이언트 (LLM Orchestrator, Frontend) | `/mcp` |
| **REST API** | HTTP/JSON | 내부 관리 및 직접 호출 | `/api/v1/*` |

### 1.2 API 버전 관리

- **MCP API**: 버전 없음 (Tool 이름에 버전 포함 가능, 예: `get_unified_profile_v2`)
- **REST API**: URL에 버전 포함 (`/api/v1/...`)

### 1.3 Content-Type

- **Request**: `application/json`
- **Response**: `application/json`

---

## 2. MCP Server API

### 2.1 MCP Protocol Overview

**JSON-RPC 2.0 기반**

**공통 요청 형식**:
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "<tool_name>",
    "arguments": {
      // tool-specific arguments
    }
  }
}
```

**공통 응답 형식**:
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    // tool-specific result
  }
}
```

**에러 응답 형식**:
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "error": {
    "code": -32001,
    "message": "Error message",
    "data": {
      // additional error details
    }
  }
}
```

### 2.2 MCP Tool: get_unified_profile

**설명**: 학생의 통합 프로필 조회 (마스터 데이터 + Node 1-6 통합)

**엔드포인트**: `POST /mcp`

**요청**:
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

**Parameters**:
| 필드 | 타입 | 필수 | 기본값 | 설명 |
|------|------|------|--------|------|
| `student_id` | string (UUID) | ✓ | - | 학생 ID |
| `include_history` | boolean | ✗ | true | 학습 히스토리 포함 여부 |
| `days` | integer | ✗ | 30 | 히스토리 조회 기간 (일) |

**응답 (200 OK)**:
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
      "class_name": "1반",
      "student_number": "01",
      "email": "student@example.com",
      "parent_contact": "010-1234-5678"
    },
    "knowledge_state": {
      "node": 1,
      "source": "logic-engine",
      "concepts": [
        {
          "name": "이차방정식",
          "mastery": 0.85,
          "last_studied": "2026-01-08T14:30:00Z"
        },
        {
          "name": "함수",
          "mastery": 0.72,
          "last_studied": "2026-01-07T10:15:00Z"
        }
      ],
      "total_concepts": 15
    },
    "mastery_levels": {
      "node": 2,
      "source": "q-dna",
      "overall": 0.78,
      "by_subject": {
        "math": 0.82,
        "science": 0.74,
        "english": 0.75
      },
      "bkt_parameters": {
        "p_learned": 0.78,
        "p_transit": 0.12,
        "p_guess": 0.15,
        "p_slip": 0.08
      }
    },
    "recent_activities": [
      {
        "date": "2026-01-08",
        "type": "study",
        "source_node": 4,
        "duration": 3600,
        "problems_solved": 15,
        "correct_rate": 0.87
      },
      {
        "date": "2026-01-07",
        "type": "test",
        "source_node": 2,
        "test_id": "TEST_001",
        "score": 85,
        "total": 100
      }
    ],
    "latest_reports": [
      {
        "node": 5,
        "report_id": "REP_20260105_001",
        "type": "weekly",
        "generated_at": "2026-01-05T10:00:00Z",
        "url": "http://localhost:8005/reports/REP_20260105_001.pdf"
      }
    ],
    "cached": false,
    "generated_at": "2026-01-09T05:30:00Z"
  }
}
```

**에러**:
- `404`: Student not found
- `500`: Internal server error
- `503`: Node communication error

---

### 2.3 MCP Tool: create_learning_intervention

**설명**: 학습 개입 생성 (자동 약점 분석 + 학습 경로 생성)

**엔드포인트**: `POST /mcp`

**요청**:
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
      "duration_days": 14,
      "focus_areas": ["이차함수", "삼각함수"]
    }
  }
}
```

**Parameters**:
| 필드 | 타입 | 필수 | 기본값 | 설명 |
|------|------|------|--------|------|
| `student_id` | string (UUID) | ✓ | - | 학생 ID |
| `type` | enum | ✗ | "auto" | 개입 유형 ("auto", "teacher_requested") |
| `target_level` | float | ✓ | - | 목표 숙달도 (0.0 ~ 1.0) |
| `duration_days` | integer | ✗ | 14 | 개입 기간 (일) |
| `focus_areas` | array[string] | ✗ | null | 집중 영역 (선택, null이면 자동 분석) |

**응답 (201 Created)**:
```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "result": {
    "intervention_id": "INT_abc123def456",
    "student_id": "550e8400-e29b-41d4-a716-446655440000",
    "type": "auto",
    "weak_areas": [
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
    ],
    "learning_path": [
      {
        "step": 1,
        "activity": "review_concept",
        "concept": "이차함수 기본",
        "estimated_duration": 1800,
        "resources": [
          {
            "type": "video",
            "url": "http://example.com/video/quadratic-basics"
          }
        ]
      },
      {
        "step": 2,
        "activity": "practice",
        "concept": "이차함수",
        "problem_set_id": "PS_456",
        "num_problems": 10,
        "estimated_duration": 2400
      },
      {
        "step": 3,
        "activity": "assessment",
        "concept": "이차함수",
        "test_id": "TEST_789",
        "estimated_duration": 1200
      }
    ],
    "scheduled_tasks": [
      {
        "task_id": "TASK_101",
        "type": "progress_check",
        "scheduled_at": "2026-01-16T10:00:00Z"
      },
      {
        "task_id": "TASK_102",
        "type": "reminder",
        "scheduled_at": "2026-01-12T09:00:00Z"
      }
    ],
    "status": "active",
    "progress": {
      "completed": 0,
      "total": 3
    },
    "created_at": "2026-01-09T05:30:00Z"
  }
}
```

**에러**:
- `404`: Student not found
- `422`: Validation error (invalid parameters)
- `500`: Internal server error
- `503`: Node communication error

---

### 2.4 MCP Tool: schedule_periodic_task

**설명**: 주기적 작업 스케줄링 (일일/주간 리포트, 분석 등)

**엔드포인트**: `POST /mcp`

**요청**:
```json
{
  "jsonrpc": "2.0",
  "id": 3,
  "method": "tools/call",
  "params": {
    "name": "schedule_periodic_task",
    "arguments": {
      "student_id": "550e8400-e29b-41d4-a716-446655440000",
      "task_type": "daily_report",
      "schedule_type": "cron",
      "cron_expression": "0 9 * * *",
      "config": {
        "report_format": "pdf",
        "include_sections": ["progress", "recommendations"]
      }
    }
  }
}
```

**Parameters**:
| 필드 | 타입 | 필수 | 기본값 | 설명 |
|------|------|------|--------|------|
| `student_id` | string (UUID) | ✗ | null | 학생 ID (null이면 전역 작업) |
| `task_type` | enum | ✓ | - | 작업 유형 ("daily_report", "weekly_analytics", "intervention_check") |
| `schedule_type` | enum | ✓ | - | 스케줄 유형 ("cron", "interval", "one_time") |
| `cron_expression` | string | ✗ | null | Cron 표현식 (schedule_type="cron"일 때 필수) |
| `interval_seconds` | integer | ✗ | null | 실행 간격 (초, schedule_type="interval"일 때 필수) |
| `scheduled_at` | string (ISO 8601) | ✗ | null | 실행 시각 (schedule_type="one_time"일 때 필수) |
| `config` | object | ✗ | {} | 작업별 설정 |

**응답 (201 Created)**:
```json
{
  "jsonrpc": "2.0",
  "id": 3,
  "result": {
    "task_id": "TASK_daily_report_001",
    "student_id": "550e8400-e29b-41d4-a716-446655440000",
    "task_type": "daily_report",
    "schedule_type": "cron",
    "cron_expression": "0 9 * * *",
    "config": {
      "report_format": "pdf",
      "include_sections": ["progress", "recommendations"]
    },
    "status": "active",
    "next_run_at": "2026-01-10T09:00:00Z",
    "created_at": "2026-01-09T05:30:00Z"
  }
}
```

**에러**:
- `404`: Student not found (student_id 지정 시)
- `422`: Validation error (invalid cron expression)
- `500`: Internal server error

---

### 2.5 MCP Tool: get_class_analytics

**설명**: 학급/학년별 통합 분석 (여러 학생의 데이터 집계)

**엔드포인트**: `POST /mcp`

**요청**:
```json
{
  "jsonrpc": "2.0",
  "id": 4,
  "method": "tools/call",
  "params": {
    "name": "get_class_analytics",
    "arguments": {
      "school_id": "SCH001",
      "grade": 10,
      "class_name": "1반",
      "period": "week",
      "metrics": ["mastery_distribution", "activity_summary", "intervention_status"]
    }
  }
}
```

**Parameters**:
| 필드 | 타입 | 필수 | 기본값 | 설명 |
|------|------|------|--------|------|
| `school_id` | string | ✓ | - | 학교 ID |
| `grade` | integer | ✗ | null | 학년 (null이면 전체 학교) |
| `class_name` | string | ✗ | null | 반 이름 (null이면 전체 학년) |
| `period` | enum | ✗ | "week" | 분석 기간 ("day", "week", "month") |
| `metrics` | array[string] | ✗ | ["all"] | 조회할 메트릭 목록 |

**응답 (200 OK)**:
```json
{
  "jsonrpc": "2.0",
  "id": 4,
  "result": {
    "school_id": "SCH001",
    "grade": 10,
    "class_name": "1반",
    "period": "week",
    "date_range": {
      "from": "2026-01-03T00:00:00Z",
      "to": "2026-01-09T23:59:59Z"
    },
    "student_count": 25,
    "metrics": {
      "mastery_distribution": {
        "average": 0.75,
        "median": 0.78,
        "distribution": {
          "0.0-0.5": 2,
          "0.5-0.7": 8,
          "0.7-0.85": 10,
          "0.85-1.0": 5
        }
      },
      "activity_summary": {
        "total_study_hours": 320,
        "average_per_student": 12.8,
        "problems_solved": 1250,
        "average_accuracy": 0.82
      },
      "intervention_status": {
        "active": 5,
        "paused": 1,
        "completed": 3,
        "total": 9
      }
    },
    "top_performers": [
      {
        "student_id": "550e8400-e29b-41d4-a716-446655440000",
        "name": "김철수",
        "mastery": 0.92
      }
    ],
    "struggling_students": [
      {
        "student_id": "660e9500-f39c-51e5-b827-557766551111",
        "name": "이영희",
        "mastery": 0.45,
        "intervention_recommended": true
      }
    ],
    "generated_at": "2026-01-09T05:30:00Z"
  }
}
```

**에러**:
- `404`: School/Class not found
- `500`: Internal server error

---

## 3. REST API (Internal)

### 3.1 Students API

#### 3.1.1 학생 생성

**엔드포인트**: `POST /api/v1/students`

**요청**:
```json
{
  "name": "김철수",
  "school_id": "SCH001",
  "grade": 10,
  "class_name": "1반",
  "student_number": "01",
  "email": "student@example.com",
  "parent_contact": "010-1234-5678",
  "metadata": {
    "learning_style": "visual",
    "interests": ["math", "science"]
  }
}
```

**응답 (201 Created)**:
```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "name": "김철수",
  "school_id": "SCH001",
  "grade": 10,
  "class_name": "1반",
  "student_number": "01",
  "email": "student@example.com",
  "parent_contact": "010-1234-5678",
  "metadata": {
    "learning_style": "visual",
    "interests": ["math", "science"]
  },
  "created_at": "2026-01-09T05:30:00Z",
  "updated_at": "2026-01-09T05:30:00Z"
}
```

---

#### 3.1.2 학생 조회

**엔드포인트**: `GET /api/v1/students/{student_id}`

**응답 (200 OK)**:
```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "name": "김철수",
  "school_id": "SCH001",
  "grade": 10,
  "class_name": "1반",
  "student_number": "01",
  "email": "student@example.com",
  "parent_contact": "010-1234-5678",
  "metadata": {},
  "created_at": "2026-01-09T05:30:00Z",
  "updated_at": "2026-01-09T05:30:00Z"
}
```

**에러**:
- `404`: Student not found

---

#### 3.1.3 학생 목록 조회

**엔드포인트**: `GET /api/v1/students`

**Query Parameters**:
| 필드 | 타입 | 필수 | 기본값 | 설명 |
|------|------|------|--------|------|
| `school_id` | string | ✗ | null | 학교 ID 필터 |
| `grade` | integer | ✗ | null | 학년 필터 |
| `class_name` | string | ✗ | null | 반 이름 필터 |
| `skip` | integer | ✗ | 0 | 페이징 offset |
| `limit` | integer | ✗ | 100 | 페이징 limit (max: 1000) |

**요청 예시**:
```
GET /api/v1/students?school_id=SCH001&grade=10&class_name=1반&skip=0&limit=25
```

**응답 (200 OK)**:
```json
{
  "total": 25,
  "skip": 0,
  "limit": 25,
  "items": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "name": "김철수",
      "school_id": "SCH001",
      "grade": 10,
      "class_name": "1반",
      "student_number": "01",
      "created_at": "2026-01-09T05:30:00Z"
    },
    // ... more students
  ]
}
```

---

#### 3.1.4 학생 정보 업데이트

**엔드포인트**: `PUT /api/v1/students/{student_id}`

**요청**:
```json
{
  "email": "new_email@example.com",
  "parent_contact": "010-9876-5432",
  "metadata": {
    "learning_style": "kinesthetic"
  }
}
```

**응답 (200 OK)**:
```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "name": "김철수",
  "email": "new_email@example.com",
  "parent_contact": "010-9876-5432",
  "metadata": {
    "learning_style": "kinesthetic"
  },
  "updated_at": "2026-01-09T06:00:00Z"
}
```

---

#### 3.1.5 학생 삭제

**엔드포인트**: `DELETE /api/v1/students/{student_id}`

**응답 (204 No Content)**:
```
(empty body)
```

**에러**:
- `404`: Student not found

---

### 3.2 Interventions API

#### 3.2.1 개입 조회

**엔드포인트**: `GET /api/v1/interventions/{intervention_id}`

**응답 (200 OK)**:
```json
{
  "id": "INT_abc123def456",
  "student_id": "550e8400-e29b-41d4-a716-446655440000",
  "type": "auto",
  "weak_areas": [...],
  "learning_path": [...],
  "status": "active",
  "progress": {
    "completed": 1,
    "total": 3
  },
  "created_at": "2026-01-09T05:30:00Z",
  "completed_at": null
}
```

---

#### 3.2.2 개입 진행 상태 업데이트

**엔드포인트**: `PUT /api/v1/interventions/{intervention_id}/progress`

**요청**:
```json
{
  "completed": 2
}
```

**응답 (200 OK)**:
```json
{
  "id": "INT_abc123def456",
  "progress": {
    "completed": 2,
    "total": 3
  },
  "status": "active",
  "updated_at": "2026-01-09T06:00:00Z"
}
```

---

#### 3.2.3 개입 일시정지

**엔드포인트**: `POST /api/v1/interventions/{intervention_id}/pause`

**응답 (200 OK)**:
```json
{
  "id": "INT_abc123def456",
  "status": "paused",
  "updated_at": "2026-01-09T06:00:00Z"
}
```

---

#### 3.2.4 개입 재개

**엔드포인트**: `POST /api/v1/interventions/{intervention_id}/resume`

**응답 (200 OK)**:
```json
{
  "id": "INT_abc123def456",
  "status": "active",
  "updated_at": "2026-01-09T06:00:00Z"
}
```

---

#### 3.2.5 개입 취소

**엔드포인트**: `POST /api/v1/interventions/{intervention_id}/cancel`

**응답 (200 OK)**:
```json
{
  "id": "INT_abc123def456",
  "status": "cancelled",
  "completed_at": "2026-01-09T06:00:00Z"
}
```

---

### 3.3 Health Check API

#### 3.3.1 헬스 체크

**엔드포인트**: `GET /health`

**응답 (200 OK)**:
```json
{
  "status": "healthy",
  "checks": {
    "database": true,
    "redis": true,
    "mcp_clients": {
      "logic-engine": true,
      "q-dna": true,
      "gen-node": true,
      "lab-node": true,
      "report-node": true,
      "school-info": true
    }
  },
  "timestamp": "2026-01-09T05:30:00Z"
}
```

**응답 (503 Service Unavailable)**:
```json
{
  "status": "unhealthy",
  "checks": {
    "database": true,
    "redis": false,
    "mcp_clients": {
      "logic-engine": true,
      "q-dna": false,
      "gen-node": true,
      "lab-node": true,
      "report-node": true,
      "school-info": true
    }
  },
  "timestamp": "2026-01-09T05:30:00Z"
}
```

---

## 4. 인증

### 4.1 Phase 1: API Key 인증

**Header**:
```
X-API-Key: your-api-key-here
```

**예시**:
```bash
curl -X POST http://localhost:8000/mcp \
  -H "Content-Type: application/json" \
  -H "X-API-Key: sk_test_abc123def456" \
  -d '{...}'
```

**에러 (403 Forbidden)**:
```json
{
  "error": {
    "code": "INVALID_API_KEY",
    "message": "Invalid or missing API key"
  }
}
```

### 4.2 Phase 2: OAuth 2.0 + JWT (계획)

**향후 Keycloak 통합 예정**

---

## 5. 에러 처리

### 5.1 REST API 에러 형식

```json
{
  "error": {
    "code": "ERROR_CODE",
    "message": "Human-readable error message",
    "details": {
      "field": "student_id",
      "reason": "Invalid UUID format"
    }
  }
}
```

### 5.2 MCP API 에러 형식 (JSON-RPC 2.0)

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "error": {
    "code": -32001,
    "message": "Student not found",
    "data": {
      "student_id": "550e8400-e29b-41d4-a716-446655440000"
    }
  }
}
```

### 5.3 HTTP 상태 코드

| 코드 | 의미 | 사용 예시 |
|------|------|-----------|
| `200` | OK | 조회 성공 |
| `201` | Created | 생성 성공 |
| `204` | No Content | 삭제 성공 |
| `400` | Bad Request | 잘못된 요청 형식 |
| `401` | Unauthorized | 인증 실패 |
| `403` | Forbidden | 권한 없음 |
| `404` | Not Found | 리소스 없음 |
| `422` | Unprocessable Entity | 검증 실패 |
| `429` | Too Many Requests | Rate limit 초과 |
| `500` | Internal Server Error | 서버 오류 |
| `503` | Service Unavailable | 서비스 이용 불가 (Node 통신 실패 등) |

### 5.4 에러 코드

| 코드 | 설명 |
|------|------|
| `VALIDATION_ERROR` | 입력 검증 실패 |
| `NOT_FOUND` | 리소스 없음 |
| `INVALID_API_KEY` | API 키 오류 |
| `MCP_ERROR` | MCP 통신 오류 |
| `DATABASE_ERROR` | 데이터베이스 오류 |
| `CACHE_ERROR` | 캐시 오류 |
| `RATE_LIMIT_EXCEEDED` | Rate limit 초과 |

---

## 6. Rate Limiting

### 6.1 제한 정책

| API | 제한 | 기간 |
|-----|------|------|
| **MCP Tools** | 100 requests | 1 minute |
| **REST API (Read)** | 1000 requests | 1 minute |
| **REST API (Write)** | 100 requests | 1 minute |

### 6.2 Rate Limit Headers

**응답 헤더**:
```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 1704783600
```

**Rate Limit 초과 (429 Too Many Requests)**:
```json
{
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "Rate limit exceeded. Try again in 30 seconds.",
    "details": {
      "limit": 100,
      "reset_at": "2026-01-09T05:31:00Z"
    }
  }
}
```

---

## 7. OpenAPI Specification

### 7.1 OpenAPI 3.0 문서

**엔드포인트**: `GET /openapi.json`

**응답 (200 OK)**:
```json
{
  "openapi": "3.0.0",
  "info": {
    "title": "Node 0: Student Hub API",
    "version": "1.0.0",
    "description": "Student management and educational workflow orchestration"
  },
  "servers": [
    {
      "url": "http://localhost:8000",
      "description": "Development server"
    }
  ],
  "paths": {
    "/mcp": {
      "post": {
        "summary": "MCP Tool Call",
        "requestBody": {
          "content": {
            "application/json": {
              "schema": {
                "$ref": "#/components/schemas/MCPRequest"
              }
            }
          }
        },
        "responses": {
          "200": {
            "description": "Successful response",
            "content": {
              "application/json": {
                "schema": {
                  "$ref": "#/components/schemas/MCPResponse"
                }
              }
            }
          }
        }
      }
    },
    "/api/v1/students": {
      "get": {
        "summary": "List students",
        "parameters": [...],
        "responses": {...}
      },
      "post": {
        "summary": "Create student",
        "requestBody": {...},
        "responses": {...}
      }
    }
    // ... more endpoints
  },
  "components": {
    "schemas": {
      "MCPRequest": {...},
      "MCPResponse": {...},
      "Student": {...},
      "Intervention": {...}
      // ... more schemas
    },
    "securitySchemes": {
      "ApiKeyAuth": {
        "type": "apiKey",
        "in": "header",
        "name": "X-API-Key"
      }
    }
  },
  "security": [
    {
      "ApiKeyAuth": []
    }
  ]
}
```

### 7.2 Swagger UI

**엔드포인트**: `GET /docs`

- Interactive API documentation
- Try-it-out functionality
- Schema exploration

---

## 부록

### A. cURL 예시

#### MCP Tool 호출
```bash
curl -X POST http://localhost:8000/mcp \
  -H "Content-Type: application/json" \
  -H "X-API-Key: your-api-key" \
  -d '{
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
  }'
```

#### REST API 호출
```bash
# 학생 생성
curl -X POST http://localhost:8000/api/v1/students \
  -H "Content-Type: application/json" \
  -H "X-API-Key: your-api-key" \
  -d '{
    "name": "김철수",
    "school_id": "SCH001",
    "grade": 10,
    "class_name": "1반"
  }'

# 학생 조회
curl -X GET http://localhost:8000/api/v1/students/550e8400-e29b-41d4-a716-446655440000 \
  -H "X-API-Key: your-api-key"
```

### B. Python Client 예시

```python
import httpx

class StudentHubClient:
    def __init__(self, base_url: str, api_key: str):
        self.base_url = base_url
        self.client = httpx.AsyncClient(
            base_url=base_url,
            headers={"X-API-Key": api_key}
        )

    async def get_unified_profile(self, student_id: str):
        response = await self.client.post("/mcp", json={
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
        return response.json()["result"]

    async def create_student(self, student_data: dict):
        response = await self.client.post("/api/v1/students", json=student_data)
        return response.json()

# 사용 예시
client = StudentHubClient("http://localhost:8000", "your-api-key")
profile = await client.get_unified_profile("550e8400-e29b-41d4-a716-446655440000")
```

### C. JavaScript/TypeScript Client 예시

```typescript
class StudentHubClient {
  private baseUrl: string;
  private apiKey: string;

  constructor(baseUrl: string, apiKey: string) {
    this.baseUrl = baseUrl;
    this.apiKey = apiKey;
  }

  async getUnifiedProfile(studentId: string) {
    const response = await fetch(`${this.baseUrl}/mcp`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'X-API-Key': this.apiKey,
      },
      body: JSON.stringify({
        jsonrpc: '2.0',
        id: 1,
        method: 'tools/call',
        params: {
          name: 'get_unified_profile',
          arguments: {
            student_id: studentId,
            include_history: true,
            days: 30,
          },
        },
      }),
    });

    const data = await response.json();
    return data.result;
  }
}

// 사용 예시
const client = new StudentHubClient('http://localhost:8000', 'your-api-key');
const profile = await client.getUnifiedProfile('550e8400-e29b-41d4-a716-446655440000');
```

---

**Document Version**: 1.0.0
**Last Updated**: 2026-01-09
**Review Date**: 2026-02-09
**Approved By**: [Pending]
