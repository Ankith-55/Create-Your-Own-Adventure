# 📖 AI Story Generator

An interactive, AI-powered choose-your-own-adventure story generator. Enter a theme, and the app generates a branching narrative using GPT-4 — complete with choices that lead to different endings.

**Live Demo:** `https://your-vercel-url.vercel.app`

---

## Features

- Generate branching, choose-your-own-adventure stories from any theme
- GPT-4 powered narrative generation via LangChain
- Async background job processing with real-time polling
- Session-based story persistence with PostgreSQL
- Fully deployed full-stack application (frontend on Vercel, backend on Render)

---

## Tech Stack

| Layer | Technology |
|---|---|
| Frontend | React + Vite |
| Backend | FastAPI (Python) |
| AI | LangChain + OpenAI GPT-4 Turbo |
| ORM | SQLAlchemy |
| Database | PostgreSQL |
| Frontend Hosting | Vercel |
| Backend Hosting | Render |
| Source Control | GitHub |

---

## Architecture

```
Browser
  ↓  HTTPS
Vercel  (React + Vite — static frontend)
  ↓  REST API calls
Render  (FastAPI — backend server)
  ↓
OpenAI API  (GPT-4 Turbo via LangChain)
  ↓
PostgreSQL  (story & job persistence)
```

---

## How It Works

1. User enters a theme (e.g. "space western", "dark fairy tale")
2. Frontend `POST /api/stories/create` — backend creates a job and returns a `job_id`
3. Story generation runs as a **background task** (non-blocking)
4. Frontend polls `GET /api/job/{job_id}` every 5 seconds
5. Once `status: completed`, frontend navigates to the story viewer
6. The story tree (nodes + choices) is fetched and rendered interactively

---

## Project Structure

```
/
├── backend/
│   ├── core/
│   │   ├── config.py          # Settings via pydantic-settings
│   │   ├── models.py          # LLM response Pydantic models
│   │   ├── prompts.py         # LangChain prompt templates
│   │   └── story_generator.py # LangChain + OpenAI generation logic
│   ├── db/
│   │   └── database.py        # SQLAlchemy engine + session
│   ├── models/
│   │   ├── story.py           # Story + StoryNode ORM models
│   │   └── job.py             # StoryJob ORM model
│   ├── routers/
│   │   ├── story.py           # /stories routes
│   │   └── job.py             # /job routes
│   └── schema/
│       ├── story.py           # Pydantic response schemas
│       └── job.py             # Job response schema
└── frontend/
    ├── StoryGenerator.jsx     # Theme input + job polling logic
    ├── StoryGame.jsx          # Interactive story tree renderer
    ├── StoryLoader.jsx        # Story fetching wrapper
    ├── ThemeInput.jsx         # Theme input form
    └── LoadingStatus.jsx      # Polling status display
```

---

## Local Development

### Prerequisites

- Python 3.11+
- Node.js 18+
- PostgreSQL running locally
- OpenAI API key

### Backend

```bash
cd backend

# Install dependencies
pip install -r requirements.txt

# Create .env file
cp .env.example .env
# Fill in: OPENAI_API_KEY, DATABASE_URL, ALLOWED_ORIGINS

# Run server
uvicorn main:app --reload --port 8086
```

### Frontend

```bash
cd frontend

# Install dependencies
npm install

# Create .env.local
VITE_API_URL=http://localhost:8086/api

# Run dev server
npm run dev
```

> During local development, you can also configure a Vite proxy in `vite.config.js` to forward `/api` to `localhost:8086`. Note: **this proxy only works in development** and must be replaced with `VITE_API_URL` in production.

### Environment Variables

**Backend `.env`:**

```
OPENAI_API_KEY=sk-...
DATABASE_URL=postgresql://user:password@localhost:5432/storydb
ALLOWED_ORIGINS=http://localhost:5173
DEBUG=True
```

**Frontend `.env.local`:**

```
VITE_API_URL=http://localhost:8086/api
```

---

## Production Deployment

This section documents the full deployment process, including the real problems encountered and how they were solved.

### Overview

| Service | Platform | Config |
|---|---|---|
| Frontend | Vercel | Root Dir: `frontend`, Build: `npm run build`, Output: `dist` |
| Backend | Render | Root Dir: `backend`, Start: `uvicorn main:app --host 0.0.0.0 --port $PORT` |

---

### Backend — Render

**Setup:**
- Connect GitHub repository to Render
- Set Root Directory to `backend`
- Build Command: `pip install -r requirements.txt`
- Start Command: `uvicorn main:app --host 0.0.0.0 --port $PORT`

**Key learning — never hardcode ports:**  
Render (and most cloud platforms) inject the port dynamically via a `$PORT` environment variable. Hardcoding `--port 8086` will cause the deployment to fail silently.

**Environment variables** must be added manually in the Render dashboard — your local `.env` is never uploaded:

```
OPENAI_API_KEY=sk-...
DATABASE_URL=postgresql://...render-postgres-url...
ALLOWED_ORIGINS=https://your-app.vercel.app
```

---

### Frontend — Vercel

**Setup:**
- Connect GitHub repository to Vercel
- Framework Preset: Vite
- Root Directory: `frontend`
- Build Command: `npm run build`
- Output Directory: `dist`

**Environment variable:**

```
VITE_API_URL=https://your-backend.onrender.com/api
```

> Vite only exposes env variables prefixed with `VITE_` to the browser. Set this in the Vercel dashboard under Environment Variables.

---

### The Vite Proxy Trap

During development, `vite.config.js` can proxy `/api` to `localhost:8086`, making it feel like frontend and backend are on the same origin. **This proxy does not exist in production.**

After deployment, all API calls must use the full backend URL:

```js
// ❌ Development shortcut (breaks in production)
const API_BASE_URL = "/api"

// ✅ Correct for production
const API_BASE_URL = import.meta.env.VITE_API_URL
```

---

### CORS — The Most Common Deployment Problem

Because the frontend (Vercel) and backend (Render) are on different domains, the browser sends a **CORS preflight `OPTIONS` request** before every API call. Without the correct headers on the backend, every request fails with:

```
No 'Access-Control-Allow-Origin' header is present
```

**Fix — FastAPI `CORSMiddleware`:**

```python
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.ALLOWED_ORIGINS,  # parsed from comma-separated env var
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

**`ALLOWED_ORIGINS` on Render must be set to the frontend URL only:**

```
ALLOWED_ORIGINS=https://your-app.vercel.app
```

Common mistakes that break CORS:
- Including `/api` in the origin → ❌ (origin is domain only, no path)
- Including the backend URL in `ALLOWED_ORIGINS` → ❌ (it's the *frontend* origin)
- Forgetting to separate multiple origins with commas → ❌

---

### API Prefix

All backend routes are registered under `/api`:

```python
# main.py
app.include_router(story_router, prefix="/api")
app.include_router(job_router, prefix="/api")
```

So `VITE_API_URL` must include `/api`:

```
# ✅ Correct
VITE_API_URL=https://your-backend.onrender.com/api

# ❌ Wrong — all requests will 404
VITE_API_URL=https://your-backend.onrender.com
```

---

### Debugging Methodology

When things broke in production, this isolation strategy found the root cause fast:

```
1. Test backend via Swagger UI (/docs)
      ↓ backend works?
2. Test database connection (check Render logs)
      ↓ DB works?
3. Test OpenAI call (trigger story generation via Swagger)
      ↓ AI works?
4. Test background job (poll job status via Swagger)
      ↓ job completes?
5. Test frontend (open browser, check Network tab)
```

**Never debug frontend and backend simultaneously.** Swagger (`/docs`) lets you test the entire backend independently — if Swagger works and the frontend doesn't, the problem is almost certainly CORS or the wrong `VITE_API_URL`.

---

### Production vs Development — Key Differences

| Concern | Development | Production |
|---|---|---|
| API URL | Vite proxy → `/api` | Full URL via `VITE_API_URL` |
| Environment variables | `.env` file | Set manually in cloud dashboard |
| Port | Hardcoded (`8086`) | Injected via `$PORT` |
| CORS | Same origin (proxied) | Cross-origin — must configure middleware |
| Secrets | Local `.env` | Never committed; set in dashboard |

---

## API Reference

### `POST /api/stories/create`

Create a new story generation job.

**Request:**
```json
{ "theme": "dark fantasy" }
```

**Response:**
```json
{ "job_id": "uuid", "status": "pending", "session_id": "uuid" }
```

### `GET /api/job/{job_id}`

Poll background job status.

**Response:**
```json
{ "job_id": "uuid", "status": "processing|completed|failed", "story_id": 1 }
```

### `GET /api/stories/{story_id}/complete`

Fetch the full story tree.

**Response:** Story object with root node and all branching nodes.

---

## License

MIT
