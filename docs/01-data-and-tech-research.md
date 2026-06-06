# 데이터 & 기술 리서치 — 여행지 추천 RAG 플랫폼

> 작성일: 2026-06-06 · 상태: **리서치/기술결정 진행 중** (확정 아님)
> 이 문서는 "어떤 데이터를 쓰고, 어떤 기술을 쓸지"를 정리한 살아있는 문서다. 결정이 바뀌면 갱신한다.

---

## 1. 프로젝트 개요

- **무엇**: 대한민국 여행지를 **대화형 챗봇**으로 추천하는 웹 플랫폼
- **핵심 기법**: **RAG** (Retrieval-Augmented Generation) 중심. 파인튜닝/LoRA는 선택.
- **1차 목표**: **부트캠프 포트폴리오** — 딥러닝/LLM 기술(RAG 등)을 직접 구현하고 보여주는 것이 핵심.

## 2. 제약사항 (확정)

| 항목 | 내용 | 영향 |
|---|---|---|
| 하드웨어 | 노트북 **GTX 1650 Mobile, VRAM 4GB** / i7-9750H / RAM 15GB | 로컬 LLM 파인튜닝 불가. RAG는 가능. |
| 목표 | 부트캠프 포트폴리오 | RAG를 제대로 보여주는 데 집중 |
| UX | 대화형 챗봇 | 스트리밍 응답 필요 |
| 데이터 범위 | **수도권으로 축소** (서울/인천/경기) | 수집량·비용 관리 가능 |

## 3. 핵심 결정사항

### 3-1. 데이터 소스 → **한국관광공사 TourAPI**
- 약 26만 건의 국문 관광정보(관광지/숙박/음식점/행사 등 15종)를 제공하는 공식 OpenAPI.
- 라이선스 깨끗(크롤링 법적 부담 없음), `개요(overview)` 필드가 RAG에 쓸 텍스트를 제공.
- **오해 정정**: "TourAPI를 쓰면 RAG를 못 쓴다" → 틀림. TourAPI는 **데이터 소스**, RAG는 **기법**. TourAPI로 받은 데이터를 임베딩해 벡터DB에 넣으면 그게 RAG다. 둘은 한 세트.

### 3-2. RAG vs 파인튜닝 → **RAG 중심, 파인튜닝은 선택**

| | RAG | 파인튜닝 / LoRA |
|---|---|---|
| 용도 | **지식·사실 주입** (여행지 정보) | **말투·형식·행동** (페르소나, 일정표 포맷) |
| 이 프로젝트 | **핵심** | 선택 (하려면 Colab T4에서) |

- 여행지 "정보"를 파인튜닝으로 외우게 하는 건 안티패턴(정보 바뀌면 재학습). 정보는 RAG로.
- 이 노트북(4GB)으로는 7B LoRA/QLoRA 불가(12GB+ 필요). 데모가 필요하면 **Colab/Kaggle 무료 T4(16GB)** 활용.

### 3-3. API 호출 한도는 "최초 수집"에만 영향
```
[최초 1회 배치] TourAPI 호출 → 로컬 DB + 벡터DB 저장   ← 여기만 한도 영향
[서비스 운영 중] 질문 → 로컬 벡터DB 검색 → LLM 답변       ← TourAPI 호출 0회
```
- 개발계정: **1,000 호출/일**. 운영계정은 활용사례 등록 시 증량 가능.
- 기본 목록(`areaBasedList`)은 페이지당 대량 → 수십~수백 호출이면 끝(실행 몇 분).
- `개요(detailCommon)`는 **건당 1 호출** → 전부 받으려면 범위 축소 또는 운영계정 필요.

## 4. 데이터 수집 계획 (수도권)

### 지역코드 / 콘텐츠타입
- 수도권 areaCode: **서울=1, 인천=2, 경기=31**
- contentTypeId: 12 관광지 · 14 문화시설 · 15 축제공연행사 · 25 여행코스 · 28 레포츠 · 32 숙박 · 38 쇼핑 · 39 음식점

### 정확한 건수 확인 (키 발급 후 실행)
`areaBasedList2` 응답의 `totalCount`로 지역×타입 분포를 즉시 확인할 수 있다 (`&_type=json`).
```python
import os, requests
KEY  = os.environ["TOUR_API_KEY"]            # data.go.kr 발급 인증키
BASE = "http://apis.data.go.kr/B551011/KorService2"   # TourAPI 4.0 (구버전 KorService1)
AREAS = {"서울": 1, "인천": 2, "경기": 31}
TYPES = {12:"관광지",14:"문화시설",15:"축제공연",25:"여행코스",
         28:"레포츠",32:"숙박",38:"쇼핑",39:"음식점"}

def count(a, t):
    r = requests.get(f"{BASE}/areaBasedList2", params={
        "serviceKey":KEY,"MobileOS":"ETC","MobileApp":"tripRAG","_type":"json",
        "numOfRows":1,"pageNo":1,"areaCode":a,"contentTypeId":t}, timeout=10)
    return int(r.json()["response"]["body"]["totalCount"])

grand = 0
for an, ac in AREAS.items():
    sub = sum(count(ac, t) for t in TYPES)
    print(f"{an}: {sub:,}건"); grand += sub
print(f"수도권 전체 ≈ {grand:,}건")
```
- 기본정보(이름·주소·좌표·카테고리)는 하루 한도로 **전부 수집 가능**.
- `개요`까지 전부 받으려면: ①운영계정 증량 ②카테고리 축소(예: 관광지+문화시설+음식점) ③throttle 중 택1. (포트폴리오는 ② 권장)

## 5. 기술 스택 (RAG 파이프라인)

| 단계 | 선택 | 비고 |
|---|---|---|
| 데이터 수집 | TourAPI → **SQLite**(또는 PostgreSQL) | 배치 스크립트, 1회성 |
| 임베딩 모델 | **BGE-m3** (또는 한국어 특화 **KURE**) | 4GB GPU/CPU에서 구동 OK |
| 벡터DB | **Chroma / FAISS / pgvector** | 로컬 |
| 검색 | **벡터 + BM25 하이브리드** (`kiwipiepy` 형태소 분석) | 한국어에서 특히 효과적 |
| 생성 LLM | **Claude API (Sonnet 4.6 권장)** | 한국어 우수, GPU 불필요 |
| 백엔드 | **FastAPI** | RAG 오케스트레이션 + 스트리밍 |
| 프론트 | **React / Next.js** | 챗봇 UI |
| 파인튜닝(선택) | Colab/Kaggle T4에서 LoRA 데모 | 스타일/페르소나용 |

### 생성 LLM — Claude API 상세 (레퍼런스 기준, 2026-05-26 캐시)

| 모델 | 입력/출력 ($/1M tokens) | 컨텍스트 | 메모 |
|---|---|---|---|
| Claude Haiku 4.5 | $1 / $5 | 200K | 가장 저렴 |
| **Claude Sonnet 4.6** | $3 / $15 | 1M | **추천 — 균형** |
| Claude Opus 4.8 | $5 / $25 | 1M | 최고 품질(과함) |

- RAG에선 LLM은 "검색된 후보로 친근한 한국어 답변 작성"만 담당 → **Sonnet 4.6이면 충분**, 더 아끼려면 Haiku 4.5.
- 대화 1턴 비용 ≈ **$0.01~0.02** (입력 2~4K + 출력 0.5K 기준). 데모 수천 번도 몇 달러.
- **스트리밍**: 챗봇 UI는 `client.messages.stream()`로 실시간 토큰 출력(타임아웃 방지 + UX).
- **프롬프트 캐싱**: 고정 시스템 프롬프트/검색 컨텍스트 캐시 시 입력 비용 ~90% 절감.

## 6. 목표 아키텍처

```
[수집 배치]  TourAPI(수도권) ──▶ SQLite(정형) + 임베딩 ──▶ 벡터DB(Chroma/FAISS)
                                                              │
[서비스]  사용자 ⇄ React 챗봇 ⇄ FastAPI
                                   │  1) 질의 임베딩 → 하이브리드 검색(벡터+BM25)
                                   │  2) 상위 후보 + 질의 → Claude API(스트리밍)
                                   └─ 3) 근거(관광지명·주소) 포함 추천 답변 반환
```

## 7. 미해결 / 다음 단계
- [ ] TourAPI 키 발급 → 수도권 실제 건수 확정
- [ ] 개요(overview) 수집 범위 확정 (전체 vs 카테고리 축소)
- [ ] 생성 LLM 최종 확정 (Sonnet 4.6 vs Haiku 4.5)
- [ ] 임베딩 모델 확정 (BGE-m3 vs KURE)
- [ ] 벡터DB 확정 (Chroma vs FAISS vs pgvector)
- [ ] (선택) 파인튜닝/LoRA 데모 포함 여부

## 8. 참고 링크
- [한국관광공사 국문 관광정보 서비스 (공공데이터포털)](https://www.data.go.kr/data/15101578/openapi.do)
- [한국관광공사 TourAPI 공식](https://www.2025tourapi.com/sub/sub01.html) · [관광콘텐츠랩 API](https://api.visitkorea.or.kr/)
- [한국어 임베딩 모델 RAG 벤치마크 (ssisOneTeam)](https://github.com/ssisOneTeam/Korean-Embedding-Model-Performance-Benchmark-for-Retriever)
- [임베딩 모델 선택 가이드](https://www.data-dynamics.io/ko/blog/embedding-model-guide)
- [TourAPI 이용 안내](https://studygov.kr/blog/13071-tourism-info-service/)
- [LLM Fine-Tuning Hardware Requirements](https://llmhardware.io/guides/llm-fine-tuning-hardware-requirements)
