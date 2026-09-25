# 🛡️ SentinelSync — Multi-Agency Disaster Coordination Platform
> **Problem Statement:** Omni_DisasterMgmt_19 — *"Coordinated Multi-Agency Emergency Communication"*  
> **Enterprise Control Room & EOC Prototype** | FastAPI • SQLite • Leaflet.js • Chart.js • Gemini 2.5 Flash • Groq Whisper

---

## ⚡ Executive Summary

During major urban disasters and multi-casualty incidents, response agencies (Police, Fire & Rescue, Medical/EMS, and Municipal Teams) operate in isolated informational silos. When 911/112 dispatch receives fragmented or contradictory reports regarding the same incident:
- Multiple command centers dispatch redundant units or leave high-severity zones unattended.
- Field responders face a "Fog of War" with conflicting severity assessments and differing location descriptions.
- Local fleet vehicle reserves deplete rapidly, leaving zones unable to treat critical trauma casualties without automated inter-agency mutual aid coordination.

**SentinelSync** is an AI-powered, multi-agency disaster synchronization layer that unifies radio voice dispatches and text reports into a single, real-time Common Operating Picture (COP). It autonomously merges duplicate incident reports, transparently arbitrates conflicting severity classifications using deterministic trust rankings, and coordinates fleet vehicles across mutual-aid zones.

---

## 🎨 Enterprise UI Redesign & Live Multi-Theme Engine

The interface has been redesigned to match state-of-the-art emergency control rooms and Emergency Operations Centers (EOC):

### 1. Three Switchable Operational Themes
Switch instantly using the top-bar theme selector (`⚓ Navy`, `☀️ Light`, `🌙 Dark`):
- **⚓ Operational Navy (Default):** Deep naval blue control-room headers (`#0b1a30`), high-contrast operational card styling, vibrant emergency badges, and optimized contrast for 24/7 command center video walls.
- **☀️ EOC Light:** Clean, ultra-sharp daylight situational awareness layout (`#ffffff` / `#f8fafc`) designed for brightly lit crisis centers, tablets, and incident command posts.
- **🌙 Tactical Dark:** Night cockpit dark mode (`#0a0e17` / `#111827`) with neon telemetry accents designed for low-light field command vehicles.

### 2. Operational Top Grid
- **Live Tactical Map (Left ~68%):** Powered by Leaflet.js with real-time zone boundary circles, live GPS user tracking, and **custom lively callout pins**:
  - `⚠️ MG Road — Vehicle Contention` (Critical red callout)
  - ` इंड Indiranagar — Medical Emergency` (Amber callout)
  - `🛡️ Koramangala — Fire Incident` (Blue callout)
  - Live animated fleet unit markers (`🚙`, `🚑`, `🚒`) showing en-route response vehicles.
  - Interactive layer toggles: `🔴 Critical`, `🟠 Medium`, `🟢 Low`, `🚙 Units`, and `⛶ Center`.
- **Incident Summary (2x2 Grid):** Real-time statistics tiles:
  - 🚨 **Active Incidents**
  - 🚙 **Units Dispatched**
  - ⚠️ **Contention Areas**
  - 🛡️ **Total Units Online**
- **Resource Status Widget:** Animated, real-time fleet capacity progress bars across all response depots:
  - `🚑 Ambulance` (e.g. 8 / 12 available)
  - `🚒 Fire Unit` (e.g. 4 / 6 available)
  - `🚔 Police Unit` (e.g. 10 / 14 available)
  - `🛟 Rescue Unit` (e.g. 3 / 5 available)

### 3. Dual-View Active Incident Feed
Toggle seamlessly between two specialized operational views:
- **📊 Table View (Operational Layout):** Enterprise data ledger featuring `#`, `Severity`, `Location`, `Incident Details`, `Vehicle Requirement`, `Dispatch Status`, `Reports`, and one-click `Action` buttons.
- **🗂️ Cards View (Granular Audit):** Detailed cards displaying full conflict explanation boxes, demand-driven vehicle badges, dispatch timelines, and agency confirmation tags.

### 4. Mutual Aid Vehicle Contention Alert Banner
When a zone exhausts its local vehicle fleet (e.g., ambulances drop to 0), a persistent high-visibility alert banner broadcasts across all agency screens:
```
⚠️ VEHICLE CONTENTION: Zone A — Central (MG Road & Brigade Rd)
0 ambulances available! Incident #b5bb5f75 requires mutual-aid assistance.     [Respond from Zone B →]
```
Clicking **"Respond from Zone B →"** immediately fulfills the mutual aid request from the neighboring depot, decrements the helper depot, updates the incident dispatch record, and **automatically turns off the contention banner across all connected screens**.

---

## 🏗️ Architecture & Processing Pipeline

```
                     [Live Ingestion: Radio Audio or Text]
                                       │
            ┌──────────────────────────┴──────────────────────────┐
            ▼                                                     ▼
   [Live Microphone / WAV]                                 [Typed Text Entry]
            │                                                     │
  [Groq Whisper STT API]                                          │
            │                                                     │
            └──────────────────────────┬──────────────────────────┘
                                       ▼
                       [Google Gemini 2.5 Flash / NLP]
                     (Extract: Type, Location, Sev, Demand)
                                       │
                                       ▼
                        [Real-World Geocoding Engine]
                 (Live GPS Bias + Landmark Gazeteer + GeoQ)
                                       │
                                       ▼
                     [Deduplication & Merge Engine (§7.1)]
                    (Matches OPEN Incidents: ≤500m & ≥0.75)
                                       │
                     ┌─────────────────┴─────────────────┐
                     ▼                                   ▼
             [Match Identified]                 [No Match Found]
                     │                                   │
          [Merge into Active Incident]         [Create New Incident]
                     │                                   │
                     └─────────────────┬─────────────────┘
                                       ▼
                     [Deterministic Conflict Arbitration]
                    (Trust: EMS 0.95 > Fire 0.90 > Police 0.80)
                                       │
                                       ▼
                    [Demand-Driven Fleet Allocation (§7.4)]
                     (Local Dispatch or Mutual-Aid Contention)
                                       │
                                       ▼
                   [Real-Time WebSocket & Dashboard Broadcast]
```

---

## 👥 Multi-Agency Roles & Trust Matrix (§4)

SentinelSync enforces strict role separation and deterministic authority weighting:

| Role | Default User | Assigned Zone | Trust Weight | Fleet Visibility | Dispatch Authority |
|---|---|:---:|:---:|:---:|:---:|
| **🚑 Medical (EMS)** | Dr. Anita / Dr. Jacob | Zone A / Zone B | **0.95** (Highest) | ✅ Yes (Own & Neighboring) | Ambulances |
| **🚒 Fire & Rescue** | Capt. Rao | Zone A | **0.90** (High) | ✅ Yes (Own & Neighboring) | Fire & Hazmat Units |
| **👮 Police** | Officer Sharma | Mobile (Zone A) | **0.80** (Baseline) | ❌ Hidden (Per §4 spec) | Patrol Units |
| **📱 Citizen (112)** | Rahul K. | Mobile Public | **0.40** (Eyewitness) | ❌ Hidden | None |

> **Development Authentication:** Click any agency card in the login modal. Any password (e.g. `sentinel2026`) is accepted for immediate testing.

---

## 💻 Terminal Commands & Execution Guide

### 1. Prerequisites
- Python 3.10+ (macOS, Linux, or Windows WSL)
- Node.js (optional, for tooling)

### 2. Environment Setup
```bash
# Clone or navigate to the project directory
cd /Users/rishi/sentinel_sync

# Activate the virtual environment
source .venv/bin/activate

# Install required dependencies
pip install fastapi uvicorn pydantic requests groq google-genai
```

### 3. Running the Server in Terminal
Run Uvicorn with auto-reload:
```bash
uvicorn backend.main:app --host 0.0.0.0 --port 8080 --reload
```
The server will start on: **`http://localhost:8080`**

### 4. Freeing Port 8080 (If ever occupied)
If port 8080 is ever in use by a stale process:
```bash
lsof -ti :8080 | xargs kill -9
```

### 5. Resetting Database to Clean Demo State
You can reset the database at any time via curl or the dashboard:
```bash
curl -X POST http://localhost:8080/api/demo/reset
```

---

## 🎬 Guided 7-Step Competition Demo Script (§14)

At the top of the dashboard, the 7-step horizontal pipeline stepper guides evaluators through the complete multi-agency disaster lifecycle:

```
[1 Police Report] ── [2 Fire Merge] ── [3 Dispatch to 0] ── [4 Flag Contention] ── [5 Cross-Zone] ── [6 Officer Audit] ── [7 Resolve]
```

### Step 1: Police Accident Report
- **Action:** Officer Sharma calls dispatch reporting a two-car collision at MG Road Junction.
- **Under the Hood:** Audio is transcribed via Groq Whisper STT, extracted via Gemini 2.5 Flash, and creates a new OPEN incident (Severity: **MEDIUM**, Demand: 1 Ambulance).
- **Result:** Incident `#inc_...` appears on the map and table view.

### Step 2: Fire Dept Merge & Escalation (45 Min Later)
- **Action:** 45 minutes later, Capt. Rao (Fire Dept) reports a severe flammable fuel leak at the same junction.
- **Under the Hood:** Merge engine computes Haversine distance ($0\text{m} \le 500\text{m}$) and semantic similarity ($0.88 \ge 0.75$). Reports are merged without duplicate ticket generation.
- **Conflict Arbitration:** Because Fire Dept trust ($0.90$) outranks Police ($0.80$), severity escalates to **CRITICAL/HIGH** with an explainable audit record:
  > *"Severity escalated to HIGH — Fire Dept override on flammable fuel leak hazard (10:45), overriding Police's initial MEDIUM assessment (10:00)."*

### Step 3: Medical Dispatches Sole Local Ambulance
- **Action:** Dr. Anita (EMS Zone A) dispatches the only available local ambulance to the trauma site.
- **Under the Hood:** Zone A ambulance depot decrements from $1 \rightarrow 0$. The incident vehicle badge updates to `1 Ambulance (on-scene)`.

### Step 4: 2nd Incident Triggers Vehicle Contention
- **Action:** A second emergency occurs at Brigade Road requiring an ambulance.
- **Under the Hood:** Zone A has $0$ ambulances available. The dispatcher flags **Vehicle Contention**.
- **Broadcast:** A flashing red alert banner appears across all agency consoles requesting cross-zone mutual aid.

### Step 5: Zone B Fulfills Mutual Aid Contention
- **Action:** Dr. Jacob (EMS Zone B Depot) clicks **"Respond from Zone B →"**.
- **Under the Hood:** Helper Zone B dispatches 1 ambulance ($4 \rightarrow 3$). The incident record records: *"1 ambulance dispatched from Zone B (MUTUAL AID)"*.
- **Contention Auto-Off:** The mutual-aid contention flag is fulfilled and the red alert banner **automatically turns off across all screens**.

### Step 6: Submitter "My Reports" Audit
- **Action:** Switch to Police Officer Sharma and open the *"My Reports"* tab.
- **Under the Hood:** Officer Sharma audits his initial report and sees:
  - Report successfully linked and merged into Incident `#inc_...`.
  - Severity escalated to HIGH by Fire Dept due to fuel leak hazard.
  - Cross-zone ambulance assistance confirmed from Zone B.

### Step 7: Incident Resolution
- **Action:** Incident commander marks the incident handled/resolved.
- **Under the Hood:** Incident status transitions to `RESOLVED`. Pin color updates to gray/green, auto-merging is deactivated, and active incident counters decrement.

---

## 🔌 API Reference & Endpoints

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/api/incidents` | Fetch all active/resolved incidents with role-based filtering |
| `POST` | `/api/reports` | Ingest and process a new emergency report (voice/text) |
| `GET` | `/api/resources` | Fetch real-time depot counts across Zone A, Zone B, and Zone C |
| `POST` | `/api/resources/dispatch` | Dispatch local vehicle (Ambulance, Fire Unit, Patrol) |
| `POST` | `/api/resources/contention/flag` | Flag mutual-aid contention when depot is depleted |
| `POST` | `/api/resources/contention/fulfill` | Fulfill cross-zone mutual-aid request from helper zone |
| `POST` | `/api/stt/transcribe` | Transcribe live microphone audio using Groq Whisper |
| `GET` | `/api/analytics/figures` | Retrieve data for severity donut and depot bar charts |
| `POST` | `/api/demo/step/{n}` | Execute specific demo step ($n \in [1..7]$) |
| `POST` | `/api/demo/reset` | Cleanly reset database to baseline state |
| `WS` | `/ws` | Live WebSocket channel broadcasting real-time updates |

---

## 🛡️ Robustness & Fallback Design

1. **Zero-Failure Offline NLP Fallback:** If internet access drops or API quotas are exceeded, the platform seamlessly switches to a rule-based regex and TF-IDF matcher with zero downtime.
2. **GPS Accuracy Assurance:** When GPS permissions are granted, `navigator.geolocation` provides high-accuracy hardware coordinates ($\pm 5\text{m}$) and dynamically anchors zone depots around the tester's real position.
3. **Database Concurrency:** SQLite is initialized with WAL (Write-Ahead Logging) and robust connection timeouts to prevent database locks during parallel multi-agency submissions.

---

## 📄 License & Attribution
Developed for the Omni Disaster Management Competition 2026. Built with FastAPI, SQLite, Leaflet.js, Chart.js, Groq Whisper STT, and Google Gemini 2.5 Flash.
