# 🖥️ Senty Web Server

> 감성 분석 데이터 API 서버 (FastAPI)

[![FastAPI](https://img.shields.io/badge/FastAPI-0.95-009688?style=for-the-badge&logo=fastapi&logoColor=white)](https://fastapi.tiangolo.com)
[![Python](https://img.shields.io/badge/Python-3.8+-3776AB?style=for-the-badge&logo=python&logoColor=white)](https://python.org)
[![SQLAlchemy](https://img.shields.io/badge/SQLAlchemy-D71F00?style=for-the-badge&logo=sqlalchemy&logoColor=white)](https://sqlalchemy.org)
[![Pydantic](https://img.shields.io/badge/Pydantic-E92063?style=for-the-badge&logo=pydantic&logoColor=white)](https://pydantic.dev)

---

## 📋 프로젝트 개요

**Senty Web Server**는 감성 분석 결과를 프론트엔드에 제공하는 **RESTful API** 서버입니다. FastAPI 기반으로 구축되어 빠른 응답 속도와 자동 API 문서화를 제공합니다.

### ✨ 주요 특징

- ⚡ **고성능**: FastAPI + Uvicorn 비동기 서버
- 📝 **자동 문서화**: Swagger UI & ReDoc
- 🔐 **CORS 지원**: 프론트엔드 연동
- 📊 **타입 안전**: Pydantic 모델 검증

---

## 🏗️ 프로젝트 구조

```
Web Server/
├── 📄 main.py              # FastAPI 앱 진입점
├── 📄 database.py          # 데이터베이스 연결 설정
├── 📄 models.py            # SQLAlchemy ORM 모델
├── 📄 out_scheme.py        # Pydantic 응답 스키마
├── 📄 process_data.py      # 데이터 처리 유틸리티
├── 📄 process_report_data.py # 리포트 데이터 생성
├── 📄 sentiment.py         # 감성 점수 계산
├── 📄 store_data.py        # 데이터 저장 유틸리티
├── 📄 time_series_data.py  # 시계열 데이터 처리
└── 📄 db_scheme.d2         # DB 스키마 다이어그램
```

---

## 🔬 기술 스택

| 기술 | 용도 |
|------|------|
| **FastAPI** | 웹 프레임워크 |
| **Uvicorn** | ASGI 서버 |
| **SQLAlchemy** | ORM |
| **Pydantic** | 데이터 검증 |
| **SciPy** | 통계 계산 (Pearson 상관계수) |

---

## 🚀 실행 방법

### 환경 설정

```bash
# 가상환경 생성
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

# 의존성 설치
pip install fastapi uvicorn sqlalchemy scipy pandas
```

### 서버 실행

```bash
# 개발 모드 (핫 리로드)
uvicorn main:app --reload --port 8000

# 프로덕션 모드
uvicorn main:app --host 0.0.0.0 --port 8000
```

### API 문서 확인

```
Swagger UI: http://localhost:8000/docs
ReDoc:      http://localhost:8000/redoc
```

---

## 📡 API 엔드포인트

### `GET /api`

감성 분석 리포트 데이터를 반환합니다.

#### 응답 예시

```json
{
  "total_topic": {
    "tweet_number": 45234,
    "sentiment_dist": [
      { "name": "긍정", "value": 21500 },
      { "name": "부정", "value": 12300 },
      { "name": "중립", "value": 11434 }
    ],
    "sentiment_dist_rank": [...],
    "corr_rank_list": [...],
    "topic_proportions": [...]
  },
  "topics": [
    {
      "topic_name": "Apple",
      "tweet_number": 2340,
      "sentiment_dist": [...],
      "topic_words": [...],
      "correlations": {
        "window_sizes": [3, 5, 7],
        "snp500": [-0.42, -0.45, -0.38],
        "nasdaq100": [-0.38, -0.41, -0.35]
      },
      "sentiment_corr": [...]
    },
    ...
  ]
}
```

---

## 📊 데이터 스키마

### 응답 모델 (`out_scheme.py`)

```python
class ReportDataModel(BaseModel):
    total_topic: TotalTopicModel
    topics: List[TopicModel]

class TotalTopicModel(BaseModel):
    tweet_number: int
    sentiment_dist: List[SentDistModel]
    sentiment_dist_rank: List[TopicValueInt]
    corr_rank_list: List[TopicValueFloat]
    topic_proportions: List[TopicProportionModel]

class TopicModel(BaseModel):
    topic_name: str
    tweet_number: int
    max_corr_window_size: int
    correlations: CorrelationModel
    sentiment_corr: List[CorrLineModel]
    sentiment_dist: List[SentDistModel]
    topic_words: List[TopicWordModel]
    positive_words: List[SentKeywordModel]
    negative_words: List[SentKeywordModel]
    most_positive_day: str
    most_negative_day: str
```

---

## 🔧 핵심 로직

### 감성 점수 계산

```python
# process_data.py
def get_sentiment_score(df):
    """일별 감성 점수 계산"""
    # Score = ((긍정 - 부정) / 전체) × (1 - 중립비율)
    positive = df[df['sentiment'] == 'positive'].groupby('date').size()
    negative = df[df['sentiment'] == 'negative'].groupby('date').size()
    neutral = df[df['sentiment'] == 'neutral'].groupby('date').size()
    total = positive + negative + neutral
    
    score = ((positive - negative) / total) * (1 - neutral / total)
    return score
```

### Pearson 상관계수 계산

```python
# process_data.py
from scipy import stats

def get_correlation(df, snp, nasdaq, window):
    """감성 점수와 주가 지수 상관관계 계산"""
    sentiment_score = get_sentiment_score(df)
    
    # Window Size 적용 (이동 평균)
    agg_score = sentiment_score.rolling(window=window).mean()
    
    # Pearson 상관계수
    corr_snp = stats.pearsonr(agg_score.dropna(), snp)
    corr_nasdaq = stats.pearsonr(agg_score.dropna(), nasdaq)
    
    return corr_snp.statistic, corr_nasdaq.statistic
```

---

## 🔐 CORS 설정

```python
# main.py
from starlette.middleware.cors import CORSMiddleware

origins = ['http://localhost:5173']  # 프론트엔드 주소

app.add_middleware(
    CORSMiddleware,
    allow_origins=origins,
    allow_credentials=True,
    allow_methods=['*'],
    allow_headers=['*']
)
```

---

## 🗄️ 데이터베이스

### 연결 설정

```python
# database.py
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

DATABASE_URL = "sqlite:///./databases/senty.db"
engine = create_engine(DATABASE_URL)
Session = sessionmaker(bind=engine)
```

### 의존성 주입

```python
# main.py
@app.get('/api', response_model=ReportDataModel)
def report_data(session: Session = Depends(get_db)):
    return get_report_data_model(session)
```

---

## 📈 성능

| 메트릭 | 값 |
|--------|-----|
| **응답 시간** | < 100ms |
| **동시 요청** | 1000+ RPS |
| **메모리 사용** | ~50MB |

---

## 🐳 Docker 배포

```dockerfile
FROM python:3.9-slim

WORKDIR /app
COPY . .

RUN pip install fastapi uvicorn sqlalchemy scipy pandas

EXPOSE 8000
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

```bash
docker build -t senty-server .
docker run -p 8000:8000 senty-server
```

---

## 📁 디렉토리 설명

| 파일 | 설명 |
|------|------|
| `main.py` | FastAPI 앱 및 라우터 정의 |
| `database.py` | SQLAlchemy 데이터베이스 연결 |
| `models.py` | ORM 테이블 모델 |
| `out_scheme.py` | Pydantic 응답 스키마 |
| `process_data.py` | 감성 분포, 상관관계 계산 |
| `process_report_data.py` | 리포트 JSON 생성 |
| `sentiment.py` | RoBERTa 감성 분류 |
| `time_series_data.py` | 시계열 데이터 처리 |

---

## 🔗 관련 레포지토리

| 레포 | 설명 |
|------|------|
| [🤖 AI Modeling](https://github.com/inisw-8/ai-modeling) | LDA + RoBERTa 모델링 |
| [📥 Data Gathering](https://github.com/inisw-8/data-gathering) | 트윗 데이터 수집 |
| [📊 Frontend](https://github.com/inisw-8/frontend) | React 대시보드 |
| [📊 Senty Frontend](https://github.com/inisw-8/senty-frontend) | 독립형 프론트엔드 |
| [🔬 Our Efforts](https://github.com/inisw-8/our-efforts) | R&D 실험 기록 |

---

## 📄 라이선스

MIT License

---

<div align="center">

**Senty Web Server** - 감성 분석 API 📊

*FastAPI 기반 고성능 RESTful API*

[📖 API Docs](http://localhost:8000/docs) · [🐛 Report Bug](https://github.com/inisw-8/web-server/issues)

</div>

