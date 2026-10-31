# BLACK ICE — Enterprise Production Deployment Guide

> **Architecture:**
> - **Frontend:** React 19 + Vite SPA hosted on **Vercel**
> - **Backend:** Python FastAPI ASGI Service hosted on **Render**
> - **Database & Auth Layer:** Managed PostgreSQL & RLS on **Supabase**
> - **CI/CD & Version Control:** **GitHub**

---

## 1. Hosting Architecture Overview

```
                      [ USER BROWSER ]
                             │
            ┌────────────────┴────────────────┐
            ▼                                 ▼
   [ Vercel CDN ]                     [ Render Gateway ]
   React 19 / Vite SPA                FastAPI Python Backend
   (Static Assets + SPA Routing)      (Security Heuristic Engine)
            │                                 │
            │  API Requests (Bearer JWT)      │
            └────────────────────────────────►│
                                              ▼
                                    [ Supabase Cloud ]
                                    PostgreSQL 16 + RLS
                                    (Scans, Threats, Users)
```

---

## 2. Supabase Setup (PostgreSQL Database)

1. Create a new project in the [Supabase Dashboard](https://supabase.com).
2. Go to **Project Settings** $\to$ **Database** and copy your **Connection String**:
   - For Render server connections, use the direct URI (Port `5432`) or the Session Pooler URI (Port `6543` / `5432`):
     ```
     postgresql://postgres:[YOUR-PASSWORD]@db.[YOUR-PROJECT-REF].supabase.co:5432/postgres
     ```
3. Open the **SQL Editor** in Supabase and run the schema & RLS migration script from:
   ```
   supabase/migrations/001_initial_schema_and_rls.sql
   ```
   *This initializes the tables (`users`, `scans`, `threat_events`, `document_scans`), performance indexes, and applies strict Row Level Security (RLS) policies blocking public unauthorized PostgREST access.*

---

## 3. Render Setup (FastAPI Backend)

### Option A: 1-Click Blueprint Deploy via `render.yaml`
1. Connect your GitHub repository to Render.
2. Render will automatically detect [`render.yaml`](file:///d:/First%20folder/Black%20Ice%20(1)/Black%20Ice/render.yaml) and configure the service.

### Option B: Manual Web Service Setup
1. Create a **New Web Service** pointing to your GitHub repository.
2. Configure settings:
   - **Root Directory:** `backend`
   - **Runtime:** `Python 3`
   - **Build Command:** `pip install --upgrade pip && pip install -r requirements.txt`
   - **Start Command:** `uvicorn app.main:app --host 0.0.0.0 --port $PORT --app-dir .`
   - **Health Check Path:** `/api/health`
3. Add Environment Variables in Render:
   | Variable | Value | Description |
   |---|---|---|
   | `ENVIRONMENT` | `production` | Enables production security mode |
   | `SECRET_KEY` | *(Generate 64-char random hex)* | JWT signing secret |
   | `DATABASE_URL` | `postgresql://postgres:...` | Your Supabase PostgreSQL URI |
   | `CORS_ORIGINS` | `https://your-app.vercel.app` | Vercel production frontend domain |
   | `GEMINI_API_KEY` | *(Optional)* | For cloud neural guardrail checks |
   | `OPENAI_API_KEY` | *(Optional)* | For cloud neural moderation checks |

---

## 4. Vercel Setup (Frontend SPA)

1. Import your GitHub repository in the [Vercel Dashboard](https://vercel.com).
2. In the project setup screen, configure:
   - **Framework Preset:** `Vite`
   - **Root Directory:** `frontend`
   - **Build Command:** `npm run build`
   - **Output Directory:** `dist`
   - **Install Command:** `npm install`
3. Add Environment Variables in Vercel:
   | Variable | Value | Description |
   |---|---|---|
   | `VITE_API_BASE_URL` | `https://blackice-firewall-api.onrender.com/api` | Your live Render backend URL |

4. Client-side SPA routing rewrites are automatically handled via [`frontend/vercel.json`](file:///d:/First%20folder/Black%20Ice%20(1)/Black%20Ice/frontend/vercel.json) to ensure direct subpath navigation (`/dashboard`, `/demo`, `/scanner`) does not 404.

---

## 5. Post-Deployment Verification

1. **Verify Backend Health:**
   ```bash
   curl -I https://blackice-firewall-api.onrender.com/api/health
   # Expected: HTTP 200 OK {"status": "healthy", ...}
   ```
2. **Verify CORS Headers:**
   ```bash
   curl -I -H "Origin: https://your-app.vercel.app" https://blackice-firewall-api.onrender.com/api/health
   # Expected: Access-Control-Allow-Origin: https://your-app.vercel.app
   ```
3. **Verify Interactive Demo:**
   Navigate to `https://your-app.vercel.app/demo` and trigger the 1-click 4-stage guided demo to verify end-to-end database writes to Supabase.
