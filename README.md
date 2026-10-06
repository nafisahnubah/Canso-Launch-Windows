# Canso Launch Windows

**When can a rocket actually launch from Spaceport Nova Scotia?**

An open launch-window decision engine for Spaceport Nova Scotia (Canso, 45.3° N, 61.0° W). It computes when a launch can reach a target orbit, rejects windows that fail the range-safety screens, and estimates the probability that each window actually launches given the weather. Every number comes with the physics and data sources behind it.

> **Hackathon submission.** This project was built by **Valyrian Astronauts** for the **Mission Accepted Space Hackathon** (MDA Space, the Canadian Space Agency and ShiftKey Labs, Halifax, October 2026), **Challenge 2: Mission Control**.

| | |
|---|---|
| **Live site** | [canso-launch-windows.example.com](https://canso-launch-windows.example.com) *(placeholder, not yet deployed)* |
| **Demo video** | [Google Drive](https://drive.google.com/drive/folders/1mn3HQBs3uSoDM56nq-s1oIFmS-eHEmhs?usp=share_link) |
| **Slides** | [`MDA Mission Accepted Hackathon.pdf`](MDA%20Mission%20Accepted%20Hackathon.pdf) |

---

## The problem

Canso is Canada's first commercial orbital spaceport, but no open orbital analysis of it exists. The only published material is environmental assessments and economic reports. Existing tools each cover one piece of the problem:

- **Window algorithms** (NASA's SLS Launch Window Algorithm, Astos ALWA) compute ascent-aware windows, but they are closed and built for US pads.
- **Mission design suites** (GMAT, STK, Orekit) propagate orbits but give no weather, no launch odds and no web API.
- **Launch weather tools** (NASA APRA / PACER) give climatological go-probabilities by month and hour, with no forecasts, no link to the orbit, and no public access.

No open tool combines orbit geometry, range safety and forecast weather into a single go/no-go probability for any Canadian site.

## What it does

The engine works as a funnel. Dates go in, geometry admits some of them, the screens remove more, and probabilities rank what is left.

| Step | Question | Physics |
|---|---|---|
| **1. Reachability** | Can a southbound launch from 45.3° N reach this inclination at all? | `cos i = cos φ · sin β`, with the launch azimuth β constrained to a corridor of [115°, 195°] over the ocean. If no azimuth works, the plane-change cost is `Δv = 2 v_c sin(Δi/2)` |
| **2. Window search** | When does Earth's rotation carry the pad into the orbit plane? | `GMST(t) + λ = Ω(t) + asin(tan φ / tan i)`, with the J2 nodal drift `dΩ/dt = −(3/2) J2 n (R/a)² cos i` computed from physical constants, not hard-coded |
| **3. Ascent correction** | The climb to orbit takes about 9 minutes, so when must liftoff happen? | A fixed point `t = Φ(t)`. The map is a contraction (`|Φ′| ≤ 0.017`), so by the Banach theorem it always converges |
| **4. Safety screens** | Is anything forcing a hold? | Hazard corridor (a debris ellipse about 105 km wide), a conjunction pre-screen and airspace notices |
| **5. Launch odds** | How likely is this window to actually go? | `p_success = P_weather · P_range · P_conj`, where `P_weather` is the fraction of ensemble forecast members that pass every launch commit criterion. Beyond the skill horizon, climatology is used and labelled as such |

When the planner asks for an orbit the site cannot reach, the engine does not invent a window. It returns `reachable: false` together with the cost of fixing it.

## Results

- **Validated against real launches.** For four published launches from three spaceports, the window centres fall within 2.3 minutes of the actual liftoff times: Sentinel-1C (−1.5 min), EarthCARE (−0.5 min), Sentinel-5P (+0.9 min) and Sentinel-3C (+2.3 min). Each was checked against a 5-minute gate. Cases that do not reproduce are listed with their reasons in [`backend/engine/README.md`](backend/engine/README.md).
- **The advertised 45.1° LEO orbit is unreachable** by direct ascent from a 45.3° N pad: `sin β = cos 45.1° / cos 45.3° = 1.0035 > 1`, so no launch azimuth exists. The engine reports this together with the 26.8 m/s plane change needed to fix it.
- **Launch headings from Canso:** 177.0° for polar (87.9°) and 191.6° for sun-synchronous (98.1°), both south over open ocean.
- **The weather forecast has measurable skill.** A hindcast over 179 days (April to September 2026, verified against ERA5) gives a Brier skill score of 0.46 at a one-day lead, and the score stays above climatology out to day 5. See [`backend/weather/HINDCAST.md`](backend/weather/HINDCAST.md).
- **Every response is reproducible.** It echoes its constants (J2, GM, R_e), the data sources used and a citation id that re-fetches the exact run.

### What we do not claim

Claims follow the status labels of spec Part II.9. The fixed-point convergence and reachability are **proved**. The chance-constrained window is **sketched**. Forecast skill at Canso remains a **conjecture**, measured over one six-month period. We do not claim ascent-aware windows or probabilistic launch weather as new ideas: NASA's SLS algorithm and APRA/PACER came first, and we cite them. The launch commit criteria are a documented proxy set, not flight-safety grade, and the conjunction screen is a pre-screen only.

## The interface

- **Globe view** ([`Canso Launch Prototype.html`](Canso%20Launch%20Prototype.html)). A 3D Earth with real sun lighting.
  - **Public mode:** a countdown to the next launch opportunity in Halifax time and UTC, an animated "Why this time?" explanation, the orbit drawn as a hoop you can hover for details, the ascent arc, and the towns where the rocket rises above the horizon.
  - **Planner mode:** the full window table, a recommended window, the expected delay and its cost (from a daily cost you enter), guided examples and a time scrubber.
- **Analysis app** ([`frontend/`](frontend/)). The researcher layer: window table with CSV and JSON download, provenance and citation for each run, skill and calibration charts, and per-criterion weather shares.

Both pages read the live API and fall back to the frozen fixtures in `backend/fixtures/`, with an offline banner, if the API is unavailable.

## Run it locally

Requires Python 3.11 or newer. The frontend tests need Node 22.12 or newer.

```bash
python -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\activate
pip install -e ".[dev]"

# 1. The API, on port 8000
python -m uvicorn backend.api.app:app --port 8000

# 2. In a second terminal, from the repo root: serve the pages on port 5500
python -m http.server 5500
```

Then open:

- Globe view: <http://localhost:5500/Canso%20Launch%20Prototype.html>
- Analysis app: <http://localhost:5500/frontend/>
- API: <http://localhost:8000/v1/site>

The pages must be served over HTTP, because a `file://` page cannot load ES modules.

### Tests

```bash
python -m pytest -q                  # contract, engine, weather and API suites
python scripts/integration_test.py   # end-to-end harness over the whole /v1 surface
cd frontend && npm ci && npx vitest run
```

## API

A versioned REST API (FastAPI) under `/v1`. Read endpoints need no authentication.

| Method | Endpoint | Returns |
|---|---|---|
| `POST` | `/v1/windows` | Launch windows for a target orbit, with reachability, screens and `p_success` |
| `GET` | `/v1/orbits/{id}/ephemeris` | The ascent and orbit track of a window |
| `GET` | `/v1/site` | Site geometry, corridor and operating hours, with provenance flags |
| `GET` | `/v1/weather/probability` | Weather go-probability for a date, labelled FORECAST or CLIMATOLOGY |
| `GET` | `/v1/validation/skill` | Hindcast Brier skill by lead time, reliability and ROC |
| `GET` | `/v1/citation` | The full record of a past run, for reproduction |
| `GET` | `/v1/decision/delay-cost` | Expected days until launch and the delay cost, from a series of daily probabilities |

There is also a Python client for researchers:

```python
import launchwin

frame = launchwin.windows(target="SSO", site="canso", dates=("2026-10-05", "2026-10-15"))
```

## Repository layout

```
backend/
  engine/     orbital mechanics: reachability, J2, window search, injection fixed point, screens
  weather/    ensemble probability, climatology, launch commit criteria, hindcast
  api/        FastAPI service, provenance and citation store
  client/     the launchwin Python client
  fixtures/   frozen offline responses (the demo floor)
frontend/     analysis app (vanilla JS, Leaflet, Vitest)
tests/contract/   frozen JSON Schemas for every response
scripts/      integration harness and physics checks
docs/         build spec, integration contract, workflow logs, physics derivations
Canso Launch Prototype.html   the globe view
```

The full scientific specification is in [`docs/spec/C2_framework_and_build_spec.md`](docs/spec/C2_framework_and_build_spec.md), and each workflow documents its own scope and limitations in its README.

## Data and acknowledgements

Site and vehicle data come from the Spaceport Nova Scotia environmental assessment, CARs 602.43 and 602.44, and the Cyclone-4M Abbreviated User's Guide. Every value is flagged VERIFIED, DERIVED or ASSUMPTION. Weather data comes from Open-Meteo (GFS and ECMWF ensemble forecasts), ECMWF ERA5 reanalysis, and Environment and Climate Change Canada station observations. The globe uses NASA Blue Marble and Black Marble imagery.

Built in 32 hours for the Mission Accepted Space Hackathon. Thanks to MDA Space, the Canadian Space Agency and ShiftKey Labs.
