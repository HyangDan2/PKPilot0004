# Market Intelligence API

기준: `main` (2026-11-17, `49d1551`)의 `backend/market_intel/app.py`, `service.py`, `scoring.py`, `domain.py`,
`backend/.env.example`, `backend/tools/nginx-market-intelligence.conf`에서 직접 확인한 내용.

## 1. 개요

기업 본질 가치와 무관한 경제 외적 주가 변동 요인 10선을 무료 공개 데이터로 태깅하는 FastAPI 백엔드다.
단기 과열(NFDS, Non-Fundamental Drift Score)과 평균 회귀 리스크(MR)를 **다른 점수**로 분리해 반환하고,
실데이터가 아닌 항목은 `data_gaps`에 그대로 노출한다.

- 프런트엔드는 `PKPilot0004` 루트의 정적 페이지(GitHub Pages)이며, 이 백엔드는 Server-HQ 이식 대상 초안이다.
- 엔드포인트는 전부 **읽기 전용 GET**이고 인증이 없다.
- 이 문서는 인수인계 문서(`HANDOFF_ANTIGRAVITY.md` §5.1)의 표를 저장소 정본 위치로 옮기고 요청·응답 스키마를 보강한 것이다.

## 2. 접속

| 구분 | 값 |
| --- | --- |
| 로컬 | `http://127.0.0.1:8791` |
| Nginx 서브패스 | `MI_ROOT_PATH`(예: `/market-intel`) → 공개 `https://hd2lab.duckdns.org/market-intel/api/...` |
| 프런트 연결 | 상단 `API 연결`에 Base URL 입력 또는 `?api=http://127.0.0.1:8791` |
| 정적 자산 | `GET /`(있으면 `backend/static/index.html`), `GET /static/*` |
| 설정 파일 | 서버에서 `/etc/hd2lab/market-intelligence.env`(0600)로 주입, `.env.example`은 형식만 |

실행:

```bash
cd backend
python3 -m venv .venv && .venv/bin/pip install -r requirements.txt
MI_OFFLINE=1 .venv/bin/python -m uvicorn market_intel.app:create_app --factory --host 127.0.0.1 --port 8791
```

`app`은 의도적으로 `None`이다. 반드시 `--factory`로 만들어 환경변수 주입 시점을 고정한다.

CORS: `MI_CORS_ORIGINS`(기본값은 `https://hyangdan2.github.io` + 로컬 포트들)만 허용, `allow_credentials=False`,
허용 메서드는 `GET`, `OPTIONS`뿐, `max_age=600`.

## 3. 인증·권한

없음. 모든 경로가 공개 읽기다.

- 쓰기 엔드포인트가 아예 없어 CSRF·세션·토큰 계약이 존재하지 않는다.
- 남용 방지는 앞단 Nginx(속도 제한)와 무료 API 호출 제한에 의존한다.
- `OPENDART_API_KEY`는 서버 환경변수로만 두고 응답에 포함하지 않는다. `/api/health`는 키의 존재 여부만 상태로 알린다.

## 4. 엔드포인트

합계 **6개**.

| Method | Path | 인증 | 설명 |
| --- | --- | --- | --- |
| GET | `/api/health` | 없음 | 서비스 상태·커넥터 상태·도메인 정합성·CORS 출처 |
| GET | `/api/factors` | 없음 | 4축·10요인·메커니즘·투자자 원칙·면책 |
| GET | `/api/symbols` | 없음 | 예시 종목 10개와 입력 힌트 |
| GET | `/api/analyze?q=` | 없음 | 종목 분석(쿼리) |
| GET | `/api/analyze/{ticker}` | 없음 | 같은 분석(경로 변수, 구현은 `q`로 위임) |
| GET | `/api` | 없음 | 서비스 요약과 엔드포인트 목록(OpenAPI 스키마 제외) |

`GET /`(정적 index)와 `/static/*`는 `STATIC_DIR`이 있을 때만 등록된다. 그 밖의 경로는 404, `POST` 등은 405.

### 4-1. `GET /api/health`

```json
{
  "status": "ok",
  "service": "HD2Lab Market Intelligence",
  "slug": "market-intelligence",
  "version": "0.1.0-draft",
  "mode": "online",
  "connectors": {
    "naver_quote":  {"status": "ready", "key_required": false},
    "yahoo_quote":  {"status": "ready", "key_required": false},
    "news_rss":     {"status": "ready", "key_required": false},
    "opendart":     {"status": "disabled", "key_required": true, "reason": "OPENDART_API_KEY 미설정 → 표본 공시 폴백"}
  },
  "generated_at": "2026-11-19T06:40:39+00:00",
  "domain_ok": true,
  "domain_problems": [],
  "cors_origins": ["https://hyangdan2.github.io", "http://127.0.0.1:8791"]
}
```

`MI_OFFLINE=1`이면 `mode`가 `offline`이고 모든 커넥터가 `disabled`다.
`domain_ok=false`는 요인·축 정의가 깨졌다는 뜻이며 `domain_problems`에 사유가 담긴다.

### 4-2. `GET /api/factors`

```json
{
  "service": "HD2Lab Market Intelligence",
  "slug": "market-intelligence",
  "version": "0.1.0-draft",
  "brief_title": "…",
  "axes": [{"id": "AX01", "name": "지배구조·자본 이벤트", "factor_ids": [2, 9], "weight": 0.2}],
  "factors": [{"id": 2, "name": "행동주의 및 주주환원", "axis_id": "AX01", "direction": "up",
               "direction_label": "▲ 환원 / ▼ 분할", "weight": 0.2, "keywords": ["…"], "disclosure_terms": ["…"]}],
  "mechanism": {"center": "…", "outcomes": [{"path": "▲", "label": "단기 급등"}], "inputs": ["지배구조·자본 이벤트"]},
  "principles": [{"type": "▲", "market": "…", "investor": "…"}],
  "core_conclusion": "…",
  "disclaimer": "…"
}
```

4축 가중치 합은 1.0이며 `AX01` 0.20 · `AX02` 0.25 · `AX03` 0.25 · `AX04` 0.30이다. 값은 백테스트 전 가설값이다.

### 4-3. `GET /api/symbols`

```json
{"examples": [{"ticker": "005930", "name": "삼성전자"}], "hint": "국내 6자리 종목코드 또는 해외 티커(예: TSLA)"}
```

`examples`는 10개.

### 4-4. `GET /api/analyze?q=005930`

`q`는 종목명·6자리 코드·해외 티커(영문 1~6자)를 받는다. 결과는 180초 캐시된다(`analysis_ttl`).

```json
{
  "service": "HD2Lab Market Intelligence",
  "slug": "market-intelligence",
  "version": "0.1.0-draft",
  "generated_at": "2026-11-19T06:40:39+00:00",
  "q": "005930",
  "entity": {"ticker": "005930", "name": "삼성전자", "market": "KOSPI", "currency": "KRW", "source": "naver"},
  "mode": "mixed",
  "quote": {
    "available": true, "price": 71200, "change_pct": 1.42, "volume": 12345678,
    "avg_volume_20d": 9876543, "volume_ratio": 1.25, "per": 12.4, "pbr": 1.31,
    "high_52w": 88800, "low_52w": 61500, "ma20": 70100, "ma60": 68250,
    "foreign_net": -123456, "institution_net": 234567, "traded_at": "2026-11-19T06:30:00",
    "source": "naver", "source_url": "…"
  },
  "signals": [
    {"factor_id": 3, "name": "주가지수 편입·리밸런싱", "axis_id": "AX02",
     "direction": "up", "direction_label": "▲ 편입 / ▼ 편출",
     "strength": 0.42, "score": 8.4, "sources": ["news"], "note": "…",
     "evidence": [{"kind": "news", "title": "…", "source": "…", "date": "2026-11-18", "url": "…", "match": ["MSCI"]}]}
  ],
  "scoring": {
    "axes": [{"id": "AX01", "name": "…", "score": 0.31, "factors": [2, 9]}],
    "nfds": {
      "score": 34.3, "weighted": 28.1, "label": "경계", "message": "…",
      "direction": {"code": "UP", "label": "▲ 단기 급등 우위", "up": 0.42, "down": 0.11}
    },
    "mean_reversion": {
      "score": 61.2, "level": "높음", "alert": "…", "message": "…",
      "price_position_score": 58.4, "coverage": 0.83,
      "components": {"range_position": 0.41, "ma60_extension": 0.043, "valuation_premium": 0.12},
      "notes": []
    },
    "valuation": {"range_position": 0.41, "ma60_extension": 0.043, "valuation_premium": 0.12,
                  "components": [], "coverage": 0.83, "notes": []},
    "fundamental": {"status": "ok", "message": "…", "flags": []},
    "principles": [{"factor_id": 2, "action": "…"}],
    "confidence": {"score": 72, "label": "높음", "live_sources": 3, "total_sources": 4, "note": "…"}
  },
  "sources": [{"name": "naver_quote", "status": "live", "items": [], "reason": "", "source_url": "…", "elapsed_ms": 120}],
  "data_gaps": ["OpenDART 키 미설정 → 공시 근거는 표본입니다(무료 키 발급 후 실데이터 전환)."],
  "disclaimer": "…",
  "sample_as_of": ""
}
```

핵심 필드 의미:

| 필드 | 의미 |
| --- | --- |
| `mode` | `live`(모든 근거 실데이터) · `mixed`(일부 표본) · `sample`(전부 표본) |
| `signals[].strength` | 요인 강도 0~1 |
| `signals[].score` | `strength × 가중치 × 100` (0~100 표기) |
| `scoring.axes[].score` | 축 확률적 OR 0~1 |
| `scoring.nfds.score` | 4축 가중 압력을 감마 보정한 0~100 과열 점수 |
| `scoring.mean_reversion.score` | 가격 위치·이격·밸류에이션 기반 되돌림 거리 0~100 |
| `scoring.confidence` | 실데이터 비율과 밸류에이션 확보율로 계산한 초안 신뢰도 |
| `data_gaps` | 실데이터가 아닌 항목과 이유(항상 1개 이상) |
| `sample_as_of` | 표본 모드일 때 표본 기준 시각, `live`면 빈 문자열 |

`scoring.valuation`은 `mean_reversion.components`와 같은 원천 값이다. `quote.available=false`면 밸류에이션 항목이 빠지고 `mean_reversion.coverage`가 낮아진다.

### 4-5. `GET /api/analyze/{ticker}`

경로 변수를 `q`로 넘겨 4-4와 완전히 같은 응답을 준다. `/api/analyze/TSLA`처럼 슬래시 뒤에 바로 쓰는 용도다.

### 4-6. `GET /api`

```json
{
  "service": "HD2Lab Market Intelligence",
  "slug": "market-intelligence",
  "version": "0.1.0-draft",
  "endpoints": ["/api/health", "/api/factors", "/api/analyze?q=005930", "/api/symbols"],
  "disclaimer": "…"
}
```

## 5. 요청·응답 예시

### 5-1. 상태와 종목 예시

```bash
curl -s http://127.0.0.1:8791/api/health
curl -s http://127.0.0.1:8791/api/symbols
```

### 5-2. 분석

```bash
curl -s 'http://127.0.0.1:8791/api/analyze?q=삼성전자'
curl -s 'http://127.0.0.1:8791/api/analyze?q=005930'
curl -s https://hd2lab.duckdns.org/market-intel/api/analyze/TSLA
```

### 5-3. 오류 응답

```bash
curl -s -i 'http://127.0.0.1:8791/api/analyze?q='
```

```json
{"error": "query_required", "message": "q 파라미터에 종목명 또는 티커를 넣어 주세요.", "examples": ["…"]}
```

```bash
curl -s -i 'http://127.0.0.1:8791/api/analyze?q=없는종목'
```

```json
{"error": "unknown_symbol", "message": "'없는종목' 종목을 찾지 못했습니다. 6자리 코드나 해외 티커로 다시 시도해 주세요.", "suggestions": ["…"]}
```

### 5-4. 프런트 연결

```text
https://hyangdan2.github.io/PKPilot0004/?api=http://127.0.0.1:8791
```

`?api=`는 그 세션에서만 쓰고, 상단 `API 연결`에서 저장하면 `localStorage`에만 남는다(서버로 전송하지 않는다).

## 6. 오류 코드

| HTTP | 본문 | 의미 / 조치 |
| --- | --- | --- |
| 400 | `{"error":"query_required", …}` | `q`가 비었음 → 종목명·코드·티커 입력 |
| 404 | `{"error":"unknown_symbol", …}` | 종목 해석 실패 → `suggestions` 참고 |
| 405 | FastAPI 기본 | `GET`이 아닌 메서드(CORS도 GET/OPTIONS만 허용) |
| 500 | `{"error":"internal_error","message":"<예외 클래스명>"}` | 처리되지 않은 예외. 내부 메시지·스택은 노출하지 않는다 |

정상 분석에서 데이터가 일부 비어도 200이며 `data_gaps`로 알린다. 커넥터 실패는 오류가 아니라 표본 폴백 + `sources[].status`로 표현된다.

## 7. 환경변수·배포

| 변수 | 기본값 | 설명 |
| --- | --- | --- |
| `OPENDART_API_KEY` | 빈 값 | 무료 OpenDART 40자 키. 비면 공시는 표본 폴백 |
| `MI_OFFLINE` | `0` | `1`/`true`/`yes`면 외부 요청 없이 표본만 사용 |
| `MI_ROOT_PATH` | 빈 값 | Nginx 서브패스(예: `/market-intel`). FastAPI `root_path` |
| `MI_CORS_ORIGINS` | Pages + 로컬 포트 목록 | 쉼표 구분 출처. 비우면 기본값 |
| `MI_HTTP_TIMEOUT` | `6` | 외부 무료 API 타임아웃(초) |
| `MI_NEWS_DAYS` | `14` | 뉴스 검색 기간(일) |
| `MI_DISCLOSURE_DAYS` | `30` | 공시 조회 기간(일) |

- 배포 단위: `backend/tools/market-intelligence.service`(systemd), `backend/tools/nginx-market-intelligence.conf`(Nginx).
- 설정 원본은 서버의 `/etc/hd2lab/market-intelligence.env`(0600, 전용 사용자). 이 파일은 Git·이미지·로그에 넣지 않는다.
- 테스트: `cd backend && pytest` (네트워크 미사용, 61개).

## 8. 변경 이력

| 날짜 | 내용 |
| --- | --- |
| 2026-11-19 | `main`(`49d1551`) 소스 기준 최초 작성. 6개 엔드포인트, `analyze` 응답 스키마·`/api/health`·오류 규약 보강(HANDOFF_ANTIGRAVITY.md §5.1 승격) |
