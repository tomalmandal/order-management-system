# Deployment Notes

This document explains the changes made to improve the Docker setup for this project.

---

## What was changed and why

### 1. Added DB health check in docker-compose.yml

**Before:** The backend just waited for the db container to start, not for Postgres to actually be ready.

**Problem:** Postgres takes a few seconds to initialize after the container starts. The backend would try to connect too early and crash on first boot.

**Fix:** Added a `healthcheck` to the db service that runs `pg_isready` every 5 seconds. The backend now only starts once the DB passes this check.

```yaml
db:
  healthcheck:
    test: ["CMD-SHELL", "pg_isready -U ${DB_USER:-admin} -d ${DB_NAME:-orderdb}"]
    interval: 5s
    timeout: 5s
    retries: 5

backend:
  depends_on:
    db:
      condition: service_healthy
```

---

### 2. Added restart policy

**Before:** No restart policy was set on any service.

**Problem:** If the backend crashed for any reason, it would just stay dead until someone manually restarted it.

**Fix:** Added `restart: unless-stopped` to all services so they automatically recover from crashes.

---

### 3. Switched to alpine images

**Before:** Both Dockerfiles used `node:18` which is a large image (~1GB).

**Fix:** Switched to `node:18-alpine` which is much smaller (~180MB). Faster to pull and deploy, less disk usage.

---

### 4. Multi-stage build for frontend

**Before:** The frontend Dockerfile ran the React dev server in the container, which is not suitable for production.

**Fix:** Added a multi-stage build:
- Stage 1 builds the React app with `npm run build`
- Stage 2 serves the static build files using `serve`

This means the final image has no build tools or node_modules - just the compiled static files. Much smaller and faster.

---

### 5. Optimized Docker layer caching

**Before:** Both Dockerfiles copied all files first then ran `npm install`. This means every code change triggers a full `npm install` even if dependencies didn't change.

**Fix:** Copy `package.json` first, run `npm install`, then copy the rest of the code. Docker caches the install layer and only re-runs it when `package.json` changes.

```dockerfile
COPY package*.json ./
RUN npm install
COPY . .
```

---

### 6. Moved DB credentials to environment variables

**Before:** DB credentials were hardcoded in `backend/src/config/db.js`.

**Fix:** The DB config now reads from environment variables, which are passed in through `docker-compose.yml`. Default values are kept for local development so it still works out of the box.

```js
const pool = new Pool({
  user: process.env.DB_USER || 'admin',
  password: process.env.DB_PASSWORD || 'admin123',
  host: process.env.DB_HOST || 'db',
  ...
});
```

---

### 7. Updated docker-compose version

**Before:** `version: '3'` was set which triggers a deprecation warning in newer Docker versions.

**Fix:** Updated to `version: '3.8'` which is the latest stable compose format.

---

## How to run locally

```bash
docker compose up --build
```

- Frontend: http://localhost:3000
- Backend API: http://localhost:3001/api
- Health check: http://localhost:3001/api/health

## Using custom credentials

Create a `.env` file in the project root:

```
DB_USER=myuser
DB_PASSWORD=mypassword
DB_NAME=mydb
```

Docker Compose will automatically pick these up.
