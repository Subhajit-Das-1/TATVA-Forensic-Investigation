# TATVA Forensic Investigation | Production Deployment Guide

This guide provides step-by-step instructions for deploying the **TATVA Forensic Investigation & Graph Intelligence Platform** to production.

---

## 🏗️ Deployment Architecture

TATVA is structured as a decoupled client-server web application utilizing serverless cloud databases:

```mermaid
graph TD
    Client[React Frontend <br> Vercel / Netlify] -- HTTPS / REST --> API[FastAPI Backend <br> Render / AWS ECS / Docker]
    API -- SQL Queries --> Supabase[(Supabase PostgreSQL <br> Case Metadata & Audits)]
    API -- Cypher Queries --> Neo4j[(Neo4j AuraDB <br> Knowledge Graph)]
    API -- Caching / Sync --> Redis[(Upstash Redis <br> Payload Cache)]
    API -- NLP/OCR & AI Dossier --> Gemini[Google Gemini API <br> LLM Analysis]
```

---

## 1️⃣ Cloud Infrastructure Provisioning (Prerequisites)

Before deploying the frontend or backend, provision your cloud data stores:

### A. Neo4j AuraDB (Knowledge Graph)
1. Sign up at [Neo4j Aura Console](https://console.neo4j.io/).
2. Create a **Neo4j AuraDB Free** instance.
3. Download the credentials text file containing the **Connection URI** (starts with `neo4j+s://` or `bolt+s://`), the username (`neo4j`), and the auto-generated **Password**.
4. The database is read-only for visualization by default. Keep these credentials handy for the backend `.env`.

### B. Supabase (PostgreSQL 13+)
1. Create a project at [Supabase](https://supabase.com).
2. Go to **Project Settings -> Database**.
3. Under **Connection string**, copy the `URI` (transaction mode pooler is recommended, port `5432` or `6543`).
4. Note your password. This URL will be used for both metadata persistence and audits.

### C. Upstash Redis (Serverless Caching)
1. Sign up at [Upstash Console](https://console.upstash.com/).
2. Create a serverless Redis database.
3. Copy the **Redis URL** (`redis://default:password@host:port`).

### D. Google Gemini API
1. Create an API key on the [Google AI Studio](https://aistudio.google.com/).
2. Keep this key for LLM-based column mapping, Spacy fallback logic, and dossier generation.

---

## 2️⃣ Backend Production Deployment

The backend is built with FastAPI. It must be deployed to an environment that supports Python 3.10+ and allows **persistent volumes** (vital for storing uploaded case files).

### A. Environment Variables Checklists (`.env`)
Configure the following environment variables in your server hosting settings (do not commit your actual secrets to Git):

```ini
# Neo4j Settings
NEO4J_URI=neo4j+s://xxxxxx.databases.neo4j.io
NEO4J_USER=neo4j
NEO4J_PASSWORD=your_neo4j_password

# Supabase PostgreSQL Settings
# Can use Host/Port split OR the unified DB URL
SUPABASE_DB_URL=postgresql://postgres.xxxxxx:your_db_password@aws-0-us-east-1.pooler.supabase.com:5432/postgres
POSTGRES_HOST=aws-0-us-east-1.pooler.supabase.com
POSTGRES_PORT=5432
POSTGRES_DB=postgres
POSTGRES_USER=postgres.xxxxxx
POSTGRES_PASSWORD=your_db_password

# Redis Caching Settings
REDIS_URL=redis://default:your_redis_password@your-upstash-redis.upstash.io:6379

# AI Model Keys
GEMINI_API_KEY=AIzaSy...
```

### B. Dockerized Deployment (Recommended)
We have added a production-ready `Dockerfile` in the `/backend` folder.

1. **Build the container image**:
   ```bash
   cd backend
   docker build -t tatva-backend .
   ```

2. **Run the container locally to test**:
   ```bash
   docker run -d \
     -p 8000:8000 \
     --env-file .env \
     -v tatva_uploads:/app/uploads \
     --name tatva-api \
     tatva-backend
   ```
   > [!IMPORTANT]
   > Ensure you mount a volume to `/app/uploads` (e.g. `-v tatva_uploads:/app/uploads`). If you do not mount a persistent volume, all uploaded raw case files (CDRs, bank statements, logs) will be deleted whenever the container restarts or redeploys!

### C. Deploying to Render via Blueprints (Recommended & Simplest)
We have added a `render.yaml` Blueprint file at the root of the repository. This enables you to deploy both the Frontend and Backend simultaneously:
1. Connect your GitHub/GitLab repository to **Render**.
2. Navigate to the **Blueprints** tab on the Render Dashboard and click **New Blueprint Instance**.
3. Select your repository. Render will automatically read the `render.yaml` file.
4. Input the environment variables (Neo4j, Supabase PostgreSQL, Upstash Redis, and Gemini key) when prompted.
5. Click **Approve**. Render will build and spin up the frontend, backend, and attach the persistent disk volume automatically.
6. Once the backend deploy succeeds, note its public HTTPS URL (e.g. `https://tatva-backend.onrender.com`) and update the `VITE_API_BASE_URL` env variable in the frontend settings if it differs from the default.

### D. Manual Deployment to Render (Alternate)
If you prefer manual deployment instead of Blueprints:
* **Backend (Web Service)**:
  1. Create a new **Web Service** on Render pointing to your repo.
  2. Set Root Directory to `backend` and runtime to **Docker** (it will auto-detect the `Dockerfile`).
  3. Go to **Advanced** and add a **Persistent Disk** mounted at `/app/uploads` (min 1GB).
  4. Fill in the environment variables from the `.env` checklist.
* **Frontend (Static Site)**:
  1. Create a new **Static Site** on Render pointing to your repo.
  2. Set Root Directory to `frontend`.
  3. Set Build Command to `npm run build` and Publish Directory to `dist`.
  4. Add `VITE_API_BASE_URL` pointing to your Backend URL.

---

## 3️⃣ Frontend Production Deployment

The frontend is a static Single Page Application built using Vite, React, and Tailwind CSS. It can be hosted on static platforms like Vercel, Netlify, or Cloudflare Pages.

### A. Environment Variable
You must set this environment variable in your frontend hosting console before running the build:

* **`VITE_API_BASE_URL`**: The HTTPS URL of your deployed backend (e.g. `https://tatva-api.onrender.com`).
  * *Note: If left unset, it will default to `http://localhost:8000`.*

### B. Build Configuration
* **Framework Preset**: Vite
* **Root Directory**: `frontend`
* **Build Command**: `npm run build`
* **Output Directory**: `dist`
* **Node Version**: `18.x` or `20.x`

### C. SPA Router Redirection
Since the frontend uses client-side routing (`react-router-dom`), ensure the hosting provider redirects all routes to `index.html`:
* **Vercel**: Handled automatically. If needed, add a `vercel.json` in the frontend directory:
  ```json
  {
    "rewrites": [{ "source": "/(.*)", "destination": "/index.html" }]
  }
  ```
* **Netlify**: Create a `_redirects` file in `frontend/public` with this line:
  ```text
  /*    /index.html   200
  ```

---

## 4️⃣ Combined Self-Hosted Deployment (Docker Compose)

If you are deploying both services together on a single VPS (like DigitalOcean, AWS EC2, or Linode), you can use the root `docker-compose.yml` file.

1. Clone the repository to your server.
2. Edit `./backend/.env` with your production API keys and database credentials.
3. Run the compose environment:
   ```bash
   docker compose up -d --build
   ```
4. This command will:
   * Build the Python API and expose it on port `8000`.
   * Compile the React application, copy it into an Nginx web server, and expose it on standard HTTP port `80`.
   * Create a persistent docker volume named `backend_uploads` to securely store your case evidence vaults.

---

## 🔍 Post-Deployment Verification Checklist

1. **Verify Database Connections**:
   * Open the API logs and confirm that you see:
     `[API Startup] Supabase PostgreSQL tables verified/created.`
2. **Access Swagger Docs**:
   * Visit `https://your-backend-domain.com/docs` to ensure the API is live and readable.
3. **CORS Validation**:
   * Verify that the backend CORS middleware permits requests from your frontend production domain. If you deploy the frontend to a custom domain, you can update the origins list in `backend/api/main.py` or customize the CORS middleware to accept your production domain.
