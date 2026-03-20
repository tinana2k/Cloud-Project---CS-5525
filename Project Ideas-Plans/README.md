# GolfIQ — Cloud-Powered Golf Performance Platform

> Track every round. Understand every trend. Play your best golf.

GolfIQ is a cloud-native golf performance analytics platform built on Snowflake, powered by an AI recommendation engine, and designed to help golfers of all skill levels identify weaknesses, track improvement, and receive personalized coaching insights after every round.

---

## Table of Contents

- [Overview](#overview)
- [Key Features](#key-features)
- [System Architecture](#system-architecture)
- [Tech Stack](#tech-stack)
- [Data Model](#data-model)
- [AI Recommendation Engine](#ai-recommendation-engine)
- [Team Structure & Division of Labor](#team-structure--division-of-labor)
- [7-Week Project Timeline](#7-week-project-timeline)
- [Weekly Deliverables](#weekly-deliverables)
- [Getting Started](#getting-started)
- [Environment Variables](#environment-variables)
- [API Endpoints](#api-endpoints)
- [Contributing](#contributing)

---

## Overview

Golf is a data-rich sport — every shot, every club selection, every missed putt tells a story. Yet most golfers track their scores on paper or in basic apps that only show totals. GolfIQ changes that.

By capturing hole-level and shot-level data and storing it in a Snowflake data warehouse, GolfIQ computes advanced statistics like GIR%, putts per GIR, scoring differential by par type, and rolling USGA handicap index. The AI recommendation engine then analyzes each golfer's personal dataset and returns natural-language coaching insights — the kind of feedback that would otherwise require a paid instructor.

**Target users:** Amateur golfers with a handicap between 5 and 30 who play regularly and want to improve.

**Cloud computing concepts demonstrated:**
- Snowflake data warehousing and materialized views
- Serverless REST API backend
- Multi-tenant authentication with JWT
- AI integration via Claude API (Anthropic)
- Snowflake Cortex ML for handicap trend forecasting
- Cloud-native deployment (Docker + cloud hosting)

---

## Key Features

### Round & Shot Logging
- Log scores, putts, GIR, fairways hit, and penalties per hole
- Optional shot-level detail: club, distance, lie, result, miss direction
- Mobile-friendly entry UI — designed for use on the course

### Analytics Dashboard
- Scoring trends over time (last 5, 10, 20 rounds)
- GIR%, fairway %, putts per round, and scrambling rate
- Scoring average broken down by par type (par 3 / 4 / 5)
- Course-by-course performance comparison
- Hole-by-hole scorecard history

### USGA Handicap Tracker
- Automatically calculated handicap index after every round
- Uses official USGA formula: best 8 of last 20 score differentials
- Handicap trend chart showing improvement or regression over time

### AI Coaching Insights
- Weekly AI-generated insight report per golfer
- Identifies the #1 weakness area based on rolling 90-day stats
- Specific, actionable recommendations (not generic tips)
- Example: *"Your GIR% from 150–175 yards is 18%. Focus approach shots from this range — it's your biggest scoring leak."*

### Leaderboard & Social
- Friend group leaderboards by handicap-adjusted net score
- Round comparison: view friends' scorecards side by side

---

## System Architecture

```
┌─────────────────────────────────────────────────┐
│                  Frontend (React)                │
│   Dashboard · Score entry · Round history        │
└─────────────────────┬───────────────────────────┘
                      │ HTTPS / REST
┌─────────────────────▼───────────────────────────┐
│              Backend API (FastAPI)               │
│   Auth · USGA calculator · Snowflake connector   │
└──────────┬──────────────────────────┬────────────┘
           │                          │
┌──────────▼──────────┐   ┌──────────▼────────────┐
│  Snowflake DWH       │   │   Claude API           │
│  GOLFERS · ROUNDS   │   │   AI recommendation    │
│  HOLES · SHOTS      │   │   engine               │
│  Materialized views │   │                        │
│  Cortex ML forecast │   │                        │
└─────────────────────┘   └────────────────────────┘
```

**Data flow:**
1. Golfer logs a round via the React frontend
2. Frontend calls the FastAPI backend via REST
3. Backend validates, computes score differential, writes to Snowflake
4. Materialized views in Snowflake recalculate aggregated stats
5. Weekly job queries Snowflake stats summary → sends to Claude API → stores insight
6. Dashboard reads trends and insights from Snowflake views

---

## Tech Stack

| Layer | Technology | Purpose |
|---|---|---|
| Frontend | React + Recharts | Dashboard UI and score entry |
| Backend | Python FastAPI | REST API, auth, business logic |
| Database | Snowflake | Cloud data warehouse |
| AI Engine | Anthropic Claude API | Natural language recommendations |
| ML Forecasting | Snowflake Cortex | Handicap trend prediction |
| Auth | JWT (python-jose) | Secure multi-user sessions |
| Containerization | Docker | Consistent dev/prod environment |
| Hosting | Cloud platform of choice | Deployment |
| Version Control | GitHub | Collaboration and CI |

---

## Data Model

### Core Tables

```sql
-- Golfer profiles
CREATE TABLE golfers (
    golfer_id     VARCHAR PRIMARY KEY,
    name          VARCHAR NOT NULL,
    email         VARCHAR UNIQUE NOT NULL,
    home_course   VARCHAR,
    created_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- One record per round played
CREATE TABLE rounds (
    round_id      VARCHAR PRIMARY KEY,
    golfer_id     VARCHAR REFERENCES golfers(golfer_id),
    course_name   VARCHAR NOT NULL,
    date          DATE NOT NULL,
    tee_played    VARCHAR,          -- Blue / White / Red
    course_rating DECIMAL(4,1),     -- Required for handicap
    slope_rating  INTEGER,          -- Required for handicap
    weather       VARCHAR,          -- Sunny / Windy / Rain
    created_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- One record per hole per round (up to 18)
CREATE TABLE holes (
    hole_id       VARCHAR PRIMARY KEY,
    round_id      VARCHAR REFERENCES rounds(round_id),
    hole_number   INTEGER NOT NULL,     -- 1–18
    par           INTEGER NOT NULL,     -- 3, 4, or 5
    score         INTEGER NOT NULL,     -- Gross strokes
    putts         INTEGER,
    fairway_hit   BOOLEAN,              -- NULL for par 3s
    green_in_reg  BOOLEAN,
    penalties     INTEGER DEFAULT 0,
    up_and_down   BOOLEAN,
    sand_save     BOOLEAN,
    hole_yardage  INTEGER
);

-- Optional shot-level detail
CREATE TABLE shots (
    shot_id         VARCHAR PRIMARY KEY,
    hole_id         VARCHAR REFERENCES holes(hole_id),
    shot_number     INTEGER NOT NULL,
    club            VARCHAR,    -- Driver, 7-iron, PW, etc.
    distance_yards  INTEGER,    -- Yards to pin before shot
    lie             VARCHAR,    -- Fairway, rough, sand, tee
    result          VARCHAR,    -- Green, fairway, rough, OB
    carry_yards     INTEGER,
    miss_direction  VARCHAR     -- Left, right, short, long
);
```

### Materialized View — Golfer Aggregates

```sql
CREATE MATERIALIZED VIEW golfer_stats AS
SELECT
    r.golfer_id,
    COUNT(DISTINCT r.round_id)                          AS rounds_played,
    AVG(h.score - h.par)                                AS scoring_avg_vs_par,
    AVG(h.putts)                                        AS avg_putts_per_hole,
    SUM(CASE WHEN h.green_in_reg THEN 1 ELSE 0 END)
        / COUNT(*)                                      AS gir_pct,
    SUM(CASE WHEN h.fairway_hit THEN 1 ELSE 0 END)
        / NULLIF(SUM(CASE WHEN h.par > 3 THEN 1 ELSE 0 END), 0)
                                                        AS fairway_pct,
    SUM(CASE WHEN h.penalties > 0 THEN 1 ELSE 0 END)   AS penalty_holes
FROM rounds r
JOIN holes h ON h.round_id = r.round_id
WHERE r.date >= DATEADD('day', -90, CURRENT_DATE)
GROUP BY r.golfer_id;
```

### USGA Handicap Formula

```sql
-- Score differential per round
-- Differential = (Score - Course Rating) × (113 / Slope Rating)
-- Handicap Index = Average of best 8 from last 20 differentials × 0.96

WITH differentials AS (
    SELECT
        golfer_id,
        round_id,
        date,
        (total_score - course_rating) * (113.0 / slope_rating) AS differential,
        ROW_NUMBER() OVER (PARTITION BY golfer_id ORDER BY date DESC) AS recency_rank
    FROM rounds_with_totals
    WHERE recency_rank <= 20
),
best_8 AS (
    SELECT golfer_id, differential,
           RANK() OVER (PARTITION BY golfer_id ORDER BY differential ASC) AS diff_rank
    FROM differentials
    WHERE diff_rank <= 8
)
SELECT golfer_id, AVG(differential) * 0.96 AS handicap_index
FROM best_8
GROUP BY golfer_id;
```

---

## AI Recommendation Engine

The engine runs on a scheduled weekly job. For each golfer, it:

1. Queries the Snowflake `golfer_stats` view for their last 90 days of data
2. Computes which area is weakest (GIR, putting, driving, short game)
3. Constructs a structured prompt and calls the Claude API
4. Stores the response as an insight card in Snowflake
5. The dashboard surfaces the insight on next login

```python
import anthropic
import snowflake.connector

def generate_insight(golfer_id: str):
    # Step 1: Pull stats from Snowflake
    stats = query_snowflake(f"""
        SELECT scoring_avg_vs_par, gir_pct, avg_putts_per_hole,
               fairway_pct, penalty_holes, rounds_played
        FROM golfer_stats
        WHERE golfer_id = '{golfer_id}'
    """)

    # Step 2: Build coaching prompt
    prompt = f"""
    You are an expert golf coach analyzing a golfer's performance data.

    Last 90 days ({stats['rounds_played']} rounds):
    - Scoring average: {stats['scoring_avg_vs_par']:+.1f} vs par
    - Greens in regulation: {stats['gir_pct']:.0%}
    - Average putts per hole: {stats['avg_putts_per_hole']:.2f}
    - Fairways hit: {stats['fairway_pct']:.0%}
    - Penalty holes per round: {stats['penalty_holes'] / stats['rounds_played']:.1f}

    Identify the single biggest scoring opportunity and give 2–3 specific,
    actionable recommendations. Be direct and concrete. No generic tips.
    Keep your response under 150 words.
    """

    # Step 3: Call Claude API
    client = anthropic.Anthropic()
    response = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=300,
        messages=[{"role": "user", "content": prompt}]
    )

    insight_text = response.content[0].text

    # Step 4: Store back to Snowflake
    store_insight(golfer_id, insight_text)
    return insight_text
```

---

## Team Structure & Division of Labor

GolfIQ is built by a team of five. Each member owns a vertical and is the subject-matter expert for their layer. Members collaborate during integration weeks.

### Member 1 — Data Engineer (Snowflake Lead)
**Owns:** Database design, Snowflake schema, SQL views, handicap logic, Cortex ML

**Responsibilities:**
- Design and create all Snowflake tables (GOLFERS, ROUNDS, HOLES, SHOTS)
- Write and maintain materialized views for aggregated stats
- Implement the USGA handicap index calculation in SQL
- Set up Snowflake Cortex `FORECAST()` for handicap trend prediction
- Write data seeding scripts for demo data
- Optimize query performance and ensure views refresh correctly
- Document all SQL schemas and view logic

**Key deliverables:** Schema DDL scripts, views, handicap SQL, seeding script, Cortex forecast query

---

### Member 2 — Backend Developer (API Lead)
**Owns:** FastAPI server, REST endpoints, authentication, Snowflake Python connector

**Responsibilities:**
- Set up FastAPI project structure with routing and middleware
- Implement JWT-based authentication (register, login, token refresh)
- Build all REST endpoints (round submission, stats retrieval, insight fetch)
- Integrate `snowflake-connector-python` for all DB reads and writes
- Implement server-side USGA differential calculation on round submission
- Write unit tests for API endpoints
- Manage environment config and `.env` structure
- Document all endpoints (auto-generated via FastAPI OpenAPI)

**Key deliverables:** Running API server, all endpoints, auth system, Snowflake connector integration, API docs

---

### Member 3 — Frontend Developer (Dashboard Lead)
**Owns:** React application, dashboard UI, score entry flow, Recharts visualizations

**Responsibilities:**
- Set up React project with routing and state management
- Build score entry form (round details → 18 holes → optional shots)
- Build analytics dashboard with Recharts line, bar, and radial charts
- Build handicap tracker page with trend chart
- Build AI insight card component
- Implement JWT auth flow (login/register screens, token storage)
- Ensure mobile-responsive layout for on-course score entry
- Connect all components to backend API endpoints

**Key deliverables:** Full React application, score entry UI, dashboard with charts, mobile layout, connected to API

---

### Member 4 — AI Engineer (Recommendation Engine Lead)
**Owns:** Claude API integration, prompt engineering, weekly insight job, insight storage

**Responsibilities:**
- Design and iterate on the coaching prompt template
- Build the Python insight generation script (query → prompt → Claude → store)
- Set up scheduled job to run insights weekly per golfer
- Tune prompts to produce specific, non-generic recommendations
- Implement fallback logic for golfers with fewer than 5 rounds
- Evaluate output quality and refine prompts across different golfer profiles
- Integrate insight storage back into Snowflake
- Test with synthetic golfer profiles at different handicap levels

**Key deliverables:** Working recommendation engine, prompt templates, insight scheduler, quality evaluation results

---

### Member 5 — DevOps & Integration Lead (Project Manager)
**Owns:** Docker setup, deployment, GitHub workflow, demo coordination, documentation

**Responsibilities:**
- Set up GitHub repository, branching strategy, and PR review process
- Write `Dockerfile` and `docker-compose.yml` for frontend + backend
- Set up environment variable management across dev and prod
- Manage cloud deployment (hosting platform setup)
- Coordinate weekly standups and track milestone completion
- Lead integration testing when all layers connect
- Prepare the demo script and presentation slide deck
- Write final project README and architecture documentation

**Key deliverables:** Docker setup, deployed app URL, GitHub workflow, demo script, slide deck, final README

---

## 7-Week Project Timeline

```
Week 1   Foundation & Setup
Week 2   Core Data Layer
Week 3   Backend API
Week 4   Frontend UI
Week 5   AI Engine & Integration
Week 6   Polish & Testing
Week 7   Demo Preparation & Presentation
```

---

## Weekly Deliverables

### Week 1 — Foundation & Setup
**Goal:** Everyone has a working local environment and understands the full architecture.

**All team:**
- Read and agree on system architecture and data model
- Set up GitHub repo with branch protection and PR template
- Set up local `.env` files and install all dependencies
- Each member gives a 5-minute architecture walkthrough in team meeting

**Member 1 (Data):**
- Create Snowflake account and configure warehouse, database, schema
- Write initial DDL for GOLFERS, ROUNDS, HOLES, SHOTS tables
- Confirm Snowflake connector works from local Python

**Member 2 (Backend):**
- Initialize FastAPI project structure
- Confirm Python + Snowflake connector runs locally
- Stub out all planned API endpoints (return 200 with placeholder JSON)

**Member 3 (Frontend):**
- Initialize React project with routing (React Router)
- Set up Recharts and confirm a sample chart renders
- Build static wireframe mockups of dashboard and score entry

**Member 4 (AI):**
- Set up Anthropic API key and confirm Claude API call works
- Write first draft of coaching prompt with hardcoded sample data
- Review output quality and note what works / what needs improving

**Member 5 (DevOps):**
- Create GitHub repo, add all members, set up branch rules
- Write initial `docker-compose.yml` (frontend + backend containers)
- Schedule weekly 30-minute team standup

**Milestone:** All five environments running, repo set up, Snowflake tables created.

---

### Week 2 — Core Data Layer
**Goal:** Snowflake is fully designed, seeded with demo data, and queryable.

**Member 1 (Data):**
- Finalize full schema (add all columns, constraints, nullable fields)
- Write materialized view for `golfer_stats` aggregation
- Implement USGA handicap SQL query (best 8 of 20 differentials)
- Write Python seeding script generating 3 demo golfers × 15 rounds each
- Run seeding script, confirm data looks correct in Snowflake UI
- Begin Snowflake Cortex `FORECAST()` experiment on handicap trend

**Member 2 (Backend):**
- Implement JWT auth endpoints: `POST /auth/register`, `POST /auth/login`
- Implement `POST /rounds` endpoint that writes to Snowflake
- Test round submission end-to-end with Snowflake

**Member 3 (Frontend):**
- Build score entry form: round details screen (course, date, tee, conditions)
- Build 18-hole entry grid (hole number, par, score, putts, GIR, fairway)
- No API connection yet — use local state

**Member 4 (AI):**
- Refine coaching prompt using seeded data as input
- Test at least 5 different golfer profiles (low/mid/high handicap)
- Document prompt versions and output quality notes

**Member 5 (DevOps):**
- Confirm Docker containers build and run together
- Set up GitHub Actions for basic lint and test checks on PR
- Review Week 1 deliverables are all complete

**Milestone:** Seeded Snowflake database with 3 golfers, working auth API, score entry form built.

---

### Week 3 — Backend API Complete
**Goal:** All API endpoints are working and tested end-to-end with Snowflake.

**Member 1 (Data):**
- Finalize `golfer_stats` materialized view with all computed fields
- Add `putts_per_gir` and `scoring_by_par` views
- Ensure views refresh correctly after new round is inserted
- Snowflake Cortex forecasting query delivering handicap projection

**Member 2 (Backend):**
- `GET /golfers/{id}/stats` — returns aggregated stats from materialized view
- `GET /golfers/{id}/rounds` — returns round history with hole details
- `GET /golfers/{id}/handicap` — returns current index + last 20 differentials
- `GET /golfers/{id}/insights` — returns latest AI insight
- Add request validation (Pydantic models) to all endpoints
- Write integration tests for all endpoints
- Generate and verify FastAPI OpenAPI docs at `/docs`

**Member 3 (Frontend):**
- Build optional shot-level entry component (club, distance, lie, result)
- Build round history page (list of past rounds, click to expand scorecard)
- Begin dashboard page shell with placeholder charts

**Member 4 (AI):**
- Build `generate_insight.py` script — full flow: query → prompt → Claude → print
- Test with all 3 seeded demo golfers, review output
- Identify edge cases: golfer with < 5 rounds, all-perfect golfer, high handicapper

**Member 5 (DevOps):**
- Deploy backend API to cloud hosting environment
- Confirm API is reachable at public URL
- Test all endpoints against deployed version

**Milestone:** All API endpoints live, tested, and documented. Frontend has full score entry flow.

---

### Week 4 — Frontend Dashboard Complete
**Goal:** The full React dashboard is built and connected to the live API.

**Member 1 (Data):**
- Support Member 2/3 with any SQL query debugging
- Begin preparing demo dataset — realistic scoring patterns for 3 personas:
  - Persona A: 18-handicap improving over time
  - Persona B: 8-handicap with a putting problem
  - Persona C: 28-handicap beginner

**Member 2 (Backend):**
- Support frontend integration — fix any CORS issues, adjust response shapes
- Add pagination to `GET /golfers/{id}/rounds`
- Assist with debugging API connection issues

**Member 3 (Frontend):**
- Connect score entry form to `POST /rounds` API — full submission works
- Build analytics dashboard with live data from `GET /golfers/{id}/stats`:
  - Scoring trend line chart (last 10 rounds)
  - GIR% bar chart by round
  - Putts per round bar chart
  - Par type performance breakdown
- Build handicap tracker page with trend chart and index display
- Build AI insight card — displays latest coaching recommendation
- Implement login / register screens with JWT token storage

**Member 4 (AI):**
- Integrate insight storage: write result back to Snowflake `insights` table
- Confirm `GET /golfers/{id}/insights` returns latest insight from DB
- Refine prompt for clarity and actionability based on prior testing

**Member 5 (DevOps):**
- Deploy frontend to cloud hosting
- Configure CORS between frontend and backend domains
- End-to-end smoke test: register → log round → view dashboard

**Milestone:** Full working application: register, log a round, see dashboard, see AI insight.

---

### Week 5 — AI Engine & Full Integration
**Goal:** Recommendation engine is fully automated and the whole stack runs end-to-end.

**Member 1 (Data):**
- Finalize demo data — all 3 personas have 15+ rounds with realistic patterns
- Snowflake Cortex `FORECAST()` running and returning handicap projection
- Expose handicap forecast as a view or endpoint-ready query

**Member 2 (Backend):**
- Add `GET /golfers/{id}/handicap/forecast` endpoint using Cortex data
- Harden error handling across all endpoints (graceful 4xx / 5xx responses)
- Final review of API security (token expiry, input sanitization)

**Member 3 (Frontend):**
- Add handicap forecast chart to handicap tracker page
- Polish dashboard UI — spacing, colors, loading states, empty states
- Add leaderboard page (friends' handicap-adjusted net scores)
- Mobile responsiveness pass — test score entry on phone screen size

**Member 4 (AI):**
- Set up weekly scheduled job (cron or cloud scheduler) to run insight generation
- Confirm job runs for all golfers and stores insights correctly
- Final prompt tuning — ensure outputs are under 150 words and specific
- Write documentation for how the engine works

**Member 5 (DevOps):**
- Full integration test across all five modules
- Document any bugs found and assign to relevant members
- Prepare demo environment: seed demo data, create 3 demo accounts

**Milestone:** Complete, connected application with automated AI insights and forecast. Demo accounts ready.

---

### Week 6 — Polish & Testing
**Goal:** The app is stable, looks professional, and handles edge cases gracefully.

**All team:**
- Each member tests the full user journey from their own device
- File and fix any bugs found
- Review the demo script together and identify gaps

**Member 1 (Data):**
- Verify all views are performant (query times under 2 seconds)
- Confirm handicap recalculates correctly after each new round
- Final data quality check on all demo personas

**Member 2 (Backend):**
- Fix all bugs identified in integration testing
- Add rate limiting to insight generation endpoint
- Confirm API handles missing/null data gracefully

**Member 3 (Frontend):**
- Fix all UI bugs, polish loading and error states
- Improve chart labels and tooltips for demo clarity
- Ensure the score entry flow works smoothly on mobile

**Member 4 (AI):**
- Run final insight generation for all 3 demo personas
- Verify outputs are compelling for demo — re-run if needed
- Prepare 3 example insight outputs to show during demo

**Member 5 (DevOps):**
- Final deployment check — confirm all services are stable
- Write test checklist and run full regression across all features
- Begin slide deck: problem statement, architecture, live demo plan

**Milestone:** Stable, polished application ready for demo. No critical bugs outstanding.

---

### Week 7 — Demo Preparation & Presentation
**Goal:** Deliver a compelling, well-rehearsed demo that showcases all cloud computing concepts.

**All team:**
- Rehearse the full demo at least twice as a group
- Each member prepares to speak about their own module
- Anticipate professor questions about Snowflake, AI integration, cloud design

**Member 1 (Data):**
- Prepare to walk through Snowflake schema and views live
- Have Snowflake UI open showing GOLFERS, ROUNDS, HOLES tables
- Show the handicap SQL query executing in real time

**Member 2 (Backend):**
- Prepare to show FastAPI `/docs` page and execute a live API call
- Explain JWT auth flow if asked

**Member 3 (Frontend):**
- Lead the live demo — log a round, watch dashboard update in real time
- Walk through each chart and explain what it shows

**Member 4 (AI):**
- Trigger a live insight generation during the demo
- Explain how the prompt is constructed and why the output is useful
- Show 3 example outputs for different golfer personas

**Member 5 (DevOps):**
- Present architecture diagram slide
- Present tech stack slide
- Manage presentation flow and timing

**Milestone:** Demo delivered. Project complete.

---

## Getting Started

### Prerequisites

- Python 3.11+
- Node.js 18+
- Docker Desktop
- Snowflake account (free trial at snowflake.com)
- Anthropic API key (console.anthropic.com)

### Clone the Repository

```bash
git clone https://github.com/your-team/golfiq.git
cd golfiq
```

### Start with Docker

```bash
docker-compose up --build
```

This starts both the frontend (port 3000) and backend (port 8000) containers.

### Manual Setup — Backend

```bash
cd backend
python -m venv venv
source venv/bin/activate       # Windows: venv\Scripts\activate
pip install -r requirements.txt
uvicorn main:app --reload
```

### Manual Setup — Frontend

```bash
cd frontend
npm install
npm run dev
```

---

## Environment Variables

Create a `.env` file in `/backend`:

```env
# Snowflake
SNOWFLAKE_ACCOUNT=your_account_identifier
SNOWFLAKE_USER=your_username
SNOWFLAKE_PASSWORD=your_password
SNOWFLAKE_WAREHOUSE=COMPUTE_WH
SNOWFLAKE_DATABASE=GOLFIQ
SNOWFLAKE_SCHEMA=PUBLIC

# Auth
JWT_SECRET=your_secret_key_here
JWT_ALGORITHM=HS256
JWT_EXPIRY_MINUTES=1440

# Anthropic
ANTHROPIC_API_KEY=sk-ant-...
```

Create a `.env` file in `/frontend`:

```env
VITE_API_BASE_URL=http://localhost:8000
```

---

## API Endpoints

| Method | Endpoint | Description |
|---|---|---|
| POST | `/auth/register` | Create new golfer account |
| POST | `/auth/login` | Login and receive JWT token |
| POST | `/rounds` | Submit a completed round |
| GET | `/golfers/{id}/stats` | Get aggregated 90-day stats |
| GET | `/golfers/{id}/rounds` | Get round history with hole details |
| GET | `/golfers/{id}/handicap` | Get current handicap index + history |
| GET | `/golfers/{id}/handicap/forecast` | Get projected handicap trend |
| GET | `/golfers/{id}/insights` | Get latest AI coaching insight |
| GET | `/leaderboard` | Get friend group net score leaderboard |

Full interactive API documentation available at `http://localhost:8000/docs` when the backend is running.

---

## Contributing

1. Create a branch from `main`: `git checkout -b feature/your-feature-name`
2. Make your changes and write tests where applicable
3. Open a pull request — at least one team member must review before merge
4. Squash commits on merge to keep history clean

**Branch naming:**
- `feature/` — new functionality
- `fix/` — bug fixes
- `data/` — schema or SQL changes
- `docs/` — documentation updates

---

## Team

| Member | Role | Module |
|---|---|---|
| Member 1 | Data Engineer | Snowflake schema, SQL views, handicap logic |
| Member 2 | Backend Developer | FastAPI, REST endpoints, auth |
| Member 3 | Frontend Developer | React dashboard, score entry, charts |
| Member 4 | AI Engineer | Claude API integration, prompt engineering |
| Member 5 | DevOps & PM | Docker, deployment, demo, coordination |

---

*GolfIQ — Big Data & Analytics Course Project*
