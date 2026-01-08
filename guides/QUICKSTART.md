# Mathesis Platform - Quick Start Guide

> 전체 시스템 30분 만에 실행하기

---

## 🎯 Goal

로컬 환경에서 Mathesis의 4개 마이크로서비스를 모두 실행하고, 각 서비스를 테스트합니다.

---

## 📋 Prerequisites

### 필수 소프트웨어

```bash
# 1. Docker & Docker Compose
docker --version        # 24.0 이상
docker-compose --version  # 2.20 이상

# 2. Python
python3 --version       # 3.11 이상

# 3. Node.js (프론트엔드 개발 시)
node --version          # 18 이상
```

### Ollama 설치 (로컬 LLM)

```bash
# 1. Ollama 설치
curl -fsSL https://ollama.com/install.sh | sh

# 2. 필요한 모델 다운로드
ollama pull llama3:latest              # 생성 모델 (4.7GB)
ollama pull nomic-embed-text:latest    # 임베딩 모델 (274MB)
ollama pull llama3.2-vision:11b        # Vision 모델 (6.0GB) - Q-DNA용

# 3. Ollama 서버 실행 확인
ollama serve
curl http://localhost:11434/api/version
```

---

## 🚀 Installation

### 1. Clone Repository

```bash
git clone <repository-url>
cd mathesis
```

### 2. Environment Setup

```bash
# 환경 변수 파일 생성
cp .env.example .env

# 편집 (필요시)
nano .env
```

**`.env` 예시**:
```bash
# Database
POSTGRES_PASSWORD=your_secure_password
NEO4J_PASSWORD=your_neo4j_password

# Ollama
OLLAMA_BASE_URL=http://localhost:11434

# Ports
NODE1_PORT=8001
NODE2_PORT=8002
NODE5_PORT=8005
NODE6_PORT=8006
```

### 3. Install mathesis-common

```bash
# 공통 라이브러리 설치 (editable mode)
pip install -e ./mathesis-common
```

---

## 🐳 Docker Compose (전체 스택 실행)

### 1. Start All Services

```bash
# 전체 서비스 시작
docker-compose up -d

# 로그 확인
docker-compose logs -f
```

### 2. Verify Services

```bash
# 서비스 상태 확인
docker-compose ps

# 예상 출력:
# NAME                STATUS       PORTS
# mathesis-node1      Up           0.0.0.0:8001->8001/tcp
# mathesis-node2      Up           0.0.0.0:8002->8002/tcp
# mathesis-node5      Up           0.0.0.0:8005->8005/tcp
# mathesis-node6      Up           0.0.0.0:8006->8006/tcp
# mathesis-postgres   Up           0.0.0.0:5432->5432/tcp
# mathesis-neo4j      Up           0.0.0.0:7474->7474/tcp, 0.0.0.0:7687->7687/tcp
# mathesis-redis      Up           0.0.0.0:6379->6379/tcp
```

### 3. Access Services

| Service | URL | Description |
|---------|-----|-------------|
| **Node 1** | http://localhost:8001/docs | Logic Engine API |
| **Node 2** | http://localhost:8002/docs | Q-DNA API |
| **Node 5** | http://localhost:8005/docs | Q-Metrics API |
| **Node 6** | http://localhost:8006/docs | School Info API |
| **Neo4j** | http://localhost:7474 | Graph Database UI |
| **Postgres** | `localhost:5432` | psql 접속 |

---

## 🔧 개별 서비스 실행 (개발 모드)

각 노드는 독립적으로 실행 가능합니다.

### Node 1: Logic Engine

```bash
cd node1_logic_engine

# 의존성 설치
pip install -r requirements.txt

# 서버 실행
python main.py

# 접속: http://localhost:8001/docs
```

### Node 2: Q-DNA

```bash
cd node2_q_dna

# Backend
pip install -r requirements.txt
python main.py  # http://localhost:8002

# Frontend (별도 터미널)
cd frontend
npm install
npm run dev  # http://localhost:5173
```

### Node 5: Q-Metrics

```bash
cd node5_q_metrics

pip install -r requirements.txt
python main.py  # http://localhost:8005/docs
```

### Node 6: School Info

```bash
cd node6_school_info

pip install -r requirements.txt
python main.py  # http://localhost:8006/docs
```

---

## ✅ 기능 테스트

### Test 1: School Info RAG System

```bash
cd node6_school_info

# RAG 시스템 테스트
python3 test_enhanced_rag.py

# 예상 출력:
# ✓ PDF 파싱
# ✓ Enhanced JSON 생성
# ✓ Vector Store 색인
# ✓ 질의응답 성공
```

**API 테스트**:
```bash
# Health Check
curl http://localhost:8006/health

# RAG 질의
curl -X POST http://localhost:8006/rag/query \
  -H "Content-Type: application/json" \
  -d '{"question": "수학 수행평가 비율은?", "k": 2}'
```

### Test 2: Q-DNA 문제 등록

```bash
# 문제 업로드 (이미지)
curl -X POST http://localhost:8002/api/v1/questions/upload \
  -F "file=@sample_question.jpg" \
  -F "subject=math" \
  -F "grade=1"

# 자동 태깅 확인
curl http://localhost:8002/api/v1/questions/1
```

### Test 3: Logic Engine 논문 파싱

```bash
# 논문 업로드
curl -X POST http://localhost:8001/api/v1/papers/upload \
  -F "file=@sample_paper.pdf"

# 지식 그래프 조회
curl http://localhost:8001/api/v1/graph/concepts
```

---

## 🗄️ Database Setup

### PostgreSQL

```bash
# 접속
psql -h localhost -U postgres -d mathesis

# 데이터베이스 생성 (Docker Compose가 자동 생성)
# - logic_engine
# - q_dna
# - q_metrics
```

### Neo4j

```bash
# 브라우저에서 접속
open http://localhost:7474

# 로그인
Username: neo4j
Password: (환경 변수에서 설정한 값)

# Cypher 쿼리 테스트
MATCH (n) RETURN n LIMIT 10
```

### ChromaDB (Node 6)

```bash
# 로컬 파일 기반 (자동 생성)
ls node6_school_info/chroma_hierarchical/
```

---

## 🐛 Troubleshooting

### Ollama 연결 실패

```bash
# Ollama 재시작
sudo systemctl restart ollama

# 또는 수동 실행
ollama serve

# 연결 확인
curl http://localhost:11434/api/version
```

### Docker Compose 오류

```bash
# 컨테이너 재시작
docker-compose restart

# 로그 확인
docker-compose logs node1 node2 node5 node6

# 전체 재구축
docker-compose down
docker-compose up --build -d
```

### 포트 충돌

```bash
# 사용 중인 포트 확인
lsof -i :8001
lsof -i :8002

# 프로세스 종료
kill -9 <PID>
```

### Database 초기화

```bash
# PostgreSQL 데이터 삭제
docker-compose down -v
docker-compose up -d postgres

# Neo4j 데이터 삭제
docker exec mathesis-neo4j cypher-shell -u neo4j -p <password> "MATCH (n) DETACH DELETE n"
```

---

## 📊 Sample Data

### 샘플 데이터 로드

```bash
# Q-DNA 샘플 문제
cd node2_q_dna
docker-compose exec backend python -m scripts.seed_database

# Logic Engine 샘플 논문
cd node1_logic_engine
python scripts/import_sample_papers.py

# School Info 샘플 PDF
cd node6_school_info
python test_enhanced_rag.py  # 샘플 PDF 자동 색인
```

---

## 🎓 Next Steps

### 1. API 탐색
각 서비스의 Swagger UI에서 API 테스트:
- http://localhost:8001/docs
- http://localhost:8002/docs
- http://localhost:8005/docs
- http://localhost:8006/docs

### 2. 프론트엔드 개발
```bash
cd node2_q_dna/frontend
npm run dev
```

### 3. 문서 읽기
- [MSA Architecture](../architecture/01_MSA_ARCHITECTURE.md)
- [Service Map](../architecture/02_SERVICE_MAP.md)
- [각 노드별 상세 문서](../../node*/docs/)

---

## 🆘 Support

문제 발생 시:
1. [Troubleshooting Guide](./TROUBLESHOOTING.md)
2. GitHub Issues
3. 개발 팀 문의

---

**Estimated Time**: 30 minutes
**Difficulty**: Intermediate
**Last Updated**: 2026-01-08
