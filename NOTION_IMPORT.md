# Mathesis Platform - Notion Import Guide

> Notion 프로젝트 관리 페이지에 문서를 업로드하는 가이드

---

## 📋 Import Checklist

### 1. 메인 페이지 생성
- [ ] Notion에서 새 페이지 생성: "Mathesis Platform"
- [ ] 아이콘 설정: 🎓
- [ ] 커버 이미지 추가 (선택)

### 2. 문서 Import

#### a) Overview (README.md)
```bash
cat /mnt/d/progress/mathesis/docs/README.md | pbcopy
```
→ Notion 페이지에 붙여넣기

**포함 내용**:
- Vision & 핵심 가치
- MSA Architecture 다이어그램
- Services Overview 표
- Quick Start 링크
- Technology Stack
- Use Cases

#### b) MSA Architecture
```bash
cat /mnt/d/progress/mathesis/docs/architecture/01_MSA_ARCHITECTURE.md | pbcopy
```
→ "Architecture" 하위 페이지 생성 후 붙여넣기

**포함 내용**:
- Architecture Principles (DDD, Service Independence)
- Service Catalog (4개 노드 상세)
- Communication Patterns (REST, Event-Driven, gRPC)
- Data Management (Database per Service, SAGA)
- Monitoring & Security

#### c) Quick Start
```bash
cat /mnt/d/progress/mathesis/docs/guides/QUICKSTART.md | pbcopy
```
→ "Guides" 하위 페이지 생성 후 붙여넣기

**포함 내용**:
- Prerequisites (Docker, Ollama)
- Installation (30분)
- Docker Compose 실행
- 개별 서비스 실행
- 기능 테스트
- Troubleshooting

#### d) System Diagram
```bash
# PlantUML 렌더링 필요
cat /mnt/d/progress/mathesis/docs/diagrams/system_context.puml
```

**렌더링 방법**:
1. http://www.plantuml.com/plantuml/uml/ 접속
2. PlantUML 코드 붙여넣기
3. 생성된 이미지 다운로드
4. Notion 페이지에 이미지 업로드

---

## 🎯 Notion Database 설정 (선택)

### Services Database

Notion Database를 생성하여 각 서비스를 관리:

| Property | Type | Values |
|----------|------|--------|
| **Name** | Title | Logic Engine, Q-DNA, ... |
| **Status** | Select | ✅ Production, 🚧 Beta, 📋 Planned |
| **Port** | Number | 8001, 8002, ... |
| **Domain** | Text | 교육 이론, 문제 은행, ... |
| **Tech Stack** | Multi-select | Python, FastAPI, Neo4j, ... |
| **Repo** | URL | GitHub 링크 |
| **Docs** | Relation | 각 노드 docs 페이지 링크 |

**샘플 데이터**:
```
Name: Logic Engine
Status: ✅ Production
Port: 8001
Domain: 교육 이론 지식 그래프
Tech Stack: Python, Neo4j, Ollama, GROBID
Repo: https://github.com/...
Docs: [Link to node1_logic_engine/docs]
```

---

## 🔗 링크 연결

### 내부 링크
- Overview 페이지에서 각 서비스로 링크
- Architecture 페이지에서 각 노드 상세 문서로 링크

### 외부 링크
- Swagger UI: `http://localhost:8001/docs`
- Neo4j Browser: `http://localhost:7474`
- GitHub Repository
- Issue Tracker

---

## 📊 Notion 템플릿 (복사용)

### Template 1: Service Page

```markdown
# [Service Name]

## 📝 Overview
- **Domain**: [교육 이론 / 문제 은행 / ...]
- **Port**: [8001 / 8002 / ...]
- **Status**: ✅ Production / 🚧 Beta

## 🎯 Responsibilities
- [책임 1]
- [책임 2]
- [책임 3]

## 🛠️ Tech Stack
- **Language**: Python 3.11+
- **Framework**: FastAPI
- **Database**: [PostgreSQL / Neo4j / ChromaDB]
- **LLM**: Ollama

## 🚀 Quick Start
\```bash
cd [service_directory]
python main.py
\```

## 📚 API Endpoints
- `GET /health` - Health check
- `POST /api/v1/...` - [설명]

## 🔗 Links
- [Swagger UI](http://localhost:800X/docs)
- [Detailed Docs](./node_x/docs/)
- [GitHub](https://github.com/...)
```

### Template 2: Architecture Decision Record (ADR)

```markdown
# ADR-001: MSA 도입 결정

## 상태
✅ Accepted

## 컨텍스트
Mathesis 플랫폼은 4개의 독립적인 도메인으로 구성됩니다...

## 결정
마이크로서비스 아키텍처(MSA)를 채택합니다.

## 이유
1. **도메인 독립성**: 각 서비스는 명확한 비즈니스 도메인 담당
2. **기술 자유도**: 서비스별 최적 기술 스택 선택
3. **확장성**: 독립적인 스케일링 가능

## 결과
### 긍정적
- 서비스 독립 배포
- 장애 격리
- 팀별 독립 개발

### 부정적
- 복잡도 증가
- 분산 트랜잭션 어려움
- 모니터링 복잡

## 날짜
2026-01-08
```

---

## ✅ Import 완료 체크리스트

- [ ] 메인 Overview 페이지 생성
- [ ] Architecture 문서 import
- [ ] Quick Start 가이드 import
- [ ] System Diagram 이미지 추가
- [ ] Services Database 생성 (선택)
- [ ] 각 서비스 페이지 링크 연결
- [ ] API 문서 링크 추가
- [ ] Roadmap 섹션 업데이트

---

## 📞 Support

문서 import 관련 문의:
- GitHub Issues
- 프로젝트 관리자

---

**Last Updated**: 2026-01-08
