# KrishiMitra

AI decision support for perishable produce batches (prototype crop: tomato), built as modular services aligned with the GI-AI4AFS Open Reference Architecture.

The prototype runs **locally**: one Python (FastAPI) backend with each component as its own module, plus a React dashboard.

---

## Project structure

```
krishimitra/
├── .vscode/
│   ├── settings.json                      # Python interpreter, formatting
│   ├── launch.json                        # Debug configs: backend + dashboard
│   └── extensions.json                    # Recommended extensions
│
├── backend/                               # FastAPI app, one module per component
│   ├── app/
│   │   ├── __init__.py
│   │   ├── main.py                        # Starts server, CORS, mounts all routers
│   │   ├── config.py                      # Reads .env (API key, paths, demo mode)
│   │   ├── database.py                    # SQLite connection
│   │   ├── events.py                      # In-process event bus: "batch.updated" → re-score
│   │   │
│   │   ├── schemas/                       # Shared data models
│   │   │   ├── __init__.py
│   │   │   ├── batch.py
│   │   │   ├── event.py
│   │   │   ├── quality.py
│   │   │   ├── shelf_life.py
│   │   │   ├── market.py
│   │   │   └── recommendation.py
│   │   │
│   │   ├── utils/
│   │   │   ├── __init__.py
│   │   │   ├── photos.py                  # Save uploads, check type and size
│   │   │   └── time.py                    # IST timestamps, hours since harvest
│   │   │
│   │   ├── modules/
│   │   │   ├── __init__.py
│   │   │   │
│   │   │   ├── quality_vision/            # C1 · photos → grade, ripeness, defects
│   │   │   │   ├── __init__.py
│   │   │   │   ├── router.py              # POST /api/quality/assess
│   │   │   │   ├── detector.py            # Runs ONNX model
│   │   │   │   ├── aggregator.py          # Fruit labels → batch grade + confidence
│   │   │   │   └── grade_table.yaml
│   │   │   │
│   │   │   ├── batch_twin/                # C2 · Batch Digital Twin / QR passport
│   │   │   │   ├── __init__.py
│   │   │   │   ├── router.py              # Batches, events, passport
│   │   │   │   ├── models.py              # SQLite tables
│   │   │   │   ├── state_builder.py       # Events → current state
│   │   │   │   ├── hash_chain.py          # Tamper-evident log
│   │   │   │   └── qr.py                  # QR code generation
│   │   │   │
│   │   │   ├── shelf_life/                # C3 · remaining life + risk
│   │   │   │   ├── __init__.py
│   │   │   │   ├── router.py              # POST /api/shelf-life/estimate
│   │   │   │   ├── kinetic_model.py       # Time-temperature model
│   │   │   │   ├── risk.py                # Life range → risk level
│   │   │   │   └── tomato_params.yaml     # Limits by ripeness, temp, humidity
│   │   │   │
│   │   │   ├── decision_engine/           # C4 · next-best-action
│   │   │   │   ├── __init__.py
│   │   │   │   ├── router.py              # POST /api/decision/recommend
│   │   │   │   ├── options.py             # SELL/STORE/PROCESS/REROUTE/ISOLATE
│   │   │   │   ├── expected_value.py      # Expected value per option
│   │   │   │   ├── safety_rules.py        # Rot / low evidence → ISOLATE
│   │   │   │   ├── explainer.py           # Reasons + "what changed"
│   │   │   │   ├── costs.yaml             # Freight, storage, handling
│   │   │   │   └── locations.yaml         # Links to mandis, stores, processors
│   │   │   │
│   │   │   ├── market_connector/          # C5 · Agmarknet / e-NAM
│   │   │   │   ├── __init__.py
│   │   │   │   ├── router.py              # GET /api/market/prices
│   │   │   │   ├── agmarknet.py           # Live pull before the demo
│   │   │   │   ├── snapshot.py            # Reads frozen_prices.csv
│   │   │   │   └── normaliser.py          # Common price format + freshness
│   │   │   │
│   │   │   ├── field_risk/                # C6 · stub
│   │   │   │   ├── __init__.py
│   │   │   │   └── router.py
│   │   │   │
│   │   │   ├── harvest_window/            # C7 · stub
│   │   │   │   ├── __init__.py
│   │   │   │   └── router.py
│   │   │   │
│   │   │   └── notification/              # C8 · farmer message (shown on screen)
│   │   │       ├── __init__.py
│   │   │       ├── router.py
│   │   │       ├── message_builder.py
│   │   │       └── templates/
│   │   │           ├── hi.yaml
│   │   │           └── en.yaml
│   │   │
│   │   └── gi_ai4afs/                     # Interoperability evidence
│   │       ├── __init__.py
│   │       ├── service_descriptions/
│   │       │   ├── quality_vision.json
│   │       │   ├── batch_twin.json
│   │       │   ├── shelf_life.json
│   │       │   ├── decision_engine.json
│   │       │   └── market_connector.json
│   │       └── openapi_export.py          # Saves each module's API spec
│   │
│   ├── ml_models/
│   │   └── tomato_yolov8n.onnx            # Copied from ml/ after export
│   ├── storage/                           # git-ignored, created at runtime
│   │   └── photos/
│   ├── tests/
│   │   ├── __init__.py
│   │   ├── test_quality_vision.py
│   │   ├── test_shelf_life.py
│   │   ├── test_decision_engine.py
│   │   └── test_demo_flow.py              # Batch → delay → flip
│   ├── pytest.ini
│   └── requirements.txt
│
├── dashboard/                             # React + Vite, runs on localhost:5173
│   ├── index.html
│   ├── public/
│   │   └── logo.svg
│   ├── src/
│   │   ├── main.tsx
│   │   ├── App.tsx
│   │   ├── router.tsx                     # Page navigation
│   │   ├── api/
│   │   │   └── client.ts                  # Calls http://localhost:8000/api
│   │   ├── types/
│   │   │   ├── batch.ts
│   │   │   ├── event.ts
│   │   │   ├── market.ts
│   │   │   └── recommendation.ts
│   │   ├── pages/
│   │   │   ├── BatchList.tsx              # All batches + risk badges
│   │   │   ├── BatchDetail.tsx            # Twin state + timeline
│   │   │   ├── Recommendation.tsx         # Before/after flip
│   │   │   ├── Passport.tsx               # Public QR view
│   │   │   └── operator/
│   │   │       ├── NewBatch.tsx
│   │   │       ├── CapturePhotos.tsx      # Upload or phone camera
│   │   │       └── LogEvent.tsx           # One-tap delay / temperature
│   │   ├── components/
│   │   │   ├── RiskBadge.tsx
│   │   │   ├── ActionCard.tsx
│   │   │   ├── WhatChangedBanner.tsx
│   │   │   ├── EventTimeline.tsx
│   │   │   ├── EvidenceTag.tsx            # Source + timestamp on numbers
│   │   │   └── FarmerMessage.tsx          # Shows the Hindi message
│   │   ├── hooks/
│   │   │   └── useBatchPolling.ts         # Refreshes every 2 s
│   │   └── styles/
│   │       └── index.css                  # Large fonts for projector
│   ├── package.json
│   ├── tsconfig.json
│   ├── tsconfig.node.json
│   └── vite.config.ts
│
├── ml/                                    # Model training (before the hackathon)
│   ├── data/                              # git-ignored
│   │   ├── images/
│   │   │   ├── train/
│   │   │   ├── val/
│   │   │   └── test/
│   │   ├── labels/
│   │   │   ├── train/
│   │   │   ├── val/
│   │   │   └── test/
│   │   └── data.yaml                      # Paths + class names for YOLOv8
│   ├── notebooks/
│   │   ├── 01_train_yolov8.ipynb
│   │   └── 02_evaluate.ipynb
│   ├── export_onnx.py                     # → backend/ml_models/
│   ├── reports/
│   │   └── metrics.md                     # Macro-F1, per-class recall
│   └── requirements.txt
│
├── data/                                  # Demo and reference data
│   ├── frozen_prices.csv                  # Dated Agmarknet snapshot
│   ├── mandis.csv                         # Names, distances, travel times
│   ├── cold_stores.csv
│   ├── processors.csv
│   ├── sample_photos/                     # Backup photos with known grades
│   ├── scenario_delay.json
│   └── scenario_price_drop.json
│
├── scripts/
│   ├── seed_demo.py                       # Creates the demo batch
│   ├── run_scenario.py                    # Fires delay or price-drop event
│   ├── reset_demo.py                      # Wipe DB and re-seed
│   └── fetch_agmarknet.py                 # Builds frozen_prices.csv
│
├── docs/
│   ├── architecture.md                    # GI-AI4AFS mapping
│   ├── model_card.md                      # Model limits and results
│   └── demo_script.md
│
├── demo/
│   └── krishimitra_demo.mp4               # Recorded backup video
│
├── .env.example                           # AGMARKNET_API_KEY, DB_PATH, DEMO_MODE
├── .gitignore                             # .venv, node_modules, __pycache__, storage, ml/data, .env
└── README.md                              # This file
```

**Created automatically (do not add by hand):** `.venv/` (`python -m venv .venv`), `node_modules/` and `package-lock.json` (`npm install`), `backend/storage/krishimitra.db` (first backend run), `__pycache__/`.

---

## Module-to-component map

| Folder | Component | Demo status |
| --- | --- | --- |
| `backend/app/modules/quality_vision` | C1 Quality and Ripeness Vision | Core |
| `backend/app/modules/batch_twin` | C2 Batch Digital Twin / QR passport | Core |
| `backend/app/modules/shelf_life` | C3 Shelf-Life and Risk Engine | Core |
| `backend/app/modules/decision_engine` | C4 Next-Best-Action Engine | Core |
| `backend/app/modules/market_connector` | C5 Market Data Connector | Interface |
| `backend/app/modules/field_risk` | C6 Field Risk | Stub |
| `backend/app/modules/harvest_window` | C7 Harvest Window | Stub |
| `backend/app/modules/notification` + `dashboard/` | C8 Dashboard and farmer messages | Core |
| `backend/app/gi_ai4afs` | Reference-architecture service descriptions | Core |
| `ml/` | Training and export of the vision model | Offline work |
| `data/`, `scripts/` | Demo data and scenario scripts | Support |

---

## Setup (once)

```bash
python -m venv .venv
# Windows: .venv\Scripts\activate   |   macOS/Linux: source .venv/bin/activate
pip install -r backend/requirements.txt
cd dashboard && npm install && cd ..
cp .env.example .env        # then add your data.gov.in API key
```

## Run (two terminals)

```bash
# Terminal 1 – backend  → http://localhost:8000/docs
cd backend
uvicorn app.main:app --reload --port 8000

# Terminal 2 – dashboard → http://localhost:5173
cd dashboard
npm run dev -- --host
```

You can also start both from VS Code's **Run and Debug** panel ("Run backend" and "Run dashboard").

Check: `http://localhost:8000/api/health` should return `{"status": "ok"}`.

## Demo

```bash
python scripts/reset_demo.py
python scripts/seed_demo.py
python scripts/run_scenario.py data/scenario_delay.json
```

---

## Build order

1. `schemas`
2. `batch_twin`
3. `shelf_life`
4. `market_connector` (snapshot first)
5. `decision_engine`
6. Dashboard pages
7. `quality_vision` (once the model is trained)
8. Demo scripts
9. `notification`
10. Stubs (`field_risk`, `harvest_window`)

---

## Production note

The prototype runs as modules inside one backend on a single machine. In production, each module is deployed as a separate container on the GI-AI4AFS Open Reference Architecture, communicating over REST and MQTT and registered in the Service Store. The `gi_ai4afs/service_descriptions/` folder holds the self-description for each service.

## License

Apache-2.0
