# Scheme Assistant — Project Documentation

> **AI-Powered Financial Scheme Guidance** · Developed by **Team Nexora**
>
> A full-stack web platform that matches users (with a focus on Scheduled Caste beneficiaries in India) to government-backed financial schemes, ranks matches with a trained ML model, routes users to nearby channel partners, and calculates repayment plans.

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Tech Stack](#2-tech-stack)
3. [Repository Structure](#3-repository-structure)
4. [Backend Architecture](#4-backend-architecture)
5. [Database Schema](#5-database-schema)
6. [ML / AI Pipeline](#6-ml--ai-pipeline)
7. [API Reference](#7-api-reference)
8. [Frontend Architecture](#8-frontend-architecture)
9. [Data Flow (End-to-End)](#9-data-flow-end-to-end)
10. [Setup & Running Locally](#10-setup--running-locally)
11. [Known Issues & Gaps](#11-known-issues--gaps)
12. [Future Changes To Be Implemented (Roadmap)](#12-future-changes-to-be-implemented-roadmap)

---

## 1. Project Overview

### The Problem

Government welfare/funding schemes (NSFDC-style schemes for Scheduled Castes) are hard to discover, hard to compare, and hard to apply for. Users don't know which scheme they qualify for, where to apply nearby, or what their repayment will look like.

### The Solution

A 3-step guided journey:

| Step | What happens | Where |
|------|--------------|-------|
| **1. Eligibility check** | User enters assistance type (Business / Education), project/course cost, loan required, family income, location | `/eligibility` |
| **2. AI matching** | Hard eligibility rules filter schemes, then a Random-Forest classifier scores each eligible scheme (0–100 % match). Nearby channel partners are scored by distance, NPA % and fund-utilization | `POST /recommend` |
| **3. Next step** | Repayment calculator (EMI, total interest, "pay early and save" simulation) or partner locator on a map | Calculator + `/partners` |

### Schemes Currently Supported

| Code | Name (inferred) | Category | Notes |
|------|-----------------|----------|-------|
| `MFS` | Micro Finance Scheme | Business | Small loans (~₹1.4 L project cap) |
| `AMY` | (Mahila/Adim Yojana-type scheme) | Business | Micro loans via NBFC-MFI channel |
| `UNY` | Unemployed Youth scheme | Business | Mid-size loans; **channel-specific interest rates** via `scheme_interest_rates` table |
| `TL` | Term Loan | Business | Large loans up to ₹50 L project cost |
| `ELS` | Education Loan Scheme | Education | Course-fee coverage % logic, up to ₹40 L |

> ⚠️ `ELS` and `UNY` are **hard-coded by code** in the eligibility service and calculator route respectively. See [Known Issues](#11-known-issues--gaps).

---

## 2. Tech Stack

### Backend (`backend/`)

| Concern | Technology |
|---------|-----------|
| Framework | **FastAPI** (0.115) + Uvicorn |
| Database | **PostgreSQL** via `DATABASE_URL` in `.env` |
| ORM | **SQLAlchemy 2.0** (declarative, no Alembic yet) |
| ML | **scikit-learn** `RandomForestClassifier` inside a `Pipeline` (ColumnTransformer + OneHotEncoder + SimpleImputer), persisted with **joblib** → `scheme_ranker.pkl` |
| Data handling | pandas, joblib |
| Geocoding | `pgeocode` + Nominatim (partner lat/long backfill script) |
| Data import | pandas + openpyxl (Excel partner seed list) |
| Python | 3.14 (venv in `backend/venv`) |

### Frontend (`frontend/`)

| Concern | Technology |
|---------|-----------|
| Framework | **React 19** + **Vite 8** |
| Routing | react-router-dom v7 |
| Maps | **Leaflet 1.9** + react-leaflet v5 (OpenStreetMap tiles) |
| i18n | i18next / react-i18next installed (**not yet wired up**) |
| Styling | Plain CSS per component (no framework) |

---

## 3. Repository Structure

```
.
├── PROJECT.md                  ← this file
├── backend/
│   ├── app.py                  ← FastAPI app, CORS, router mounting
│   ├── database.py             ← engine, SessionLocal, Base, create_all
│   ├── .env                    ← DATABASE_URL (⚠️ contains secrets)
│   ├── scheme_ranker.pkl       ← trained RandomForest pipeline
│   ├── models/
│   │   ├── scheme.py           ← Scheme ORM model
│   │   ├── partner.py          ← Partner ORM model
│   │   └── scheme_partner_type.py ← Scheme↔partner-type mapping
│   ├── routes/
│   │   ├── recommendation.py   ← POST /recommend (eligibility + AI + partners)
│   │   └── calculator.py       ← POST /calculate-loan (EMI plans)
│   ├── services/
│   │   ├── eligibility.py      ← hard/fail-fast eligibility rules
│   │   ├── ai_ranker.py        ← loads scheme_ranker.pkl, rank_scheme()
│   │   ├── partner_router.py   ← Haversine distance + partner scoring
│   │   ├── calculator.py       ← EMI math (reducing balance), plan comparison
│   │   ├── recommendation_engine.py  ← (legacy, unused) rule-based engine
│   │   └── scheme_scorer.py    ← (legacy, unused) handcrafted 100-pt scorer
│   ├── train_model.py          ← trains RandomForest, saves .pkl
│   ├── generate_training_data.py           ← small rule-labelled dataset
│   ├── generate_ai_training_data.py        ← dataset from curated training_cases
│   ├── generate_large_ai_dataset.py        ← 10k synthetic applicants
│   ├── training_cases.py       ← curated (applicant → preferred scheme) tuples
│   ├── ai_training_data*.csv   ← generated datasets
│   ├── import_partners.py      ← Excel → partners table
│   ├── geocode_partners.py     ← pincode → lat/long backfill
│   └── test_*.py               ← manual smoke scripts (not pytest)
└── frontend/
    ├── index.html
    ├── vite.config.js
    ├── package.json
    └── src/
        ├── main.jsx            ← React root
        ├── App.jsx             ← routes + Home landing page
        ├── services/api.js     ← fetch wrapper → FastAPI
        └── components/
            ├── Navbar.jsx
            ├── EligibilityForm.jsx      ← Step 1 form (location-aware)
            ├── RecommendationResults.jsx← Step 2 results (610 lines)
            ├── SchemeDetails.jsx        ← scheme detail view
            ├── RepaymentCalculator.jsx  ← Step 3 EMI calculator
            ├── PartnerLocator.jsx       ← /partners route
            └── PartnerMap.jsx           ← Leaflet map with markers
```

---

## 4. Backend Architecture

### Layering

```
routes/ (HTTP, FastAPI)
   ↓
services/ (pure business logic, no HTTP)
   ↓
models/ (SQLAlchemy ORM) ← database.py (engine/session)
```

### `app.py`
- Creates the FastAPI app.
- CORS middleware allowing only `localhost:5173` / `127.0.0.1:5173` (Vite dev server).
- Mounts `routes.recommendation` and `routes.calculator`.

### `database.py`
- Loads `.env`, creates the engine from `DATABASE_URL`.
- `SessionLocal` sessionmaker; `Base.metadata.create_all()` on import (⚠️ runs for every CLI script too).
- Prints the connected database name on startup (used as a health signal).

### Services

| Service | Responsibility |
|---------|----------------|
| `eligibility.py` → `is_scheme_eligible()` | Fail-fast hard rules: income limit, education vs non-education category, project cost min/max, loan ≤ max loan, ELS course-fee coverage % cap. Returns boolean. |
| `ai_ranker.py` → `rank_scheme()` | Builds a 13-feature pandas row (user inputs + scheme attributes), calls the trained pipeline's `predict_proba`, returns the "preferred" probability. Loads `scheme_ranker.pkl` **once at import time**. |
| `partner_router.py` | `calculate_distance()` — Haversine km; `calculate_partner_score()` — `max(0, 40 − km) + max(0, 30 − 2·NPA%) + 0.30·fundUtil%`; `get_compatible_partners()` — active partners whose `partner_type` is allowed for the scheme via `scheme_partner_types`. |
| `calculator.py` | `calculate_emi()` — standard reducing-balance EMI; `calculate_loan_plan()` — normal plan + optional shorter "prepay" plan with interest saved. |
| `recommendation_engine.py`, `scheme_scorer.py` | **Legacy/superseded** — duplicate eligibility/scoring logic, only referenced by manual test scripts. Candidates for deletion. |

### Special-case logic to know about

1. **ELS (education)** — eligible loan = `min(max_loan_amount, course_fee × coverage%)`.
2. **UNY (channel rates)** — `POST /calculate-loan` looks up `scheme_interest_rates` (raw SQL) joined by scheme code and a *mapped channel name* (e.g. `Cooperative Bank` → `Co-operative Banks / Societies`). This mapping is hard-coded in `routes/calculator.py`.

---

## 5. Database Schema

```
schemes
├── id (PK)
├── name, code (unique), category, status ("active" | "verify")
├── min_project_cost, max_project_cost, max_loan_amount
├── interest_rate, loan_coverage_percent
├── min_repayment_years, max_repayment_years, moratorium_months
├── income_limit
├── description, eligibility (text)
└── source_url, last_verified

partners
├── id (PK)
├── name, partner_type        ← matched against scheme_partner_types
├── state, district, city
├── address, pincode, phone
├── latitude, longitude       ← backfilled by geocode_partners.py
├── npa_percent, fund_utilization_percent   ← used in partner score
└── official_source, last_verified, verification_status, active

scheme_partner_types
├── id (PK)
├── scheme_id  ← ⚠️ INTEGER with NO foreign key constraint to schemes.id
└── partner_type (string)     ← ⚠️ no FK to a partner_types lookup table

scheme_interest_rates         ← ⚠️ raw-SQL only, no ORM model
├── scheme_id → schemes.id
├── channel (text)
└── interest_rate
```

---

## 6. ML / AI Pipeline

### Data generation (synthetic, prototype-grade)

1. **`training_cases.py`** — ~25 curated applicants, each tagged with a preferred scheme + channel.
2. **`generate_ai_training_data.py`** — for each case × each scheme: keep only eligible ones, label `preferred = (scheme == preferred)`. → `ai_training_data.csv`.
3. **`generate_large_ai_dataset.py`** — 10,000 synthetic applicants with per-scheme cost/income distributions (seeded, seed=42), `applicant_id` groups, partner channel jitter, feature noise. → `ai_training_data_large.csv`.

### Training (`train_model.py`)

- **Features (13):** `project_type`, `project_cost`, `family_income`, `loan_required`, `course_fee`, `partner_channel`, `scheme_code`, `scheme_category`, `scheme_max_project_cost`, `scheme_max_loan`, `scheme_interest_rate`, `scheme_repayment_years`, `scheme_coverage_percent`
- **Target:** `preferred` (binary)
- **Preprocessing:** numeric → constant-0 impute; categorical → most-frequent impute + one-hot (`handle_unknown="ignore"`)
- **Model:** `RandomForestClassifier(n_estimators=200, max_depth=12, class_weight="balanced", n_jobs=-1, random_state=42)`
- **Split:** `GroupShuffleSplit(test_size=0.2, groups=applicant_id)` — keeps one applicant's rows (candidate schemes) on the same side of the split.
- **Output:** `scheme_ranker.pkl` (full sklearn Pipeline). Accuracy + classification report printed to console only (not persisted).

### Inference

`services/ai_ranker.py` loads the pickle at import time; each `/recommend` call scores every eligible scheme and the probability is surfaced as `ai_match_score` (0–100).

> **Reality check:** labels are generated from the *same* eligibility rules the app enforces, so the model largely re-learns those rules plus channel/cost preferences baked into the synthetic distributions. Improving label quality (real applications, feedback loops) is the highest-value ML roadmap item — see §12.

---

## 7. API Reference

Base URL: `http://127.0.0.1:8000` (all params are **query parameters**, not JSON body — roadmap item).

### `POST /recommend`

Returns ranked schemes + best partner for each.

| Param | Type | Default | Notes |
|-------|------|---------|-------|
| `project_type` | str | — | e.g. `"Business / Enterprise"`, `"Education"` (checked lower-cased) |
| `project_cost` | float | — | Total project/course cost |
| `family_income` | float | — | Annual family income |
| `loan_required` | float | — | Requested loan |
| `course_fee` | float | `0` | Only meaningful for education |
| `partner_channel` | str | `"Bank"` | Fed to the ML model |
| `latitude` | float | `17.3850` | Hyderabad fallback |
| `longitude` | float | `78.4867` | |

**Response shape:**

```jsonc
{
  "count": 2,
  "recommended_scheme": {
    "code": "TL", "name": "...", "category": "Business",
    "ai_match_score": 87.4,
    "normal_repayment_years": 10,
    "recommended_partner": {
      "id": 12, "name": "...", "type": "Bank",
      "latitude": 17.4, "longitude": 78.5,
      "distance_km": 3.2, "npa_percent": 4.0,
      "fund_utilization_percent": 91.0,
      "partner_score": 95.3
    },
    "other_partners": [ /* same shape, sorted by partner_score desc */ ]
  },
  "recommendations": [ /* all matched schemes, ai_match_score desc */ ]
}
```

### `POST /calculate-loan`

| Param | Type | Notes |
|-------|------|-------|
| `scheme_code` | str | Must exist with status `active` → else 404 |
| `loan_amount` | float | Must be ≤ `scheme.max_loan_amount` → else 400 |
| `partner_id` | int | Must exist and be `active` → else 404 |
| `desired_repayment_years` | int? | Optional; must be `< normal` and `> 0` → else 400 |

**Response:** scheme + partner summaries, `normal_plan` and (optional) `shorter_plan` with `monthly_emi`, `total_interest`, `total_payment`, `estimated_interest_saved`.

**Errors:** 404 (scheme/partner missing), 400 (limit exceeded, missing UNY channel rate, bad repayment period).

> There is **no** endpoint to list schemes, partners, or scheme details — the frontend only gets data via these two calls and route state.

---

## 8. Frontend Architecture

### Routes (react-router)

| Route | Component | Purpose |
|-------|-----------|---------|
| `/` | `Home` (inline in `App.jsx`) | Landing page: hero, features, how-it-works, CTA |
| `/eligibility` | `EligibilityForm` | Step 1 form; holds results & calculator state |
| `/partners` | `PartnerLocator` | Map + list; **requires `location.state.scheme`** (refresh breaks it) |

### Component responsibilities

- **`EligibilityForm.jsx`** — assistance type (Business/Education) toggle, project type / education level, costs, income, **SC-category question** (collected but *not yet sent to the backend*), location section with **"Use my location"** (browser geolocation + Nominatim reverse geocoding to auto-fill state/district/city). Persists last search in `sessionStorage["nexoraSearch"]` and restores it on mount.
- **`RecommendationResults.jsx`** — top-recommendation hero card, alternatives, per-scheme actions: `Calculate repayment` (opens `RepaymentCalculator`) and `Find partners` (navigates to `/partners` with scheme + user location in router state). Also inlines a details view.
- **`RepaymentCalculator.jsx`** — calls `/calculate-loan`; optional "shorter period" simulation with interest saved.
- **`PartnerLocator.jsx` + `PartnerMap.jsx`** — Leaflet map: user marker + partner markers, tooltips, score badges, selected-partner highlighting.
- **`Navbar.jsx`** — dark-mode toggle, language selector (English / తెలుగు / हिंदी — **display only, i18next never initialized**), Get Started CTA. Note: the Home page renders its **own inline duplicate navbar** instead of reusing this component.

### API layer

`services/api.js` exposes a single `getRecommendations(formData)`; the base URL `http://127.0.0.1:8000` is **hard-coded** here *and again inside* `RepaymentCalculator.jsx`.

---

## 9. Data Flow (End-to-End)

```
User fills form (EligibilityForm)
        │  POST /recommend?project_type=...&project_cost=...
        ▼
routes/recommendation.py
        │  1. load active schemes
        │  2. services/eligibility.py  → hard filter (drop ineligible)
        │  3. services/ai_ranker.py    → predict_proba per eligible scheme
        │  4. services/partner_router.py:
        │       scheme_partner_types → allowed partner types
        │       partners (active, has lat/long)
        │       Haversine distance + partner score → sort desc
        ▼
Response {recommended_scheme, recommendations[]}
        ▼
RecommendationResults (top card + alternatives)
        ├─► RepaymentCalculator ── POST /calculate-loan ──► services/calculator.py (EMI)
        └─► navigate("/partners", {state}) ──► PartnerLocator + PartnerMap (Leaflet)
```

---

## 10. Setup & Running Locally

### Backend

```bash
cd backend

# 1. Virtual environment (already present: backend/venv)
python -m venv venv
venv\Scripts\activate          # Windows  (source venv/bin/activate on POSIX)

# 2. Dependencies
pip install fastapi uvicorn sqlalchemy psycopg2-binary python-dotenv \
            pandas scikit-learn joblib pgeocode openpyxl

# 3. Environment
#    .env → DATABASE_URL=postgresql://user:password@localhost:5432/dbname

# 4. Run
uvicorn app:app --reload       # http://127.0.0.1:8000  (docs at /docs)
```

### Data / model bootstrap (one-time, in order)

```bash
python import_partners.py            # partners from Excel seed list
python geocode_partners.py           # lat/long backfill (Nominatim)
python generate_large_ai_dataset.py  # synthetic training data (needs DB schemes)
python train_model.py                # → scheme_ranker.pkl
```

### Frontend

```bash
cd frontend
npm install
npm run dev                  # http://localhost:5173
npm run lint                 # ESLint
```

> CORS allows only `localhost:5173` / `127.0.0.1:5173`, and the frontend hard-codes `http://127.0.0.1:8000`.

---

## 11. Known Issues & Gaps

**Architecture / correctness**

1. `scheme_partner_types.scheme_id` has **no FK** to `schemes.id`; no ORM `relationship()` — partner mapping resolves via an extra query per scheme.
2. `scheme_interest_rates` is queried with **raw SQL** and has **no ORM model**; UNY channel-name mapping is hard-coded in `routes/calculator.py`; `ELS` is hard-coded in `services/eligibility.py`.
3. **POST endpoints read query params**, not JSON bodies; no Pydantic request models → weak validation and poor OpenAPI docs.
4. `recommendation_engine.py` and `scheme_scorer.py` are **dead code** duplicating eligibility rules.
5. `ai_ranker.py` loads the model at import time — a missing/corrupt `.pkl` crashes every script that imports `database`, not just the API.
6. No Alembic migrations; schema changes require dropping tables (`create_all` only).
7. `Home` in `App.jsx` renders its own inline navbar (duplicate of `Navbar.jsx`) and ignores the `darkMode` props it receives.
8. Language selector is decorative — **i18next is installed but never initialized**; no translation resources exist.
9. SC-category answer is collected in the UI but **never sent** to `/recommend` (no backend field either).
10. `/partners` depends on router `location.state`; a refresh or direct visit shows "No scheme selected".

**Robustness / security**

11. `backend/.env` sits in the repo working tree; `venv/`, `__pycache__/`, `.pkl`, CSVs are not confirmed gitignored. Repository has **no commits yet** (`master` with zero history).
12. No backend `requirements.txt` / `pyproject.toml` → environment is not reproducible.
13. No request size/type limits, no rate limiting, no auth anywhere (fine for a prototype, not for public deployment).
14. Frontend geocoding calls Nominatim directly from the browser (usage-policy sensitive: should identify the app and preferably be server-proxied/cached).
15. `sessionStorage` (not `localStorage`) — saved search disappears when the tab closes.

**Quality / DX**

16. `test_*.py` files are manual scripts requiring a live DB — not pytest, no assertions, no CI.
17. Frontend has zero tests; backend has zero tests; no CI pipeline.
18. `RecommendationResults.jsx` is 610 lines mixing 3 views; `index.html` title is still `"frontend"`.
19. Training metrics (accuracy, classification report) are printed and lost; no model versioning/registry.
20. Hard-coded API URL in two frontend files; no `.env` support in Vite (`import.meta.env`).

---

## 12. Future Changes To Be Implemented (Roadmap)

Priorities: **P0** = do next, **P1** = high value soon, **P2** = polish/scale, **P3** = ambitious bets.

### P0 — Fix the foundations

| # | Change | Why / Where |
|---|--------|-------------|
| 1 | **Add `backend/requirements.txt`** (pin versions) and root `.gitignore` for `venv/`, `__pycache__/`, `.env`, `*.pkl`, CSVs | Reproducibility; repo has zero commits — first commit should be clean |
| 2 | **JSON request bodies with Pydantic models** for `/recommend` and `/calculate-loan` + response models | Validation, self-documenting OpenAPI, feeds future frontend types |
| 3 | **Add FK constraints + relationships**: `scheme_partner_types.scheme_id → schemes.id` (and a `partner_types` lookup table); make `scheme_interest_rates` a real ORM model | Kills raw SQL; removes N+1 partner queries via `selectinload` |
| 4 | **Config-driven scheme behavior**: replace hard-coded `ELS` / `UNY` special cases with scheme attributes/flags (e.g. `coverage_based: true`, `channel_rates: true`) or a `scheme_rules` JSONB column | Adding scheme #6 should require *zero code changes* |
| 5 | **Delete dead code** (`recommendation_engine.py`, `scheme_scorer.py`, duplicate `training_data.csv`) and rename `test_*.py` → `scripts/` | Clarity for every future contributor |
| 6 | **Centralize the API base URL** in `frontend/src/services/api.js` using `import.meta.env.VITE_API_URL`; also centralize the single `/calculate-loan` call there | One place to point at prod |
| 7 | **Fix the Home page** — reuse `Navbar.jsx`, remove the inline duplicate, wire `darkMode` correctly | Removes ~150 duplicated lines |
| 8 | **Wire up i18next** (init file + `en`/`te`/`hi` JSON resources) and use `useTranslation()` in Navbar/Home/EligibilityForm | The selector exists; the audience it targets doesn't get served yet |

### P1 — Real product features

| # | Change | Why |
|---|--------|-----|
| 9 | **`GET /schemes` + `GET /schemes/{code}` endpoints** with filters (category, cost range) and a `/schemes` browse page | Scheme discovery without filling the form; partner locator stops needing router state |
| 10 | **`GET /partners?scheme_code=&lat=&lon=` endpoint** so the map can refresh independently | Direct visits to `/partners` become meaningful |
| 11 | **Send SC-category (and state/district/city) to `/recommend`** and use them in eligibility (many schemes are SC-specific) | The most important user segment attribute is currently discarded |
| 12 | **Improve label quality for the ML model**: mix rule-labelled + real application outcomes, add per-scheme `acceptance_likelihood` priors, persist metrics (accuracy/AUC per run) into a `model_metadata` table, and version `scheme_ranker.pkl` | Current labels are rule-copies; today the "AI score" can't exceed its rules |
| 13 | **Save searches in `localStorage`** with a "recent searches" list | sessionStorage dies with the tab |
| 14 | **EMI amortization schedule endpoint** (`GET /calculate-loan/schedule`) + chart in `RepaymentCalculator` (yearly principal vs interest) | Natural extension of Step 3; easy win with recharts or plain SVG |
| 15 | **Input UX hardening**: format ₹ amounts (Indian grouping), range validation with helper text, disable submit until form valid, loading skeletons | Trust + fewer bad requests |
| 16 | **Accessibility pass**: labels on all inputs, focus states, `aria-*` on choice buttons, keyboard-navigable results, skip-to-content | Statutory + usability |

### P2 — Quality, safety, scale

| # | Change | Why |
|---|--------|-----|
| 17 | **Real test suite**: pytest + TestClient with a SQLite/Postgres test fixture for services (eligibility, partner scoring, EMI math are pure functions — easy wins), Vitest + React Testing Library for form → results flow | Currently 0 automated tests |
| 18 | **CI** (GitHub Actions): backend pytest + ruff, frontend lint + vitest + `vite build` on every push | Prevents regressions |
| 19 | **Alembic migrations** and stop calling `create_all` at import time | Safe schema evolution |
| 20 | **Auth + admin panel**: scheme/partner CRUD (FastAPI + simple React admin), audit fields, verification workflow using the existing `status`/`verification_status` columns | Data is currently maintained only by scripts |
| 21 | **Server-side geocoding proxy with caching** (pincode → lat/long via pgeocode already exists) | Respects Nominatim policy, faster UX |
| 22 | **Deployment**: Dockerfile + docker-compose (FastAPI + Postgres + Vite build served by nginx), env-driven CORS, gzip, structured logging, `/health` endpoint | Currently dev-only |
| 23 | **Rate limiting + request validation limits** (slowapi), Honeypot on the public form | Public-deployment hygiene |

### P3 — Ambitious bets

| # | Change | Why |
|---|--------|-----|
| 24 | **Conversational assistant (LLM)**: "I want to start a tailoring shop in Nizamabad with ₹80k" → structured form fill; explain *why* a scheme was rejected and *what to change* to qualify | Turns the eligibility filter into guidance; use the existing rejection reasons |
| 25 | **Eligibility explainer**: return `rejection_reasons[]` per scheme from `is_scheme_eligible` and render "₹20k short of loan cap" style messages | Converts a hard filter into user education |
| 26 | **Multilingual + regional**: real translations, voice input for low-literacy users, offline-capable PWA | True accessibility for the target segment |
| 27 | **Application tracker**: save a user's scheme applications, document checklist per scheme, status timeline | Moves from "advice" to "journey" |
| 28 | **Partner health analytics**: dashboard over `npa_percent` / `fund_utilization_percent` trends; retrain partner score on real disbursement data | The scoring formula is a prototype heuristic |
| 29 | **Expand scheme universe**: generalize ingestion (CSV import script per scheme) so onboarding scheme #6–#10 is data-entry, not code | Scales the product beyond 5 schemes |

### Suggested order of execution

```
Week 1:  #1 → #2 → #6 → #7          (hygiene, contract, config)
Week 2:  #3 → #4 → #5 → #8          (schema, data-driven schemes, i18n)
Week 3:  #9 → #10 → #11 → #13       (list endpoints, SC data, persistence)
Week 4:  #17 → #18 → #19            (tests + CI + migrations)
Then:    #12, #14–#16, then P2/P3 by product need.
```

---

*Last updated: 2026-09-11 · Generated from a full-codebase review of `backend/` and `frontend/`.*
