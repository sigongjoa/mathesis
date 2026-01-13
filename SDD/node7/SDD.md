# Node 7: Error Note System - System Design Document (SDD)

**Version**: 1.0.0
**Last Updated**: 2026-01-09
**Status**: Design Phase

---

## 1. 시스템 아키텍처

Node 7은 MSA 환경에서 FastAPI 기반의 독립적인 서비스로 동작하며, MCP를 통해 다른 노드와 통신합니다.

### 1.1 구성 요소
- **FastAPI Server**: REST API 제공 및 비즈니스 로직 처리.
- **Anki Engine**: SM-2 알고리즘 구현체.
- **LLM Agent**: 오답 분석 및 변형 문제 생성 (Ollama 활용).
- **Celery Worker**: 주기적인 복습 알림 처리.

---

## 2. 데이터베이스 설계 (PostgreSQL)

### 2.1 테이블 정의
- `error_notes`: 오답노트 메인 데이터 (JSONB 필드 포함).
- `review_history`: 복습 이력 및 Anki 파라미터 변화 기록.
- `teacher_notifications`: 알림 발송 대기열.

---

## 3. 알고리즘 상세 (Anki SM-2)

- **Ease Factor (EF)**: 초기값 2.5.
- **Interval**: 1 -> 6 -> 14 -> (I * EF).
- **Quality (0-5)**: 사용자 입력에 따른 EF 및 Interval 조정.

---

## 4. 인터페이스 설계 (MCP)

### 4.1 제공 도구 (Tools)
- `create_error_note`: 오답노트 등록.
- `submit_review`: 복습 결과 반영.
- `get_due_reviews`: 오늘 복습 목록 조회.
- `analyze_error_patterns`: 통계 분석.

---

## 5. 보안 및 성능
- 데이터 접근 권한 (담당 선생님만 접근 가능).
- LLM 호출 비동기 처리 및 타임아웃 설정.
