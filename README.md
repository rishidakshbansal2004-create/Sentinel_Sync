# SentinelSync
### Coordinated Multi-Agency Emergency Communication
**Problem Statement:** Omni_DisasterMgmt_19 — "Coordinated Multi-Agency Emergency Communication"

---

## 1. Problem Statement (as given)

Multiple agencies responding to large-scale emergencies often struggle to communicate effectively with each other. Police, Fire, Medical, and Municipal teams each receive and report the same incident independently — different wording, different formats, no shared record — and dispatchers manually piece together fragmented, sometimes contradictory reports under time pressure.

## 2. One-Line Pitch

SentinelSync is an AI-powered coordination layer that ingests emergency reports from multiple agencies, automatically understands and structures them, merges duplicate reports of the same incident, intelligently resolves conflicting information between sources, and helps agencies coordinate scarce resources (ambulances, fire units) across zones — all through one live, shared dashboard.

## 3. Core Differentiators (why this is not "just another dashboard")

1. **Understanding, not just messaging.** Most submissions in this space are shared-inbox/chat clones. SentinelSync turns messy, multi-source, differently-worded reports into one clean, structured, trustworthy incident record.
2. **Explainable conflict arbitration.** When agencies disagree (e.g. on severity), the system doesn't average or silently pick one — it applies a transparent, trust-weighted rule and explains its reasoning in plain language.
3. **Self-contained resource coordination.** Rather than depending on external fleet-tracking APIs that don't exist/aren't public, the platform itself becomes the live source of truth for resource availability, updated by the agencies using it.
4. **Human-in-the-loop, not fully automated.** Cross-zone resource escalation ("Vehicle Contention") is a manual action taken by a human dispatcher, not an automatic reallocation — the system surfaces the right information to the right people at the right time, but humans stay in control of dispatch decisions.
5. **Multi-channel input.** Reports can be typed or spoken (via Groq Whisper STT), and location can be explicit (named landmark, geocoded) or inferred (live device location) — mirroring how real-world agency reporting actually happens (radio, calls, mobile apps).

---

## 4. User Roles & Access Model

| Role | Location basis | Can submit reports | Can set severity | Can request resources | Sees resource counts | Can dispatch resources |
|---|---|---|---|---|---|---|
| **Fire** | Zone-assigned + live geolocation (§8.2) | ✅ | ✅ (trust: 0.9) | ✅ (self-serves) | ✅ own zone | ✅ |
| **Medical** | Zone-assigned + live geolocation (§8.2) | ✅ | ✅ (trust: 0.95) | ✅ (self-serves) | ✅ own zone | ✅ |
| **Police** | Live geolocation (per report) | ✅ | ✅ (trust: 0.8) | ✅ (request only) | ❌ | ❌ |
| **Citizen** | Live geolocation (per report) | ✅ | ✅ (trust: 0.4) | ❌ | ❌ | ❌ |

**Key design rationale:**
- Fire/Medical own a fixed zone's resource pool (matches how real depots work — resources belong to a station), but their *visibility* into incidents extends beyond their home zone whenever their device is physically located in another zone (e.g. responding to a contention flag and traveling there — see §8.2, §8.3).
- Police/Citizens are mobile — their live device location anchors their report's geocoding and their incident visibility.
- Citizens cannot request resources directly (prevents abuse/spam of the resource system by unqualified users) — but they *can* manually link their report to an existing incident (see §7.3).
- Trust weights (in parentheses above) drive the severity-conflict arbitration logic (see §7.2). These are config values, not learned/dynamic.

---

## 5. Full Pipeline — Step by Step

```
[Report Input: text or voice]
        ↓
[If voice → Groq Whisper STT → transcript]
        ↓
[LLM Extraction (Gemini) → structured schema]
        ↓
[Location Resolution: geocode location_text, biased by live device location or zone]
        ↓
[Dedup/Merge Check against OPEN incidents: geo-distance ≤500m + semantic similarity ≥0.75]
        ↓
   ┌────┴────┐
   │         │
[Match]   [No Match]
   │         │
[Merge into    [Create new
existing        incident,
incident]       status: OPEN]
   │         │
   └────┬────┘
        ↓
[Conflict Arbitration: severity/incident-type resolved by trust weight if disagreement]
        ↓
[Resource request handling: union all requests from Fire/Medical/Police; citizens excluded]
        ↓
[Dashboard updates in real time for all relevant agencies (WebSocket/SSE)]
```

---

## 6. Data Extraction Schema

Every report — text or transcribed voice — is passed through the LLM extraction step and normalized into this structure:

```json
{
  "report_id": "uuid",
  "incident_id": "uuid or null (set after merge/creation)",
  "source_agency": "police | fire | medical | citizen",
  "source_user_id": "uuid",
  "raw_text": "original submitted text or transcript",
  "incident_type": "road_accident | fire | flood | medical_emergency | other",
  "location_text": "as extracted from report, e.g. 'MG Road junction'",
  "lat": 0.0,
  "lng": 0.0,
  "geocode_confidence": "ROOFTOP | APPROXIMATE | failed",
  "severity": "low | medium | high",
  "resources_requested": ["ambulance", "fire_unit"],
  "timestamp": "ISO8601",
  "status": "submitted | merged_new_incident | merged_existing_incident | severity_confirmed | severity_overridden | geocode_failed"
}
```

### Incident record (post-merge aggregate)

```json
{
  "incident_id": "uuid",
  "incident_type": "road_accident",
  "lat": 0.0,
  "lng": 0.0,
  "zone": "MG_Road_Area",
  "status": "OPEN | RESOLVED",
  "arbitrated_severity": "high",
  "severity_history": [
    {"agency": "police", "severity": "medium", "timestamp": "10:00", "trust": 0.8},
    {"agency": "fire", "severity": "high", "timestamp": "10:45", "trust": 0.9, "overrode": true}
  ],
  "confirming_agencies": ["police", "fire"],
  "resources_requested": ["ambulance", "fire_unit"],
  "resource_status": [
    {"type": "ambulance", "status": "dispatched", "dispatched_by": "zone_b_medical", "contention": true}
  ],
  "summary": "LLM-generated one-paragraph dispatch brief",
  "created_at": "ISO8601",
  "resolved_at": null
}
```

---

## 7. Core Logic Modules

### 7.1 Deduplication / Merge Logic

A new report is merged into an existing **OPEN** incident if:
- `geo_distance ≤ 500m` (haversine formula on lat/lng)
- `semantic_similarity ≥ 0.75` (cosine similarity on embeddings of incident_type + location_text + description)
- Incident is not `RESOLVED`

**Important:** there is **no hard time cutoff** for merging (see §7.5 for rationale) — a report arriving hours later can still merge into an OPEN incident, as long as it matches geographically and semantically. A soft 2-hour window is used only to help disambiguate borderline/ambiguous cases, not to hard-block merges.

```python
def find_matching_incident(new_report, open_incidents):
    for incident in open_incidents:
        if incident["status"] == "RESOLVED":
            continue
        geo_distance = haversine(new_report["lat"], new_report["lng"], incident["lat"], incident["lng"])
        semantic_sim = cosine_similarity(new_report["embedding"], incident["embedding"])
        if geo_distance <= 500 and semantic_sim >= 0.75:
            return incident
    return None
```

### 7.2 Conflict Arbitration ("Fog of War" resolution)

Applies **only** to fields where agencies can genuinely disagree in a way that needs a single resolved value — primarily **severity**, and optionally **incident_type**.

```python
SEVERITY_RANK = {"low": 0, "medium": 1, "high": 2}

def resolve_severity(reports):
    # reports: [{"agency": "fire", "severity": "high", "trust": 0.9, "timestamp": ...}, ...]
    high_trust_reports = [r for r in reports if r["trust"] >= 0.85]
    if high_trust_reports:
        top = max(high_trust_reports, key=lambda r: (r["trust"], r["timestamp"]))
        conflict = any(r["severity"] != top["severity"] for r in reports)
        return top["severity"], conflict, top["agency"]
    else:
        top = max(reports, key=lambda r: SEVERITY_RANK[r["severity"]])
        return top["severity"], False, top["agency"]
```

**Explanation string generation (shown in UI):**
> "Severity escalated to HIGH — Fire Dept override on fuel leak risk (10:45), overriding Police's initial MEDIUM assessment (10:00)."

This is generated via a short LLM prompt fed the severity_history array, or can be templated directly from the data without an LLM call if time is short.

**Explicitly NOT trust-weighted:**
- **Resources requested** → union of all requests from Fire/Medical/Police (never dropped/overridden — under-provisioning is more dangerous than over-requesting). Citizens cannot contribute to this field at all.
- **Merge/dedup decision** → based purely on geo+semantic similarity, not trust.
- **Location** → resolved by geocode confidence, not agency trust.

### 7.3 Manual Report Linking (bypasses automatic matching)

A user (including Citizens) can manually select "link this report to Incident #X" instead of relying on automatic dedup. This:
- Skips the geo/semantic similarity check entirely (human confirmation is sufficient)
- Adds the report to the incident's confirming sources
- Still respects role-based field permissions (citizen reports still cannot populate `resources_requested`)

Use case: catches cases where auto-matching might miss a genuine match (paraphrased very differently, just outside 500m, etc.), and gives citizens a lightweight way to say "I'm also seeing this" without full agency-level reporting rights.

### 7.4 Resource Contention & Vehicle Contention Flag

**Normal dispatch:**
```python
def dispatch_resource(incident_id, resource_type, zone):
    if resource_pool[zone][f"{resource_type}s_available"] > 0:
        resource_pool[zone][f"{resource_type}s_available"] -= 1
        update_incident(incident_id, resource_status="dispatched", dispatched_by=zone)
    else:
        return "NO_AVAILABILITY"  # triggers contention flag option in UI
```

**Vehicle Contention flag (manual, agency-triggered):**
1. Zone A's Fire/Medical team has 0 available of the needed resource.
2. They click **"Flag Contention"** on the incident.
3. The incident is pushed to **neighboring zones'** dashboards, labeled: *"⚠️ [Zone A] — Ambulance Contention — assistance requested."*
4. Any neighboring zone's Fire/Medical team can review and click **"Send Ambulance"** (or relevant resource) if they have availability.
5. On dispatch:
   - **Helper zone's** resource count decrements.
   - Incident updates: *"1 ambulance dispatched from Zone B."*
   - **The contention flag is removed from all other zones' dashboards** — only Zone A (original) and Zone B (helper) retain visibility into this incident going forward. No other zone can double-dispatch to an already-resolved contention.

```python
def flag_contention(incident_id, zone, resource_type):
    incident.contention_flags.append({"resource_type": resource_type, "origin_zone": zone, "status": "open"})
    broadcast_to_neighboring_zones(incident_id, zone, resource_type)

def fulfill_contention(incident_id, resource_type, helper_zone):
    incident.contention_flags = [
        f for f in incident.contention_flags if f["resource_type"] != resource_type
    ]  # closes the flag
    remove_from_dashboards(incident_id, exclude_zones=[incident.origin_zone, helper_zone])
    dispatch_resource(incident_id, resource_type, helper_zone)
```

### 7.5 Incident Lifecycle (OPEN / RESOLVED)

**Why no hard time cutoff on merging:** early design used a 30-minute merge window, but this breaks a realistic scenario — Police reports an accident at 10:00, Fire arrives on-scene and reports a fuel leak at 10:45 (outside the old window). Under the old logic, Fire's report would incorrectly spawn a *new* incident instead of updating the existing one. Fixed by tying merge eligibility to incident **status** (OPEN vs RESOLVED) rather than elapsed time. A 2-hour soft window remains as a secondary signal for ambiguous cases only.

```
OPEN  → new/updates reports can merge in at any time
      → severity can be re-arbitrated as new reports arrive
      → resources can be requested/dispatched/contested
RESOLVED → set manually by a responding agency ("handled")
         → no further auto-merging; would need manual reopening
```

### 7.6 Report Status (per-submitter visibility)

Shown to the individual who submitted a report, in "My Reports":

| Status | Meaning |
|---|---|
| Merged — New Incident Created | No match found; became a new incident |
| Merged — Combined with Existing Incident #X | Matched and merged into an existing incident |
| Severity Confirmed | Their severity assessment matched the arbitrated result |
| Severity Overridden | Their assessment was outranked by a higher-trust source; shows by whom and why |
| Geocode Failed | Location couldn't be resolved; flagged for manual follow-up |

---

## 8. Location Resolution Logic

**Device geolocation is captured on every report submission, for every role** (Police, Citizen, Fire, Medical). What it's used for differs by role — see §8.1 and §8.2.

### 8.1 Determining a report's coordinates (all roles)

**Priority order when resolving a report's coordinates:**
1. **Specific, resolvable landmark in the report text** ("near Lulu Mall, MG Road, Kochi") → geocode directly via Google Geocoding API.
2. **Ambiguous/generic text** ("there's an accident nearby," "fire broke out") → fall back entirely to the **reporting device's live location** (`navigator.geolocation`) as the coordinates.
3. Live device location is always at least used as a **bias anchor** for the geocoding API call in case 1, to correctly disambiguate location names that exist in multiple cities (e.g. "MG Road").

### 8.2 Determining what a user CAN SEE (visibility), Fire/Medical specific

Police and Citizen visibility is always geolocation-based (they have no zone concept — see role table in §4).

Fire/Medical visibility is **zone OR geolocation, combined**:

- **By assigned zone (default):** their home dashboard — own zone's incidents and own zone's resource pool (ambulances/fire units), as defined in §4.
- **By live geolocation (in addition):** if their device is physically located within a *different* zone's area (e.g. a Zone B medical team has driven into Zone A), Zone A's incidents/reports also appear on their dashboard automatically — no manual zone-switching required. They can then directly view and add updates to those reports as if they were a home-zone responder.

```python
def get_visible_incidents(user):
    if user.role in ["fire", "medical"]:
        zone_incidents = get_incidents_by_zone(user.assigned_zone)
        geo_incidents = get_incidents_near(user.current_lat, user.current_lng, radius=2000)  # meters
        return merge_unique(zone_incidents, geo_incidents)
    elif user.role in ["police", "citizen"]:
        return get_incidents_near(user.current_lat, user.current_lng, radius=2000)
```

### 8.3 Relationship to Vehicle Contention (§7.4) — two moments, one continuous flow

These two mechanisms are complementary, covering different points in an incident's timeline:

1. **Vehicle Contention flag (discovery moment):** Zone A has 0 available resources of a needed type, flags it, and it's pushed to neighboring zones' dashboards — *before* anyone has physically traveled anywhere. This is how a neighboring zone finds out help is needed.
2. **Geolocation-based visibility (arrival moment):** once a neighboring zone's team decides to respond and physically travels to Zone A, their device location now falls within Zone A's area, so they automatically gain full visibility into Zone A's incidents/reports (not just the one flagged) and can add direct updates — no separate "join zone" action needed.

**Sequence:** Contention flag alerts a neighboring zone → they decide to respond and travel there → geolocation grants them full access to the zone's incidents once they physically arrive.

### 8.5 Location-Based Search (any logged-in user)

In addition to automatic visibility (§8.2), any authenticated user — regardless of role — can **manually search for reports/incidents by location**, entering a place name or area rather than relying only on their current geolocation or assigned zone.

- Search input is geocoded the same way as report submission (§8.1), then matched against incidents within a reasonable radius (e.g. 2–5km, tunable).
- This is **read access only** — search does not grant the report-linking, resource-viewing, or dispatch permissions defined per role in §4. A Police officer searching a distant zone can *see* incidents there but cannot see that zone's resource counts or dispatch anything, since those permissions remain zone/role-gated as before.
- Requires authentication — this is not a public/unauthenticated feature. Full incident data (severity, resource status, exact location) is only visible to logged-in agency/citizen accounts, not the general public, to avoid exposing sensitive incident details (e.g. medical emergencies at a specific address) openly.

Use the `location_type` field returned by Google (`ROOFTOP` = precise, `APPROXIMATE` = rough) — surface a "location approximate" flag rather than presenting false precision. If geocoding fails entirely, mark report status `geocode_failed` and surface for manual review rather than silently dropping it or guessing.

```python
params = {
    "address": location_text,
    "location": f"{device_lat},{device_lng}",  # bias, not hard restrict
    "radius": 50000,
    "key": GOOGLE_API_KEY
}
```

**Handling geocode failure/low confidence:** use the `location_type` field returned by Google (`ROOFTOP` = precise, `APPROXIMATE` = rough) — surface a "location approximate" flag rather than presenting false precision. If geocoding fails entirely, mark report status `geocode_failed` and surface for manual review rather than silently dropping it or guessing.

---

## 9. Speech-to-Text (STT)

- **Provider:** Groq API, `whisper-large-v3` (or `-turbo` for speed) — chosen over the browser's Web Speech API for better accuracy on Indian place names/accents and because it's a server-side call (not dependent on the judge's/demo device's browser).
- **Demo approach:** since this is a recorded (not live) submission, use **pre-recorded, pre-tested audio clips** run through the real STT pipeline — not live microphone input. This proves the capability works end-to-end without live-demo risk (background noise, mic permissions, recognition errors).

```python
from groq import Groq
client = Groq(api_key=GROQ_API_KEY)

def transcribe_audio(audio_file_path):
    with open(audio_file_path, "rb") as f:
        transcription = client.audio.transcriptions.create(
            file=f,
            model="whisper-large-v3"
        )
    return transcription.text
```

---

## 10. Dashboard Structure (per logged-in user)

1. **My Reports** — submission history with per-report status (§7.6)
2. **Live Incidents** — merged incident feed: map pin, arbitrated severity + conflict badge, confirming agencies, auto-generated summary, resource status
3. **Submit Report** — text or voice input; auto-attaches role, zone (Fire/Medical) or live location (Police/Citizen)
4. **Resources** (Fire/Medical only) — own-zone resource counts, dispatch actions, contention flag/response UI
5. **Map View** — all active incident pins on a city/state map (Leaflet.js + OpenStreetMap tiles), color-coded by severity; merging incidents visually collapse two pins into one

---

## 11. Team Split

| You — AI/NLP Layer | Teammate — Backend/Platform |
|---|---|
| LLM extraction & normalization (Gemini) | Auth/login + role-zone assignment |
| Groq STT integration | Multi-source ingestion API |
| Dedup logic (geo-distance + semantic similarity) | Incident + resource data store |
| Trust-weighted conflict arbitration + explanation generation | Real-time dashboard (WebSocket/SSE) |
| Geocoding + live-location bias logic | Resource dispatch, contention flag/broadcast logic |
| Auto-generated incident summary ("dispatch brief") | Map rendering (Leaflet) + full dashboard UI |

---

## 12. Data Sources — Honest Accounting (for pitch/Q&A)

| Data | Source for this demo | Production equivalent |
|---|---|---|
| Incident reports | Simulated agency logins submitting typed/pre-recorded voice reports | Real agency personnel, in-field |
| Resource availability | Seeded starting counts per zone, updated live by the platform itself as units are dispatched | Same model — platform becomes system of record, or optionally syncs with agency fleet systems if such APIs exist |
| Location coordinates | Google Geocoding API, live-location-biased | Same |
| Trust weights | Config values (Fire/Medical > Police > Citizen) | Same, possibly refined per-agency over time |

---

## 13. Tech Stack Summary

- **LLM extraction/summarization:** Gemini API
- **STT:** Groq API (Whisper)
- **Geocoding:** Google Maps Geocoding API
- **Embeddings/similarity:** sentence-transformers or Gemini embeddings, cosine similarity (in-memory, no vector DB needed at this scale)
- **Backend:** FastAPI or Node (teammate's choice)
- **Real-time updates:** WebSocket or SSE
- **Map:** Leaflet.js + OpenStreetMap tiles
- **Auth:** simple role/zone-based login (demo-seeded accounts; production would integrate with agency SSO)

---

## 14. Demo Script (suggested flow for recorded submission)

1. Log in as Police (MG Road area) → submit a voice report of an accident → show Groq STT transcribing it live → extraction happens → new incident created, OPEN.
2. Log in as Fire (same zone), submit a follow-up report ~45 min later (scripted) mentioning a fuel leak → show it **merging into the same incident** (not creating a duplicate) despite the time gap → severity escalates MEDIUM → HIGH with the conflict-resolution badge and explanation shown.
3. On the incident, Medical dispatches their 1 available ambulance → count drops to 0.
4. A second incident in the same zone requests an ambulance → 0 available → Medical flags **Vehicle Contention**.
5. Switch to Zone B's Medical login (second window) → contention flashes on their dashboard → they click **Send Ambulance** → Zone B's count drops, original incident updates, contention flag disappears from all other zones.
6. Show the Police officer's "My Reports" view reflecting the full outcome: severity overridden by Fire, ambulance dispatched from Zone B.
7. Close on the map view showing the full picture, and briefly state the honest data-sourcing model (§12) and production extension points (real fleet-sync APIs, agency SSO, live-mic input).

---

## 15. Positioning Statement for Judges

> "Omni_DisasterMgmt_19 describes agencies that struggle to communicate effectively with each other. SentinelSync solves that at the root — not with another chat window, but by making their existing reports automatically understandable, mergeable, and actionable together, with transparent reasoning behind every arbitration decision and real, functional resource coordination across zones."
