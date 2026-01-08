# Mathesis MSA Architecture

> Microservices Architecture for Educational Intelligence Platform

**Last Updated**: 2026-01-08
**Version**: 1.0
**Status**: Production (Phase 2 진행 중)

---

## 1. Architecture Principles

### 1.1 Domain-Driven Design (DDD)

각 마이크로서비스는 **명확한 비즈니스 도메인**을 담당합니다:

| Service | Bounded Context | Core Domain |
|---------|-----------------|-------------|
| **Logic Engine** | 교육 이론 관리 | 지식 그래프, 개념 관계 |
| **Q-DNA** | 문제 생명주기 | 문제 은행, 학습 추적 |
| **Q-Metrics** | 평가 데이터 분석 | 시험 분석, 교육공학 |
| **School Info** | 외부 데이터 통합 | 크롤링, RAG |

### 1.2 Service Independence

#### 배포 독립성
- 각 서비스는 **독립적으로 배포** 가능
- 한 서비스의 장애가 다른 서비스에 영향 최소화
- 독립적인 스케일링 (수평 확장)

#### 데이터 독립성 (Polyglot Persistence)
```
Logic Engine   → Neo4j (그래프) + PostgreSQL (메타데이터)
Q-DNA          → PostgreSQL (관계형)
Q-Metrics      → Neo4j (그래프) + Redis (캐시)
School Info    → ChromaDB (벡터) + 파일시스템
```

#### 기술 스택 자유도
- Python: 모든 서비스 (공통 생태계)
- FastAPI: 표준 웹 프레임워크
- LLM: Ollama (로컬 추론)

### 1.3 Shared Libraries

**mathesis-common** 패키지:
- 공통 LLM 클라이언트 (`OllamaClient`)
- 데이터베이스 어댑터 (`ChromaHybridStore`, `HierarchicalChromaStore`)
- 크롤러 베이스 (`BaseCrawler`, `SchoolInfoCrawler`)
- 문서 처리 (`PDFGenerator`, `TypstWrapper`)

**버전 관리**:
- Semantic Versioning (SemVer)
- 하위 호환성 유지
- Breaking Change는 Major 버전 업

---

## 2. Service Catalog

### 2.1 Node 1: Logic Engine

**도메인**: 교육 이론 및 개념 관리

#### 책임 (Responsibilities)
- 학술 논문 파싱 (GROBID)
- 교육 개념 추출 (LLM)
- 지식 그래프 구축 (Neo4j)
- GraphRAG 질의

#### 기술 스택
- **Language**: Python 3.11+
- **Framework**: FastAPI
- **Database**: Neo4j 5.x (Primary), PostgreSQL 14 (Metadata)
- **LLM**: Ollama (Llama 3.1)
- **Parser**: GROBID

#### API Endpoints
```
POST /api/v1/papers/upload          - 논문 업로드
GET  /api/v1/concepts/{concept_id}  - 개념 조회
POST /api/v1/graph/query            - GraphRAG 질의
```

#### Port
`8001`

#### Dependencies
- Neo4j: `bolt://localhost:7687`
- PostgreSQL: `postgresql://localhost:5432/logic_engine`
- Ollama: `http://localhost:11434`

---

### 2.2 Node 2: Q-DNA

**도메인**: 지능형 문제 은행 및 학습 추적

#### 책임
- 문제 이미지 OCR (Tesseract + Ollama Vision)
- 자동 태깅 (LLM)
- 학습 추적 (BKT - Bayesian Knowledge Tracing)
- 문제 추천 (IRT - Item Response Theory)
- 계층형 교육과정 관리 (PostgreSQL ltree)

#### 기술 스택
- **Language**: Python 3.11+ / TypeScript
- **Framework**: FastAPI (Backend), React 19 (Frontend)
- **Database**: PostgreSQL 14+ (ltree, JSONB)
- **LLM**: Ollama (llama3.2-vision, llama3.1)
- **OCR**: Tesseract

#### API Endpoints
```
POST /api/v1/questions/upload       - 문제 업로드
GET  /api/v1/questions/recommend    - 문제 추천
POST /api/v1/attempts/submit        - 답안 제출
GET  /api/v1/students/{id}/mastery  - 숙련도 조회
```

#### Port
`8002`

#### Dependencies
- PostgreSQL: `postgresql://localhost:5432/q_dna`
- Ollama: `http://localhost:11434`

---

### 2.3 Node 5: Q-Metrics (Synapse-K)

**도메인**: 시험지 분석 및 교육공학 평가

#### 책임
- 시험지 OCR 및 구조화
- 시맨틱 분석 (문제 유형, 난이도 파악)
- 교육공학 프레임워크 적용 (Bloom's Taxonomy, Webb's DOK)
- 분석 리포트 생성

#### 기술 스택
- **Language**: Python 3.11+
- **Framework**: FastAPI
- **Database**: Neo4j 5.x (지식 그래프), Redis (캐시)
- **LLM**: Ollama

#### API Endpoints
```
POST /api/v1/exams/upload           - 시험지 업로드
GET  /api/v1/exams/{id}/analysis    - 분석 결과
POST /api/v1/exams/{id}/report      - 리포트 생성
```

#### Port
`8005`

#### Dependencies
- Neo4j: `bolt://localhost:7687`
- Redis: `redis://localhost:6379`
- Ollama: `http://localhost:11434`

---

### 2.4 Node 6: School Info

**도메인**: 학교 정보 수집 및 RAG

#### 책임
- schoolinfo.go.kr 크롤링
- PDF → Enhanced JSON 변환
- Hierarchical RAG (Parent-Child Retrieval)
- Korean BM25 + Vector Hybrid Search
- 고품질 PDF 생성 (Typst)

#### 기술 스택
- **Language**: Python 3.11+
- **Framework**: FastAPI
- **Database**: ChromaDB (벡터 DB)
- **LLM**: Ollama (llama3, nomic-embed-text)
- **Parser**: pdfplumber
- **PDF Generator**: Typst

#### API Endpoints
```
POST /schools/{code}/teaching-plans  - 크롤링 + PDF 생성
POST /rag/ingest                     - PDF → RAG 색인
POST /rag/query                      - RAG 질의
GET  /rag/export/{doc_id}            - Enhanced JSON 다운로드
```

#### Port
`8006`

#### Dependencies
- ChromaDB: 로컬 파일시스템
- Ollama: `http://localhost:11434`
- Typst: 시스템 PATH

---

### 2.5 mathesis-common

**도메인**: 공통 라이브러리

#### 제공 기능
- **LLM Clients**: `OllamaClient`, `AnthropicClient` (계획)
- **Database**: `ChromaHybridStore`, `HierarchicalChromaStore`
- **Crawlers**: `BaseCrawler`, `SchoolInfoCrawler`
- **Export**: `PDFGenerator`, `TypstWrapper`, `Visualizers`
- **gRPC**: 서비스 간 통신 (계획)

#### 설치
```bash
pip install -e ../mathesis-common
```

---

## 3. Communication Patterns

### 3.1 Synchronous (REST API)

**현재 방식** (Phase 1):
- HTTP/REST로 서비스 간 직접 호출
- 예: `Q-DNA` → `Logic Engine` (개념 정보 조회)

```python
# Q-DNA에서 Logic Engine 호출
import httpx

response = httpx.get("http://localhost:8001/api/v1/concepts/123")
concept = response.json()
```

**장점**: 구현 단순, 디버깅 용이
**단점**: Tight Coupling, 장애 전파 가능

### 3.2 Asynchronous (Event-Driven)

**계획 중** (Phase 2):
- 이벤트 버스 도입 (RabbitMQ / Kafka)
- 이벤트 예시:
  - `QuestionCreated`: 새 문제 등록 → 자동 태깅 트리거
  - `ExamAnalyzed`: 분석 완료 → 리포트 생성 트리거

```
[Q-DNA] --{QuestionCreated}--> [Event Bus]
                                     ↓
                            [Logic Engine] (개념 자동 연결)
```

**장점**: Loose Coupling, 확장성 높음
**단점**: 복잡도 증가, 디버깅 어려움

### 3.3 gRPC (High-Performance)

**계획 중** (Phase 2):
- 서비스 간 고성능 통신 필요 시
- 예: 대량 데이터 동기화

---

## 4. Data Management

### 4.1 Database Per Service

**원칙**: 각 서비스는 **자신의 데이터베이스만 접근**

```
❌ 금지:  Q-DNA → Logic Engine DB 직접 접근
✅ 권장:  Q-DNA → Logic Engine API 호출
```

### 4.2 Data Consistency

#### Eventual Consistency
- 이벤트 기반 동기화
- 예: Logic Engine에서 개념 수정 → `ConceptUpdated` 이벤트 → Q-DNA가 캐시 갱신

#### SAGA Pattern (계획)
- 분산 트랜잭션 처리
- 예: 문제 등록 + 태깅 + 개념 연결 → 하나라도 실패 시 보상 트랜잭션

### 4.3 Shared Master Data

**중앙 관리** (Phase 3 계획):
- 학생 정보
- 학교 마스터 데이터
- 교육과정 코드

각 서비스는 **읽기 전용 복제본** 유지 (CQRS)

---

## 5. Service Discovery & Load Balancing

### 5.1 Development (현재)
- 하드코딩된 포트 (`localhost:8001`, ...)
- Docker Compose로 관리

### 5.2 Production (계획)
- **Service Discovery**: Consul / Eureka
- **Load Balancer**: Nginx / Traefik
- **API Gateway**: Kong / KrakenD

```
[Client] → [API Gateway] → [Service Discovery]
                              ↓
                    [Node1] [Node2] [Node5] [Node6]
```

---

## 6. Monitoring & Observability

### 6.1 Logging (Phase 3)
- **중앙 로깅**: ELK Stack (Elasticsearch + Logstash + Kibana)
- **구조화 로그**: JSON 형식
- **Correlation ID**: 요청 추적

### 6.2 Metrics (Phase 3)
- **Prometheus**: 메트릭 수집
- **Grafana**: 대시보드

모니터링 항목:
- API 응답 시간
- 에러율
- 데이터베이스 커넥션 풀
- LLM 추론 시간

### 6.3 Tracing (Phase 3)
- **Jaeger / Zipkin**: 분산 추적
- 서비스 간 호출 흐름 시각화

---

## 7. Security

### 7.1 Authentication & Authorization (계획)
- **Keycloak**: 중앙 인증 서버
- **OAuth 2.0 / OIDC**: 표준 프로토콜
- **JWT**: 서비스 간 토큰 전달

### 7.2 API Security
- **API Gateway**: Rate Limiting, IP Whitelist
- **HTTPS**: TLS 1.3
- **API Key**: 외부 서비스 접근 제어

---

## 8. Deployment

### 8.1 Development
```bash
docker-compose up -d
```

### 8.2 Staging (Phase 3)
- Docker Swarm / Kubernetes (Minikube)
- Blue-Green Deployment

### 8.3 Production (Phase 3)
- **Kubernetes**: Orchestration
- **Helm**: 패키징
- **ArgoCD**: GitOps CD
- **Istio**: Service Mesh

---

## 9. Scalability

### 9.1 Horizontal Scaling
각 서비스는 독립적으로 스케일 아웃:
```
Q-DNA: 2 replicas (트래픽 많음)
Logic Engine: 1 replica (CPU 집약적)
School Info: 1 replica (I/O 집약적)
```

### 9.2 Database Scaling
- **Read Replica**: PostgreSQL, Neo4j
- **Sharding**: PostgreSQL (학생 수 증가 시)
- **Caching**: Redis (자주 조회되는 데이터)

---

## 10. Migration Strategy

### Phase 1 → Phase 2
- [ ] API Gateway 도입 (Kong)
- [ ] 이벤트 버스 추가 (RabbitMQ)
- [ ] 중앙 인증 (Keycloak)

### Phase 2 → Phase 3
- [ ] Kubernetes 마이그레이션
- [ ] Service Mesh (Istio)
- [ ] 중앙 로깅/모니터링

---

## 11. Best Practices

### 11.1 API Design
- RESTful 원칙 준수
- Versioning (`/api/v1/`)
- OpenAPI 명세 작성

### 11.2 Error Handling
- 표준 HTTP 상태 코드
- 구조화된 에러 응답:
```json
{
  "error": {
    "code": "RESOURCE_NOT_FOUND",
    "message": "Question not found",
    "details": {"question_id": 123}
  }
}
```

### 11.3 Testing
- **Unit Tests**: 각 서비스 내부
- **Integration Tests**: API 계약 테스트
- **E2E Tests**: 전체 시나리오

---

## 12. References

- [C4 Model](https://c4model.com/) - 아키텍처 문서화
- [Domain-Driven Design](https://www.domainlanguage.com/ddd/) - DDD 원칙
- [12-Factor App](https://12factor.net/) - 클라우드 네이티브 앱
- [Microservices Patterns](https://microservices.io/patterns/) - MSA 패턴

---

**Next**: [Service Map](./02_SERVICE_MAP.md) - 서비스 간 관계 상세
