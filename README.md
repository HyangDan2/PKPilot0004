# Market Intelligence — PKPilot0004

An HD2Lab dashboard that tags the “ten non-fundamental drivers of share-price movement” using **free public
data only**. It keeps short-term overheating (Non-Fundamental Drift Score) and mean-reversion risk as **two
separate scores**, and states on screen which numbers are not live data (sample).

- Live: `https://hyangdan2.github.io/PKPilot0004/`
- Status: **draft (DRAFT v0.1)** · live static front end on GitHub Pages · `noindex`
- Handover spec: [HANDOFF_ANTIGRAVITY.md](HANDOFF_ANTIGRAVITY.md)
- Backend draft: [`backend/`](backend/) → Server-HQ porting target

## Two modes

| Mode | Condition | Data | Scoring | Network |
| --- | --- | --- | --- | --- |
| **Static demo** (default) | open it as-is | built-in sample of four stocks (Samsung Electronics, SK hynix, Ecopro, NAVER) | in the browser | none |
| **API mode** | header `API connection` → base URL | the server's response | on the server | `/api/analyze` and friends |

The demo pins its clock to `2026-11-13 15:30 KST` so it returns the **same result whenever you open it**, and it
uses the same scoring rules as the backend (all four stocks match on NFDS, MR and signal count). The API address
is stored only in `localStorage` and is never sent anywhere else. The response's own `mode` (`live`, `mixed` or
`sample`) drives the mode pill and the “sample data included” badges, so a partly sampled answer says so.

## Layout

| File | Contents |
| --- | --- |
| `index.html` | header (Web-HQ lockup) · hero · mechanism diagram · stock analyser · ten-driver table · investor principles · API settings sheet |
| `style.css` | Soft Iris light (`#FAFAFD` / `#FFFFFF` / `#8B6CE8`) + Navy & Lavender dark (`#0B1020` / `#12192E` / `#A78BFA`) tokens, responsive from 320 to 1440px, zero dependencies (`./assets/hd2-theme.*` toggle) |
| `app.js` | domain copy · sample data · demo scoring engine · API client · renderers (external strings go through `textContent` only) |
| `manifest.json`, `favicon.*`, `icon-*.png` | PWA and icon set (kept to avoid 404s) |
| `assets/hd2-theme.*` | shared theme tokens and the 44px floating light/dark toggle |
| `backend/` | FastAPI draft (free-API pipeline + 61 pytest cases) |

## API

The backend contract in `backend/` lives in **[API reference (docs/API.md)](docs/API.md)**. It promotes the
table in §5.1 of the handover document [HANDOFF_ANTIGRAVITY.md](HANDOFF_ANTIGRAVITY.md) to the canonical place
in the repository and adds request and response schemas.

| Method | Path | Purpose |
| --- | --- | --- |
| GET | `/api/health` | service and connector status, domain consistency, CORS origins |
| GET | `/api/factors` | four axes · ten drivers · mechanism · investor principles |
| GET | `/api/symbols` | ten example stocks |
| GET | `/api/analyze?q=<name\|code\|ticker>` | one stock's analysis (NFDS, MR, signals, data gaps) |
| GET | `/api/analyze/{ticker}` | the same analysis (path version) |
| GET | `/api` | service summary and endpoint list |

- No authentication (everything is a public read `GET`). CORS allows only the `MI_CORS_ORIGINS` list.
- Errors: `400 query_required`, `404 unknown_symbol`, `500 internal_error`

```sh
# front end (static)
python3 -m http.server 8804 --bind 127.0.0.1
# → http://127.0.0.1:8804/

# backend (sample mode)
cd backend
python3 -m venv .venv && .venv/bin/pip install -r requirements.txt
MI_OFFLINE=1 .venv/bin/python -m uvicorn market_intel.app:create_app --factory --host 127.0.0.1 --port 8791

# front end → backend (allow the static origin in CORS)
MI_CORS_ORIGINS="http://127.0.0.1:8804,http://127.0.0.1:8791" \
  .venv/bin/python -m uvicorn market_intel.app:create_app --factory --host 127.0.0.1 --port 8791
# → http://127.0.0.1:8804/?api=http://127.0.0.1:8791
```

`?api=` is used for that session only. Saving it with `API connection` in the header connects automatically from
the next visit.

## Verification

| Item | Result |
| --- | --- |
| Backend tests | **61 passed** (`pytest`, no network) |
| Demo ↔ backend parity | four stocks identical on NFDS, MR, signals and axis scores |
| Screen (1440 / 390 / 320) | no horizontal overflow, no console errors |
| API mode measured live | real free sources → `mode: mixed`, NFDS 34.3 (005930) |

The numbers and the way to reproduce them are in §3.3–3.5 of [HANDOFF_ANTIGRAVITY.md](HANDOFF_ANTIGRAVITY.md).

## Known limits

- The demo sample holds four stocks and is not live (its clock is fixed).
- Overseas tickers can come back with empty signals because of Yahoo rate limits (429) and the missing Korean
  keyword dictionary.
- Short-selling balance has no free real-time source, so it is estimated from a supply/demand reversal instead.
- The score coefficients are pre-backtest hypotheses.

This is not investment advice, and every response carries a disclaimer.
