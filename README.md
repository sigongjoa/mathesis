# Mathesis Platform

> AI 기반 교육 인텔리전스 플랫폼 - MSA 아키텍처

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Python 3.11+](https://img.shields.io/badge/python-3.11+-blue.svg)](https://www.python.org/downloads/)
[![Microservices](https://img.shields.io/badge/architecture-microservices-green.svg)](./architecture/01_MSA_ARCHITECTURE.md)

---

## 🎯 Vision

교육 빅데이터와 AI를 결합하여 **개인화된 학습 경험**을 제공하는 통합 교육 플랫폼

### 핵심 가치
- 📚 **지식 그래프**: 교육 이론을 실행 가능한 로직으로 변환
- 🎯 **적응형 학습**: 학생 수준에 맞춘 문제 추천 (BKT/IRT)
- 📊 **심층 분석**: 시험지 데이터를 교육공학적 관점에서 분석
- 🏫 **학교 인텔리전스**: 학교별 맞춤 교육 정보 제공

---

## 🏗️ MSA Architecture

**Node 0 (마스터 노드) + 6개 도메인 서비스 + 공통 라이브러리**로 구성

```
┌──────────────────────────────────────────────────────────────┐
│                    Mathesis Platform                          │
├──────────────────────────────────────────────────────────────┤
│                     Node 0: Student Hub                       │
│              학생 통합 관리 & 교육 워크플로우 오케스트레이션    │
│          (Single Source of Truth + Workflow Orchestrator)    │
├──────────────────────────────────────────────────────────────┤
│  Node 1    Node 2    Node 3    Node 4    Node 5    Node 6   Node 7 │
│  Logic     Q-DNA     Gen       Lab       Report    School   Error  │
│  Engine    문제은행  문제생성   학습추적   리포트    정보     Note   │
│  교육이론   BKT/IRT   AI생성    히트맵    Typst     RAG      Anki   │
├──────────────────────────────────────────────────────────────┤
│                mathesis-common (공통 라이브러리)              │
│           LLM • Database • Crawlers • Export                 │
└──────────────────────────────────────────────────────────────┘
```

**상세 문서**: [MSA 아키텍처](./architecture/01_MSA_ARCHITECTURE.md)

---

## 📦 Services Overview

| Service | Port | Domain | Tech Stack | Status |
|---------|------|--------|------------|--------|
| **[Node 0: Student Hub](./nodes/NODE0_STUDENT_HUB.md)** 🌟 | 8000 | 학생 통합 관리 & 워크플로우 오케스트레이션 | FastAPI, PostgreSQL, Redis, Celery | 🚧 Design |
| **[Node 1: Logic Engine](./nodes/NODE1_LOGIC_ENGINE.md)** | 8001 | 교육 이론 지식 그래프 | Python, Neo4j, Ollama | ✅ Production |
| **[Node 2: Q-DNA](./nodes/NODE2_Q_DNA.md)** ⭐ | 8002 | 지능형 문제 은행 | FastAPI, PostgreSQL, mathesis_core | ✅ Refactored |
| **[Node 3: Gen Node](./nodes/NODE3_GEN_NODE.md)** | 8003 | AI 기반 문제 생성 | FastAPI, Ollama | 🚧 Design |
| **[Node 4: Lab Node](./nodes/NODE4_LAB_NODE.md)** | 8004 | 학습 활동 추적 & 히트맵 | FastAPI, PostgreSQL, Redis | 🚧 Design |
| **[Node 5: Report Node](./nodes/NODE5_REPORT_NODE.md)** | 8005 | 진단 리포트 생성 | FastAPI, Typst, Plotly | 🚧 Design |
| **[Node 6: School Info](./nodes/NODE6_SCHOOL_INFO.md)** | 8006 | 학교 정보 RAG | FastAPI, ChromaDB, Typst | ✅ Production |
| **[Node 7: Error Note](./nodes/NODE7_ERROR_NOTE.md)** ⭐ | 8007 | 메타인지 오답노트 & Anki | FastAPI, PostgreSQL, mathesis_core | ✅ Refactored |
| **[mathesis-common](../mathesis-common/)** ⭐ | - | 공통 모듈 라이브러리 | Vision, Analysis, Generation | ✅ Core Complete |

⭐ **Recent Updates**:
- **mathesis-common**: 핵심 3대 모듈 완성 (Vision, Analysis, Generation)
- **Node 2**: mathesis_core 모듈 활용으로 리팩토링 완료
- **Node 7**: mathesis_core 모듈 통합, 기능 강화

🌟 **Node 0 (Student Hub)**: 마스터 노드로서 모든 학생 데이터의 단일 진실 공급원이자, 교육 워크플로우 자동화를 담당

---

## 🚀 Quick Start

### Prerequisites
```bash
# Docker & Docker Compose
docker --version  # 24.0+

# Ollama (로컬 LLM)
curl -fsSL https://ollama.com/install.sh | sh
ollama pull llama3:latest
ollama pull nomic-embed-text
```

### Launch All Services (30분)
```bash
# 1. Clone repository
git clone <repository-url>
cd mathesis

# 2. Environment setup
cp .env.example .env

# 3. Start all services
docker-compose up -d

# 4. Verify
docker-compose ps
```

### Access Services
- **Student Hub** (마스터 노드): http://localhost:8000/docs
- **Logic Engine**: http://localhost:8001/docs
- **Q-DNA**: http://localhost:8002/docs
- **Gen Node**: http://localhost:8003/docs
- **Lab Node**: http://localhost:8004/docs
- **Report Node**: http://localhost:8005/docs
- **School Info**: http://localhost:8006/docs
- **Error Note**: http://localhost:8007/docs

**상세 가이드**: [Quick Start Guide](./guides/QUICKSTART.md)

---

## 📚 Documentation

### Architecture
- [00. MCP System Design](./architecture/00_MCP_SYSTEM_DESIGN.md) - MCP 시스템 설계
- [01. MSA Architecture](./architecture/01_MSA_ARCHITECTURE.md) - 마이크로서비스 아키텍처
- [02. Modular Architecture](./architecture/02_MODULAR_ARCHITECTURE.md) ⭐ - 모듈 기반 아키텍처
- [03. Module Specifications](./architecture/03_MODULE_SPECIFICATIONS.md) ⭐ - 모듈 API 명세
- [04. Modularization Recommendations](./architecture/04_MODULARIZATION_RECOMMENDATIONS.md) ⭐ - 모듈화 분석

### Diagrams
- [System Context](./diagrams/system_context.puml) - C4 Level 1
- [Container Diagram](./diagrams/container_diagram.puml) - C4 Level 2
- [Service Interactions](./diagrams/service_interactions.puml) - 시퀀스 다이어그램
- [Deployment](./diagrams/deployment.puml) - 배포 구조

### API
- [API Gateway](./api/00_API_GATEWAY.md) - API 게이트웨이 (계획)
- [Service Contracts](./api/01_SERVICE_CONTRACTS.md) - 서비스 간 계약
- [OpenAPI Specs](./api/openapi/) - Swagger/OpenAPI 명세

### Guides
- [Quick Start](./guides/QUICKSTART.md) - 빠른 시작
- [Modular Migration Guide](./guides/MODULAR_MIGRATION_GUIDE.md) ⭐ - 모듈화 마이그레이션 가이드
- [Deployment](./guides/DEPLOYMENT.md) - 배포 가이드 (예정)
- [Monitoring](./guides/MONITORING.md) - 모니터링 (예정)
- [Troubleshooting](./guides/TROUBLESHOOTING.md) - 문제 해결 (예정)

### ADR (Architecture Decision Records)
- [001. MSA Adoption](./adr/001-msa-adoption.md) - MSA 도입 결정
- [002. Common Library Strategy](./adr/002-common-library-strategy.md) - 공통 라이브러리 전략
- [003. Data Consistency](./adr/003-data-consistency.md) - 데이터 일관성

---

## 🔧 Technology Stack

### Backend
- **Python 3.11+** - 모든 서비스
- **FastAPI** - REST API 프레임워크
- **Ollama** - 로컬 LLM (Llama 3, Qwen)

### Databases
- **PostgreSQL 14+** - Q-DNA, Logic Engine
- **Neo4j 5.x** - Logic Engine, Q-Metrics (지식 그래프)
- **ChromaDB** - School Info (벡터 DB)
- **Redis** - Q-Metrics (캐시)

### AI/ML
- **Ollama** - 로컬 LLM 추론
- **Tesseract OCR** - 문제 이미지 인식
- **BKT/IRT** - 학습 추적 및 문제 난이도 측정

### Document Processing
- **GROBID** - 학술 논문 파싱
- **pdfplumber** - PDF 테이블 추출
- **Typst** - 고품질 PDF 생성

### Frontend
- **React 19 + TypeScript** - Q-DNA, Logic Engine
- **React Flow** - 지식 그래프 시각화
- **TailwindCSS** - 스타일링

---

## 🎯 Use Cases

### 1. 학생 - 개인화 학습
```
학생 로그인
  → Q-DNA: 현재 실력 분석 (BKT)
  → Q-DNA: 맞춤 문제 추천 (IRT)
  → Q-Metrics: 풀이 분석 및 피드백
```

### 2. 교사 - 시험 분석
```
시험지 업로드
  → Q-Metrics: OCR + 시맨틱 분석
  → Q-Metrics: 교육공학 프레임워크 적용
  → Q-Metrics: 분석 리포트 생성
```

### 3. 연구자 - 이론 탐색
```
논문 PDF 업로드
  → Logic Engine: GROBID 파싱
  → Logic Engine: 개념 추출 (LLM)
  → Logic Engine: Neo4j 지식 그래프 구축
  → Logic Engine: GraphRAG 질의
```

### 4. 학부모 - 학교 정보 조회
```
학교명 검색
  → School Info: 크롤링 (schoolinfo.go.kr)
  → School Info: PDF → Enhanced JSON
  → School Info: RAG 질의응답
  → School Info: 학교별 평가 계획 제공
```

---

## 📈 Roadmap

### Phase 1: Core Services (✅ 완료)
- [x] Logic Engine - 교육 이론 지식 그래프
- [x] Q-DNA - 문제 은행 + BKT/IRT
- [x] School Info - 학교 정보 RAG
- [x] mathesis-common - 공통 라이브러리 구조 확립

### Phase 2: Module-Based Architecture (✅ 완료)
- [x] **mathesis_core 핵심 모듈 구축**
  - [x] Vision Module (OCREngine) - 이미지 → 텍스트/LaTeX
  - [x] Analysis Module (DNAAnalyzer) - 문제 DNA 추출
  - [x] Generation Module (ProblemGenerator) - 문제 생성
  - [x] Prompts Module - LLM 프롬프트 중앙화
- [x] **Node 2 리팩토링** - mathesis_core 모듈 활용
- [x] **Node 7 구축 & 리팩토링** - Anki 시스템 + mathesis_core 통합
- [x] **TDD 테스트 작성** - 30개 테스트, 97% 커버리지

### Phase 3: Advanced Features (🚧 진행 중)
- [x] Q-Metrics - 시험 분석
- [ ] **Node 0 (Student Hub)** - 마스터 노드 구현
- [ ] Gen Node - AI 문제 생성 (mathesis_core 활용 가능)
- [ ] Lab Node - 학습 추적
- [ ] Report Node - 진단 리포트
- [ ] API Gateway 구축
- [ ] 이벤트 기반 통신 (RabbitMQ/Kafka)
- [ ] 중앙 인증/인가 (Keycloak)

### Phase 3: Production Ready (📋 계획)
- [ ] Kubernetes 배포
- [ ] Service Mesh (Istio)
- [ ] 중앙 로깅 (ELK Stack)
- [ ] 모니터링 (Prometheus + Grafana)
- [ ] CI/CD 파이프라인 (GitHub Actions)

---

## 🤝 Contributing

### Development Workflow
```bash
# 1. Fork & Clone
git clone <your-fork>

# 2. Create feature branch
git checkout -b feature/your-feature

# 3. Develop (각 서비스는 독립 개발 가능)
cd node2_q_dna
python main.py

# 4. Test
pytest tests/

# 5. Submit PR
```

### Code Standards
- **Python**: PEP 8, Type Hints 필수
- **TypeScript**: ESLint + Prettier
- **Documentation**: 모든 API는 OpenAPI 명세 작성

---

## 📞 Support

- **Documentation**: [/docs](./architecture/)
- **Issues**: GitHub Issues
- **Discussions**: GitHub Discussions

---

## 📄 License

MIT License - see [LICENSE](../LICENSE) for details

---

## 🙏 Acknowledgments

- **Ollama** - 로컬 LLM 실행 환경
- **Neo4j** - 지식 그래프 데이터베이스
- **FastAPI** - 고성능 Python 웹 프레임워크
- **schoolinfo.go.kr** - 학교 공시 데이터

---

**Last Updated**: 2026-01-10
**Version**: 1.2.0
**Architecture**: Microservices (MSA) + Module-Based Core (mathesis_core)
