<h1 align="center">APEX — Formula 1 ML Race Predictor</h1>
<p align="center"><strong>Compare ML models on a custom Formula 1 grid.</strong></p>
<p align="center">
  <img alt="Python 3.11" src="https://img.shields.io/badge/Python-3.11-3776AB?logo=python&logoColor=white">
  <img alt="scikit-learn 1.7.1" src="https://img.shields.io/badge/scikit--learn-1.7.1-F7931E?logo=scikitlearn&logoColor=white">
  <img alt="FastAPI" src="https://img.shields.io/badge/API-FastAPI-D2F03C?logo=fastapi&logoColor=141516">
  <img alt="React 18" src="https://img.shields.io/badge/React-18-61DAFB?logo=react&logoColor=black">
  <img alt="Three.js" src="https://img.shields.io/badge/3D-Three.js-141516?logo=threedotjs&logoColor=D2F03C">
</p>

![APEX running a real 22-driver ML prediction at Spa](docs/assets/screenshots/prediction-results.webp)

<p align="center"><a href="#quick-start--windows-powershell">Run locally</a> · <a href="#inside-apex">Explore the app</a> · <a href="#how-the-ml-works">How it predicts</a> · <a href="#dataset-and-model-results">Results</a> · <a href="#coursework-deliverables">Coursework</a> · <a href="CREDITS.md">Credits</a></p>

APEX is an **educational machine-learning race simulator**. Pick historical F1 drivers, assign constructors at their automatically selected peak, choose a classifier and regressor, configure a circuit and weather, then watch a 3D race end in the models’ predicted order.

**208 eligible drivers · 65 constructor identities · 537 historical car-season profiles · 22 grid positions · 11 teams, two seats each.** Historical data ends in 2025. This is an independent coursework project, not an official F1 forecasting service.

## Inside APEX

Every image below comes from the running application at **1600 × 960**, using real model inference and local assets.

| A circuit-first control room | Your all-time starting grid |
|---|---|
| ![Dashboard with the Spa 3D circuit](docs/assets/screenshots/dashboard.webp) | ![Three-column lineup editor with positions P1–P22](docs/assets/screenshots/lineup-editor.webp) |
| True-scale terrain, circuit selection and two model pickers. | Portraits, performance years, team marks, drag-and-drop and Undo/Redo. |

| Search through F1 history | Give a driver a different car |
|---|---|
| ![Historical driver selection with Ayrton Senna](docs/assets/screenshots/driver-selection.webp) | ![Assigning peak Aston Martin cars to two grid seats](docs/assets/screenshots/constructor-selection.webp) |
| Full-name A–Z lists and accent-insensitive search. | One constructor entry, automatic peak season, two-driver capacity. |

| Set the conditions | Follow the running order |
|---|---|
| ![Rain, air temperature, humidity and wind controls](docs/assets/screenshots/environment.webp) | ![Paused 22-car race and live timing tower](docs/assets/screenshots/race-overview.webp) |
| Dry / Damp / Wet presets plus supported weather sliders. | Five-light start, pause/resume and approximately ten-second playback. |

| A closer look at the car | The cockpit perspective |
|---|---|
| ![Detailed GLB car in Follow camera](docs/assets/screenshots/race-follow.webp) | ![Onboard camera during the model-predicted race](docs/assets/screenshots/race-onboard.webp) |
| Local GLB geometry, constructor-colour patterns and rolling wheels. | Follow, Onboard, Overview, Top and bounded manual camera controls. |

## Features

- **An all-time field:** 208 eligible drivers through 2025, each in their validated form season; 65 distinct historical constructors, including defunct teams.
- **A coherent peak car:** era-relative qualifying/grid performance, race finishes and reliability select one real eligible season per constructor. Driver skill and car performance stay separate.
- **A valid full grid:** exactly 22 unique drivers and 11 constructors with two seats each. Move/Swap, click or drag assignment, search, Undo/Redo and restore default.
- **Two ML perspectives:** choose from 10 classifiers and 6 regressors. Circuit history and supported pre-start weather features can change the result.
- **Interactive 3D:** five locally bundled GLB bodies, 42 mapped circuits, measured terrain where quality permits, start lights, timing tower and podium/full standings.

Historical cars use simplified era approximations; 2010–2017 shares the disclosed 2000s body. The detailed ground-effect base covers 2022–2025. Car colour patterns are approximate, not exact constructor chassis or sponsor liveries.

## How the ML works

| Stage | What it contributes |
|---|---|
| Classification | Scores each driver’s likelihood of a podium finish. |
| Regression | Scores relative finishing performance. |
| Race sampling | Calibrated scores drive 5,000 seeded simulations per selected model, using shared random numbers. |
| Final result | Average the two models’ expected finishing positions, then rank the complete field and show P1–P22. |

Different models learn different relationships, so their expected positions and podium choices can differ. Driver inputs come from each driver’s validated eligible form season; constructor inputs come from **one complete real peak-season profile**, never a fictional combination of the best metrics from different cars.

Peak selection weights relative qualifying-grid percentile **40%**, race finishing percentile **40%**, and DNF share against that season’s field **20%**. Eligible seasons with at least eight races are preferred; thinner constructors remain available with an internal evidence flag. [Methodology and decisions →](docs/DECISIONS_LOG.md)

**The finishing order is the model output. Overtakes and intermediate race motion are illustrative**, not predicted telemetry or lap-by-lap events. Cross-era matchups cannot be validated against an actual race. Model training/evaluation uses chronological splits; some reported simulator holdout results are explicitly post-hoc. [Research and limitations →](docs/LITERATURE_REVIEW.md)

## Quick start — Windows PowerShell

**Prerequisites:** Python **3.11**, Node.js **20+** with npm, and Git. Browser E2E checks use installed Microsoft Edge; Chrome can be selected where supported.

```powershell
git clone https://github.com/dhruvagarwal4543/Apex-f1.git
Set-Location Apex-f1

py -3.11 -m venv .venv
& .\.venv\Scripts\python.exe -m pip install -r requirements-lock.txt

Push-Location frontend
npm ci
npm run build
Pop-Location

# Models, frozen datasets, circuit geometry and assets are already included.
# Verify them; no API key, download or retraining is needed.
& .\.venv\Scripts\python.exe scripts/verify_bundle.py

& .\.venv\Scripts\python.exe -m uvicorn src.api:app --host 127.0.0.1 --port 8012
```

Open **[http://127.0.0.1:8012](http://127.0.0.1:8012)**. If the port is occupied, choose a free one with `--port`; frontend and API share the same origin. Using the environment’s Python directly avoids PowerShell activation-policy changes.

**Try it:** Edit Lineup → assign two seats per team → choose both models → choose a circuit and conditions → Predict Race → inspect the podium and full standings. Change the scenario and predict again.

<details>
<summary>Frontend development and scientific reproduction</summary>

For hot reload, keep the backend running and use another PowerShell terminal:

```powershell
Set-Location frontend
$env:API_PORT = '8012'
npm run dev
```

Use the URL Vite prints. Production builds are served directly by FastAPI.

The original data acquisition, policy, feature-engineering, training, calibration and evaluation scripts remain in `src/`. Required frozen tables and model bundles are committed so the showcase can run offline without re-collecting years of data. [Reproduction guide →](docs/REPRODUCIBILITY.md)

</details>

## Dataset and model results

The bundled Jolpica F1 and Open-Meteo data cover 1950–2025. The collected history contains **25,873 entries across 1,149 races**. Eligibility filters leave **25,066 modelling rows across 1,138 races**. Historical features use race-by-race chronology. Weather is retrospective pre-start reanalysis.

| Evaluation | System | Winner top-1 ↑ | Winner log loss ↓ | Podium overlap ↑ | Rank MAE ↓ |
|---|---|---:|---:|---:|---:|
| First frozen test, 70 races in 2023–2025 | Grid Plackett–Luce | 58.6% | 2.335 | 69.0% | 3.284 |
| First frozen test, 70 races in 2023–2025 | Legacy historical system | 64.3% | 1.106 | 66.2% | 3.099 |
| Chronological replay, same previously examined races | Calibrated grid baseline | 58.6% | 1.376 | 69.0% | 3.284 |
| Chronological replay, same previously examined races | APEX default: Random Forest + Ridge | 60.0% | 1.122 | 69.0% | 3.211 |
| Chronological replay, same previously examined races | APEX showcase: Logistic Regression + Ridge | 60.0% | 1.117 | 69.5% | 3.159 |

Compare rows within the same evaluation. The APEX replays fixed the model settings and refitted through 2022. The project had already examined the 2023–2025 outcomes. The released pair has no independent accuracy score on new races, and cross-era matchups have no observed results.

[Frozen test](outputs/tables/final_test_2023_2025_run1_frozen.csv) · [Full replay and component results](outputs/tables/live_pair_replay.csv) · [Replay settings and model hashes](outputs/tables/live_pair_replay.json)

The site shows both **predicted P1 by average finish** and **highest win probability**. Expand **Prediction context** for evidence limitations. Of 77 selectable circuits, 42 have 3D layouts and 35 show standings directly. Fuji, Jerez and Yeongam use attributed mapped outlines at flat elevation.

## Coursework deliverables

- [Word report](report/final/F1_Machine_Learning_Project_Report.docx) · [Report PDF](report/final/F1_Machine_Learning_Project_Report.pdf)
- [Editable PowerPoint](reports/F1_ML_Project_Presentation.pptx) · [Slides PDF](reports/F1_ML_Project_Presentation.pdf)
- [Verification record](report/final/REPORT_VERIFICATION.md)

The contribution assignments are proposed. The declaration awaits the candidates’ signatures and signing date.

## Project structure

```text
F1-Race-Prediction/
├── src/                  API, features, training, calibration and prediction
├── frontend/
│   ├── src/              React UI and Three.js race renderer
│   ├── public/           Circuit geometry, cars, portraits, logos and credits
│   └── e2e/              Unit, browser, asset and GPU checks
├── data/                 Frozen historical tables and derived features
├── models/               Required trained bundles and exact metadata
├── tests/                Python data, leakage, model and API checks
├── scripts/              Artifact integrity and screenshot optimization
├── docs/                 Decisions, reproduction, verification and screenshots
├── notebooks/            Original exploration and EDA
└── outputs/tables/       Scientific comparison and audit results
```

Local environments, dependencies, raw download caches, old browser captures, logs, and unused rollback archives are excluded from Git. Required model files are committed directly; **Git LFS is not required**. [Publication exclusions and cleanup manifest →](docs/CLEANUP.md)

## Testing

From the repository root, with the application running on port 8012 for browser checks:

```powershell
& .\.venv\Scripts\python.exe -m pytest -m 'not live' -q

Push-Location frontend
npm run test                 # frontend unit, geometry, motion and asset checks
$env:E2E_URL = 'http://127.0.0.1:8012'
npm run e2e                  # unit tests + complete browser journey
npm run e2e:assets           # all five GLB bodies and full-grid rendering
npm run e2e:logos
npm run e2e:portraits
npm run e2e:gpu              # optional visible-browser hardware performance check
Pop-Location
```

The live collection test is excluded from offline checks. GPU frame rates depend on the machine. [Actual publication verification and clean-copy results →](docs/VERIFICATION.md)

To refresh the README’s real screenshots, run `npm run showcase` from `frontend` with `E2E_URL` set, then run `python scripts/optimize_screenshots.py` from the root. Only optimized WebP images are published.

## Credits and scope

Historical race data: [Jolpica-F1 / Ergast](https://github.com/jolpica/jolpica-f1), **CC BY-NC-SA 4.0**. Weather data by [Open-Meteo.com](https://open-meteo.com/), **CC BY 4.0**. Circuits: [Tomislav Bacinger](https://github.com/bacinger/f1-circuits), MIT; elevation: Copernicus DEM. Cars, driver photographs and constructor marks retain their individual licences and attribution.

[Full data and asset credits](CREDITS.md) · [Per-image credits](frontend/public/credits.html) · [3D model credits](frontend/public/models/ATTRIBUTION.html)

Independent educational ML simulation. No affiliation with Formula 1, the FIA, constructors or drivers. No actual race outcome or overtaking telemetry is promised.
