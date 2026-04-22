# 🥟 Frien-chy

> **공정거래위원장상(우수상) 수상작** — 공정거래 데이터 활용 공모전 AI 모델 개발 부문 전체 2위 (RAG 성능 평가)

프랜차이즈 예비 창업자를 위한 **도메인 특화 검색·질의응답 시스템**입니다.  
46종의 프랜차이즈 공공데이터를 기반으로, 사용자 질문에 대해 관련 근거 문서를 정확하게 찾아 응답합니다.

<br>

## 📌 핵심 성과

### 공모전 제출 버전 — Retrieval 구조 비교 실험

| Retriever | Hit@5 | MRR |
|---|---|---|
| BM25 | 0.096 | 0.098 |
| KiwiBM25 | 0.635 | 0.496 |
| FAISS (Dense) | 0.750 | 0.606 |
| **Ensemble (KiwiBM25 + FAISS)** | **0.923** | **0.611** |

- BM25 대비 Hit@5 **약 9.6배 향상**
- 해당 Ensemble 구조로 공정거래 데이터 활용 공모전 RAG 성능 평가 **전체 2위** → **공정거래위원장상(우수상)** 수상

### 서비스 운영 버전 — Qdrant 기반 재설계 (최종)

수상 이후 범정부 공공데이터 활용 창업경진대회 참가를 계기로, 실서비스 운영을 고려한 구조로 재설계했습니다.

| 항목 | 기존 (공모전 버전) | 최종 (서비스 버전) |
|---|---|---|
| **Retriever** | Ensemble (KiwiBM25 + FAISS) | Qdrant Dense-only |
| **벡터 DB** | FAISS (메모리 로드 방식) | Qdrant (서버형 DB) |
| **임베딩 모델** | multilingual-e5-large (512 token) | KURE-v1 (2048 token) |
| **데이터 갱신** | 인덱스 재빌드 필요 | 실시간 업데이트 가능 |
| **메타데이터 필터링** | 미지원 | gRPC + payload index |

**전환 이유**: FAISS는 인덱스를 메모리에 로드하는 구조라 데이터 갱신과 운영 관리에 한계가 있었습니다. Qdrant로 전환하면서 데이터 업데이트 용이성, latency 안정성, 운영 비용을 함께 개선했고, 임베딩 모델 교체와 청크 전략 재설계를 통해 Dense-only 구조로도 성능 저하를 최소화했습니다.

<br>

## 🏗️ 시스템 아키텍처

```
[사용자 질문]
     │
     ▼
[질문 이해 & 재구성]
  - 유사어 처리 (Gemini Flash)
  - 대화기록 조회 (PostgreSQL)
     │
     ▼
[문서 검색 & 필터링]
  - VectorDB 검색 (Qdrant)
  - 다층 스코어링 + 동적 임계값
  - 문서 top7 최종 선정
     │
     ▼
[답변 생성]
  - 프롬프팅 + Gemini Flash
  - 출처 문서 함께 반환
```

**3-Tier 서빙 아키텍처**
- **프레젠테이션 티어**: 웹 UI (프론트엔드 분리)
- **애플리케이션 티어**: FastAPI + RAG 파이프라인
- **데이터 티어**: Qdrant (벡터 DB) + PostgreSQL (사용자 기록)

<br>

## 🔍 검색 품질 개선 과정

### 문제 정의
프랜차이즈 정보공개서는 **브랜드명 변형**, **한국어 형태론**, **긴 문서 구조** 특성으로 인해 일반 BM25 검색에서 누락과 relevance 저하가 빈번하게 발생했습니다.

### 1단계: Hybrid Retrieval 설계 (공모전 버전)
- 기존 BM25의 한국어 형태소 처리 한계 → **KiwiBM25** 도입으로 Hit@5 0.096 → 0.635
- 의미 기반 검색을 위한 **FAISS Dense Retrieval** 결합
- Sparse + Dense **Ensemble 구조**로 Hit@5 0.923 달성 → 수상

### 2단계: 서빙 최적화 (서비스 버전)
- FAISS 메모리 방식의 데이터 갱신 한계 → **Qdrant 서버형 벡터 DB** 로 전환
  - Qdrant 채택 이유: gRPC 지원, 메타데이터 필터링, payload index 최적화
- 임베딩 모델 교체: `multilingual-e5-large` (512 token) → **`nlpai-lab/KURE-v1`** (2048 token)
  - 확장된 입력 길이로 문맥 손실 최소화하는 청크 전략 재설계
- Hybrid Retriever 의존도를 줄이고 **Dense-only 구조**로 전환하면서 성능 저하 최소화

### 3단계: Memory-Aware Retrieval
- **PostgreSQL 기반 세션 메모리** 도입
- 이전 대화의 브랜드·업종 맥락을 Gemini 2.5 Flash가 query rewriting에 반영
- 단발성 질의가 아닌 대화 흐름이 이어지는 챗봇 경험 구현

<br>

## 🛠️ Tech Stack

| 분류 | 기술 |
|---|---|
| **언어 / 프레임워크** | Python, FastAPI, Uvicorn |
| **LLM** | Gemini 2.5 Flash / Pro |
| **검색** | Qdrant, FAISS, KiwiBM25, HuggingFace Embeddings (KURE-v1) |
| **오케스트레이션** | LangGraph, LangChain |
| **데이터베이스** | PostgreSQL, SQLiteCache |
| **인프라** | Docker, DigitalOcean (Linux) |
| **통신** | FastAPI SSE (Server-Sent Events) |

<br>

## 📂 파일 구조

```
Frien-chy/
├── main_inference.py       # FastAPI 서버 + RAG 메인 파이프라인
├── build_db_final.py       # Qdrant 벡터 DB 구축 (최종)
├── build_db_11.py          # Qdrant 벡터 DB 구축 (v11)
├── update_db.py            # 벡터 DB 데이터 업데이트
├── migrate_data.py         # 데이터 마이그레이션
├── brand_name.py           # 브랜드명 정규화 유틸리티
├── check_qdrant_data.py    # Qdrant 데이터 검증
├── brand_list_from_qdrant.csv  # Qdrant에서 추출한 브랜드 목록
├── Dockerfile              # Docker 컨테이너 설정
└── requirements.txt        # 의존성 목록
```

<br>

## 🚀 실행 방법

### 환경 변수 설정

```bash
GEMINI_API_KEY=your_gemini_api_key
QDRANT_URL=your_qdrant_url
QDRANT_API_KEY=your_qdrant_api_key
DATABASE_URL=your_postgresql_url
```

### Docker로 실행

```bash
docker build -t frien-chy .
docker run -p 8000:8000 --env-file .env frien-chy
```

### 로컬 실행

```bash
pip install -r requirements.txt
uvicorn main_inference:app --host 0.0.0.0 --port 8000
```

### 벡터 DB 구축

```bash
# 최초 DB 구축
python build_db_final.py

# 데이터 업데이트
python update_db.py
```

<br>

## 🏆 수상

- **공정거래위원장상(우수상)** — 공정거래위원회 주관 「공정거래 데이터 활용 공모전」 AI 모델 개발 부문
  - 프랜차이즈 공공데이터 기반 RAG 질의응답 시스템으로 **RAG 성능 평가 전체 2위** 기록
  - 수상을 통해 **[범정부 공공데이터 활용 창업경진대회]** 참가

<br>

## 👤 Author

**이서정 (Lee Seojung)**  
M.S. in Mathematics, Ewha Womans University  
📝 [AI 논문 정리 블로그](https://haeseung-jeon.notion.site/ENIRC-s-Blog-18000414c2f8800dbac0f486568b7b9a?pvs=74)
