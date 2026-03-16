# S01: API Foundation & Deploy Pipeline

**Goal:** A working API server runs on the Pi, wrapping shell scripts safely, with a repeatable build-deploy pipeline from the dev machine
**Demo:** FastAPI responds to `/api/health`, player control endpoints call `playout_controls.sh` without `shell=True`, Nginx proxies `/api/`, single-command deploy works

## Must-Haves

- FastAPI server starts on Pi, responds to health check at `/api/health`
- Player control endpoints (play, pause, next, prev, volume) call `playout_controls.sh` via subprocess without `shell=True`
- Nginx serves static placeholder page and proxies `/api/` to FastAPI
- Single command on dev machine builds and deploys to Pi via rsync/scp
- SQLite database stores configuration, replacing flat-file reads for settings consumed by the API

## Tasks


## Files Likely Touched

