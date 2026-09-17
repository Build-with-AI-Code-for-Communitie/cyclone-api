# cyclone-api

FastAPI backend for the **Cyclone AI Command Center**. Serves risk scores, flood simulation results, and AI-generated action plans to `cyclone-dashboard`. Reads static artifacts produced by `cyclone-data` — does not train or preprocess geospatial data itself.

## What this repo does

- Simulates flood extent for a selected cyclone scenario using the calibrated HAND model from `cyclone-data`
- Scores infrastructure risk (hospitals, shelters, substations, roads) using the trained damage-probability model
- Builds a structured payload from risk + infrastructure data and sends it to Gemini for a prioritized action plan
- Serves everything over a REST API consumed by the dashboard

## Repo structure

```
cyclone-api/
├── app/
│   ├── main.py                  # FastAPI app entrypoint
│   ├── risk_engine.py           # flood extent from calibrated_params.json
│   ├── damage_scorer.py         # loads damage_model.pkl, scores infrastructure
│   ├── gemini_service.py        # builds payload, calls Gemini, parses structured response
│   ├── schemas.py                # Pydantic request/response models
│   └── data_loader.py           # loads static artifacts from cyclone-data/outputs
├── data/                         # copy or symlink of cyclone-data/outputs
├── tests/
├── requirements.txt
├── .env.example                  # GEMINI_API_KEY, DATA_PATH, etc.
└── README.md
```

## Setup

```bash
git clone <repo-url>
cd cyclone-api
python -m venv venv && source venv/bin/activate
pip install -r requirements.txt
cp .env.example .env   # fill in GEMINI_API_KEY
```

Pull in the latest data artifacts from `cyclone-data`:

```bash
# either symlink a local clone of cyclone-data
ln -s ../cyclone-data/outputs data

# or copy the specific release/tag you're pinning to
cp -r /path/to/cyclone-data/outputs ./data
```

## Running locally

```bash
uvicorn app.main:app --reload --port 8000
```

API docs available at `http://localhost:8000/docs` (FastAPI auto-generated Swagger UI).

## Key endpoints

| Endpoint | Method | Description |
|---|---|---|
| `/scenarios` | GET | List available cyclone scenarios (Hudhud, Phailin, custom) |
| `/simulate` | POST | Given a scenario + intensity, return flood extent + infrastructure risk |
| `/action-plan` | POST | Given a simulation result, return Gemini-generated prioritized actions |
| `/infrastructure` | GET | Return infrastructure layer (hospitals, shelters, substations, roads) |
| `/health` | GET | Health check |

## Environment variables

```
GEMINI_API_KEY=       # required for /action-plan
DATA_PATH=./data      # path to cyclone-data outputs
CORS_ORIGINS=http://localhost:5173   # dashboard dev URL
```

## Design notes

- **All numbers are computed in Python, not by Gemini.** The Gemini call receives a JSON payload with surge height, flooded area, affected infrastructure counts, and population exposure already calculated. Gemini's job is prioritization and phrasing, via a forced JSON schema (`zone_id`, `priority`, `action`, `rationale`, `lead_agency`, `deadline_hours`) — never arithmetic.
- **Cache Gemini responses** for the demo scenarios so the live demo doesn't depend on API latency/availability. See `app/gemini_service.py` for the fallback-to-cache logic.

## Consumes

- [`cyclone-data`](../cyclone-data) — static GeoJSON/raster/model artifacts

## Consumed by

- [`cyclone-dashboard`](../cyclone-dashboard) — frontend client

## Disclaimer

Outputs are decision-support estimates, not operational forecasts.
