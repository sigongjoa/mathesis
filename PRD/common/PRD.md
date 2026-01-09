# mathesis-common - Product Requirements Document (PRD)

**Version**: 1.0.0
**Last Updated**: 2026-01-09
**Status**: Stable (Production)
**Owner**: Mathesis Platform Team

---

## 📋 Table of Contents

1. [제품 개요](#1-제품-개요)
2. [비즈니스 목표](#2-비즈니스-목표)
3. [사용자 페르소나](#3-사용자-페르소나)
4. [기능 요구사항](#4-기능-요구사항)
5. [비기능 요구사항](#5-비기능-요구사항)
6. [기술 스택](#6-기술-스택)
7. [성공 지표](#7-성공-지표)
8. [로드맵](#8-로드맵)

---

## 1. 제품 개요

### 1.1 제품 비전

**mathesis-common**은 Mathesis 플랫폼의 모든 노드(Node 0-6)에서 공통으로 사용하는 **핵심 라이브러리**로, 일관된 개발 경험과 코드 재사용성을 제공합니다.

### 1.2 제품 목적

- **코드 재사용**: 중복 코드 제거 및 개발 생산성 향상
- **일관성**: 모든 노드에서 동일한 방식으로 LLM, DB, 크롤링 등을 사용
- **유지보수성**: 공통 기능의 중앙 집중식 관리
- **확장성**: 새로운 노드 추가 시 빠른 개발 가능

### 1.3 핵심 가치

- 🧩 **모듈화**: 독립적인 모듈로 필요한 기능만 선택적 사용
- 🔌 **플러그인 방식**: 쉬운 통합 및 설정
- 📚 **명확한 문서화**: 모든 모듈에 대한 상세 가이드 제공
- ⚡ **고성능**: 비동기 처리 및 최적화된 구현

---

## 2. 비즈니스 목표

### 2.1 정량적 목표

| 목표 | 현재 | 목표 (3개월) | 측정 방법 |
|------|------|--------------|-----------|
| **코드 재사용률** | 40% | 70% | 공통 라이브러리 사용 비율 |
| **개발 시간 단축** | - | 30% 감소 | 새 노드 개발 시간 비교 |
| **버그 감소** | - | 25% 감소 | 공통 기능 관련 버그 수 |
| **테스트 커버리지** | 60% | 85% | pytest-cov |

### 2.2 정성적 목표

- **개발자 경험 향상**: 통일된 API와 명확한 문서로 학습 곡선 감소
- **유지보수 효율성**: 공통 기능 수정 시 한 곳에서 모든 노드에 적용
- **품질 향상**: 중앙 집중식 테스트 및 검증으로 안정성 확보

---

## 3. 사용자 페르소나

### 3.1 백엔드 개발자 (김개발)

**배경**:
- 경력 3년차 Python 개발자
- FastAPI, SQLAlchemy 경험 있음
- 새로운 Node (예: Node 7) 개발 담당

**니즈**:
- LLM 연동을 위한 간단한 API
- 데이터베이스 연결 및 모델 정의 표준화
- 크롤링 및 데이터 처리 유틸리티

**Pain Points**:
- 각 노드마다 다른 방식으로 LLM을 사용해서 혼란스러움
- DB 연결 설정을 매번 다시 작성해야 함
- 문서 파싱(PDF, DOCX) 로직을 반복 구현

### 3.2 데이터 엔지니어 (이데이터)

**배경**:
- 경력 5년차 데이터 엔지니어
- 크롤링 및 ETL 파이프라인 구축 경험
- Node 2 (Q-DNA), Node 6 (School Info) 관리

**니즈**:
- 웹 크롤링 표준화 도구
- 데이터 파싱 및 변환 유틸리티
- 일관된 에러 처리 및 로깅

**Pain Points**:
- schoolinfo.go.kr 크롤링 로직이 여러 곳에 중복됨
- PDF 파싱 라이브러리를 각각 다르게 사용
- 크롤링 에러 처리가 일관되지 않음

### 3.3 AI 연구원 (박연구)

**배경**:
- 경력 2년차 AI/ML 연구원
- LLM, RAG, 프롬프트 엔지니어링 전문
- Node 1 (Logic Engine), Node 3 (Gen Node) 개발

**니즈**:
- Ollama, OpenAI 등 다양한 LLM 통합 인터페이스
- 프롬프트 템플릿 관리
- LLM 응답 파싱 및 검증

**Pain Points**:
- Ollama와 OpenAI의 API가 달라서 매번 다르게 구현
- 프롬프트 버전 관리가 어려움
- LLM 응답 형식이 일관되지 않음

---

## 4. 기능 요구사항

### 4.1 모듈 구조

```
mathesis-common/
├── llm/                  # LLM 통합
│   ├── base.py           # LLM 추상 클래스
│   ├── ollama.py         # Ollama 구현
│   ├── openai.py         # OpenAI 구현
│   └── prompts.py        # 프롬프트 템플릿
│
├── database/             # 데이터베이스
│   ├── base.py           # SQLAlchemy Base
│   ├── session.py        # DB 세션 관리
│   └── utils.py          # DB 유틸리티
│
├── crawlers/             # 크롤러
│   ├── base.py           # Crawler 추상 클래스
│   ├── schoolinfo.py     # 학교 정보 크롤러
│   └── utils.py          # 크롤링 유틸리티
│
├── export/               # 문서 내보내기
│   ├── pdf.py            # PDF 생성
│   ├── typst.py          # Typst 지원
│   └── utils.py          # Export 유틸리티
│
├── parsers/              # 문서 파싱
│   ├── pdf.py            # PDF 파싱
│   ├── docx.py           # DOCX 파싱
│   └── json.py           # JSON 파싱
│
└── utils/                # 공통 유틸리티
    ├── logger.py         # 구조화 로깅
    ├── validators.py     # 데이터 검증
    └── cache.py          # 캐싱 유틸리티
```

### 4.2 FR-1: LLM 통합 모듈

**우선순위**: P0 (Critical)

#### 기능 상세

**FR-1.1: LLM 추상 인터페이스**
- 모든 LLM 제공자를 통합하는 공통 인터페이스
- 메서드: `generate()`, `stream()`, `embed()`
- 비동기 처리 지원

**FR-1.2: Ollama 구현**
- Ollama API 연동
- 모델: llama3, qwen, nomic-embed-text
- 스트리밍 응답 지원

**FR-1.3: OpenAI 구현**
- OpenAI API 연동
- GPT-4, GPT-3.5, Embeddings
- 재시도 및 Rate Limiting 처리

**FR-1.4: 프롬프트 템플릿 관리**
- Jinja2 기반 템플릿 엔진
- 버전 관리 및 A/B 테스트 지원
- 프롬프트 변수 검증

**Acceptance Criteria**:
- [ ] LLM 제공자를 바꿔도 코드 수정 없이 동작
- [ ] 모든 LLM 호출은 5초 내 응답 또는 타임아웃
- [ ] 프롬프트 템플릿 100개 이상 관리 가능

### 4.3 FR-2: 데이터베이스 모듈

**우선순위**: P0 (Critical)

#### 기능 상세

**FR-2.1: SQLAlchemy Base 클래스**
- 공통 Base 클래스 제공
- Mixin 클래스 (TimestampMixin, SoftDeleteMixin)
- UUID 자동 생성

**FR-2.2: 비동기 DB 세션 관리**
- AsyncSession 관리
- Connection Pool 설정
- 트랜잭션 컨텍스트 매니저

**FR-2.3: Repository 패턴**
- BaseRepository 추상 클래스
- CRUD 기본 operations
- 페이징, 필터링, 정렬

**Acceptance Criteria**:
- [ ] 모든 노드에서 동일한 Base 클래스 사용
- [ ] DB 연결 설정은 환경 변수로 관리
- [ ] Repository 패턴으로 데이터 접근 일관성 확보

### 4.4 FR-3: 크롤러 모듈

**우선순위**: P1 (High)

#### 기능 상세

**FR-3.1: 학교 정보 크롤러**
- schoolinfo.go.kr 크롤링
- 학교 기본 정보, 평가 계획 추출
- PDF 다운로드 및 파싱

**FR-3.2: Crawler 추상 클래스**
- Rate Limiting
- 재시도 메커니즘
- User-Agent 관리

**FR-3.3: 크롤링 에러 처리**
- 네트워크 에러 처리
- 로깅 및 알림
- Graceful Degradation

**Acceptance Criteria**:
- [ ] 학교 정보 크롤링 성공률 > 95%
- [ ] 크롤링 속도: 100개 학교 < 5분
- [ ] 에러 발생 시 자동 재시도 (최대 3회)

### 4.5 FR-4: 문서 Export 모듈

**우선순위**: P1 (High)

#### 기능 상세

**FR-4.1: PDF 생성**
- ReportLab, WeasyPrint 지원
- 템플릿 기반 PDF 생성
- 한글 폰트 지원

**FR-4.2: Typst 지원**
- Typst 파일 생성 및 컴파일
- 수학 수식, 표, 그래프 지원

**FR-4.3: 문서 템플릿 관리**
- 리포트, 시험지, 분석 문서 템플릿
- 변수 치환 및 동적 생성

**Acceptance Criteria**:
- [ ] PDF 생성 속도: 10페이지 < 2초
- [ ] 한글 폰트 정상 렌더링
- [ ] Typst 컴파일 성공률 > 98%

### 4.6 FR-5: 문서 파싱 모듈

**우선순위**: P1 (High)

#### 기능 상세

**FR-5.1: PDF 파싱**
- pdfplumber, PyPDF2 통합
- 텍스트, 테이블, 이미지 추출
- OCR 지원 (Tesseract)

**FR-5.2: DOCX 파싱**
- python-docx 통합
- 텍스트, 표, 이미지 추출

**FR-5.3: JSON 파싱 및 검증**
- Pydantic 기반 검증
- JSON Schema 지원

**Acceptance Criteria**:
- [ ] PDF 파싱 정확도 > 90%
- [ ] 테이블 추출 성공률 > 85%
- [ ] OCR 지원 (한글, 영어)

### 4.7 FR-6: 공통 유틸리티

**우선순위**: P2 (Medium)

#### 기능 상세

**FR-6.1: 구조화 로깅**
- structlog 기반
- JSON 형식 로그
- 중앙 집중식 로깅 준비

**FR-6.2: 데이터 검증**
- Pydantic Validators
- 공통 검증 함수 (UUID, 이메일, 전화번호 등)

**FR-6.3: 캐싱 유틸리티**
- Redis 캐싱 래퍼
- TTL 관리
- Cache invalidation 패턴

**Acceptance Criteria**:
- [ ] 모든 노드에서 동일한 로그 형식 사용
- [ ] 캐시 히트율 > 70%
- [ ] 데이터 검증 에러율 < 1%

---

## 5. 비기능 요구사항

### 5.1 성능

| 요구사항 | 목표 | 측정 방법 |
|----------|------|-----------|
| **LLM 호출 응답 시간** | < 5초 (p95) | 프로메테우스 메트릭 |
| **DB 쿼리 응답 시간** | < 100ms (p95) | SQLAlchemy 로깅 |
| **PDF 생성 시간** | < 2초 (10페이지) | 벤치마크 테스트 |
| **크롤링 속도** | 100개 학교 < 5분 | 크롤링 로그 |

### 5.2 안정성

- **테스트 커버리지**: > 85%
- **CI/CD**: 모든 PR은 테스트 통과 필수
- **버전 관리**: Semantic Versioning (MAJOR.MINOR.PATCH)

### 5.3 보안

- **API Key 관리**: 환경 변수로 관리, 코드에 하드코딩 금지
- **데이터 암호화**: 민감 데이터 암호화 유틸리티 제공
- **의존성 검사**: Snyk, Dependabot으로 취약점 검사

### 5.4 확장성

- **모듈화**: 각 모듈은 독립적으로 사용 가능
- **플러그인 지원**: 새로운 LLM 제공자, 크롤러 추가 용이
- **비동기 처리**: asyncio 기반 비동기 I/O

### 5.5 유지보수성

- **문서화**: 모든 public API는 docstring 필수
- **타입 힌트**: Type Hints 100% 적용
- **코드 품질**: Black, isort, mypy, flake8 통과

---

## 6. 기술 스택

### 6.1 Core

| 기술 | 버전 | 용도 |
|------|------|------|
| **Python** | 3.11+ | 언어 |
| **Pydantic** | 2.5+ | 데이터 검증 |
| **asyncio** | - | 비동기 처리 |

### 6.2 LLM

| 기술 | 버전 | 용도 |
|------|------|------|
| **httpx** | 0.25+ | HTTP 클라이언트 (Ollama, OpenAI) |
| **tiktoken** | 0.5+ | OpenAI 토큰 카운팅 |

### 6.3 Database

| 기술 | 버전 | 용도 |
|------|------|------|
| **SQLAlchemy** | 2.0+ | ORM |
| **asyncpg** | 0.29+ | PostgreSQL 비동기 드라이버 |

### 6.4 Crawling

| 기술 | 버전 | 용도 |
|------|------|------|
| **httpx** | 0.25+ | HTTP 클라이언트 |
| **BeautifulSoup4** | 4.12+ | HTML 파싱 |
| **lxml** | 4.9+ | XML/HTML 파서 |

### 6.5 Document Processing

| 기술 | 버전 | 용도 |
|------|------|------|
| **pdfplumber** | 0.10+ | PDF 파싱 |
| **PyPDF2** | 3.0+ | PDF 조작 |
| **python-docx** | 0.8+ | DOCX 파싱 |
| **reportlab** | 4.0+ | PDF 생성 |
| **WeasyPrint** | 60+ | HTML → PDF |
| **Tesseract** | 5.0+ | OCR |

### 6.6 Utilities

| 기술 | 버전 | 용도 |
|------|------|------|
| **structlog** | 23.2+ | 구조화 로깅 |
| **redis** | 5.0+ | 캐싱 |
| **Jinja2** | 3.1+ | 템플릿 엔진 |

---

## 7. 성공 지표

### 7.1 KPI (Key Performance Indicators)

| KPI | 현재 | 목표 (3개월) | 측정 주기 |
|-----|------|--------------|-----------|
| **코드 재사용률** | 40% | 70% | 월간 |
| **테스트 커버리지** | 60% | 85% | 매 릴리스 |
| **버그 발생률** | - | < 5 bugs/month | 월간 |
| **개발 시간 단축** | - | 30% 감소 | 프로젝트별 |
| **문서화 완성도** | 50% | 90% | 분기별 |

### 7.2 사용 메트릭

| 메트릭 | 목표 | 측정 방법 |
|--------|------|-----------|
| **일일 활성 사용자 (개발자)** | 5명 | Git commits |
| **모듈 사용 빈도** | 100회/일 | Import 로깅 |
| **GitHub Stars** | 20+ | GitHub |
| **이슈 응답 시간** | < 24시간 | GitHub Issues |

---

## 8. 로드맵

### 8.1 Phase 1: 핵심 모듈 (✅ 완료)

**기간**: 2025년 Q4
**목표**: 기본 LLM, DB, 크롤링 모듈 제공

- [x] LLM 모듈 (Ollama, OpenAI)
- [x] Database 모듈 (SQLAlchemy)
- [x] Crawler 모듈 (schoolinfo.go.kr)
- [x] Export 모듈 (PDF, Typst)

### 8.2 Phase 2: 확장 및 최적화 (🚧 진행 중)

**기간**: 2026년 Q1
**목표**: 문서 파싱, 유틸리티 추가

- [x] PDF 파싱 모듈
- [x] DOCX 파싱 모듈
- [ ] 구조화 로깅
- [ ] 캐싱 유틸리티
- [ ] 프롬프트 템플릿 관리

**Success Criteria**:
- 테스트 커버리지 85% 달성
- 모든 노드에서 mathesis-common 사용

### 8.3 Phase 3: 고급 기능 (📋 계획)

**기간**: 2026년 Q2
**목표**: AI 기능 강화, 성능 최적화

- [ ] LLM 응답 캐싱
- [ ] Vector DB 통합 (ChromaDB, Qdrant)
- [ ] 멀티모달 LLM 지원 (이미지, 음성)
- [ ] GraphRAG 유틸리티
- [ ] 크롤링 병렬화 및 분산 처리

**Success Criteria**:
- LLM 호출 비용 30% 절감
- 크롤링 속도 2배 향상

### 8.4 Phase 4: 프로덕션 강화 (📋 계획)

**기간**: 2026년 Q3
**목표**: 안정성 및 모니터링 강화

- [ ] 중앙 집중식 로깅 (ELK Stack)
- [ ] 분산 추적 (OpenTelemetry)
- [ ] 성능 모니터링 (Prometheus + Grafana)
- [ ] 자동화된 테스트 및 배포

**Success Criteria**:
- 버그 발생률 < 5 bugs/month
- 모든 모듈 99.5% 가용성

---

## 부록

### A. 용어 정의

| 용어 | 정의 |
|------|------|
| **LLM** | Large Language Model (대규모 언어 모델) |
| **RAG** | Retrieval-Augmented Generation |
| **OCR** | Optical Character Recognition (광학 문자 인식) |
| **ORM** | Object-Relational Mapping |
| **Typst** | 차세대 마크업 기반 조판 시스템 |

### B. 참고 문서

- [Architecture Overview](../../architecture/01_MSA_ARCHITECTURE.md)
- [Node 0 PRD](../node0/PRD.md)
- [mathesis-common GitHub](https://github.com/mathesis/mathesis-common)

### C. 연락처

- **Product Owner**: mathesis-team@example.com
- **Tech Lead**: tech-lead@example.com
- **GitHub Issues**: https://github.com/mathesis/mathesis-common/issues

---

**Document Version**: 1.0.0
**Last Updated**: 2026-01-09
**Review Date**: 2026-02-09
**Approved By**: [Pending]
