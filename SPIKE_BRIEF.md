# HeatWave Spike Brief: 1–2 Day End-to-End Prototype

**Goal:** Test entire workflow (signup → SMS → report → results) in throwaway prototype. Learn stack. Surface unknowns. Delete at end.

**Timeline:** Day 1 (4–5 hrs) backend, Day 2 (4–5 hrs) PWA + end-to-end test.

**Success Criteria:**
- ✓ Create resident via form → stored in SQLite
- ✓ Send SMS alert to phone (Twilio) with deep link
- ✓ Click SMS link → PWA opens with pre-filled resident_id + event_id
- ✓ Submit thermal indicator (Hot/Cool/Unbearable) + optional adaptation strategy
- ✓ View postcode aggregation results
- ✓ One complete loop: signup → alert → report → results

---

## Tech Stack Overview

- **Backend:** FastAPI (Python) on Ubuntu
- **Database:** SQLite (single file, local)
- **SMS:** Twilio (one-way alerts, webhook for delivery status)
- **Frontend:** PWA (vanilla JS, no build step)
- **Hosting:** Self-hosted Ubuntu + Tailscale (or localhost for spike)
- **URL linking:** Query params (resident_id, event_id) → PWA form pre-fill

---

## Day 1: Backend + SMS (4–5 hrs)

### 1.1 Project Structure (30 min)
```
spike/
├── main.py              # FastAPI app
├── requirements.txt     # pip dependencies
├── database.py          # SQLite setup
├── models.py            # Pydantic schemas
└── .env.example         # Template for Twilio keys
```

### 1.2 Dependencies
```
fastapi
uvicorn
sqlalchemy
twilio
python-dotenv
```

### 1.3 SQLite Schema (30 min)
**Resident table:**
- id (integer, PK)
- phone_last4 (text, hash + last 4 digits)
- postcode (text)
- dwelling_age (text: pre-1980 | 1980-2000 | post-2000)
- tenure (text: own | rent | other)
- has_cooling (boolean)
- created_at (timestamp)

**Report table:**
- id (integer, PK)
- resident_id (FK → Resident)
- event_id (text, e.g., "event_20261001")
- postcode (text, cached from resident)
- thermal_indicator (text: hot | cool | unbearable)
- adaptation_strategy (text, optional)
- created_at (timestamp)

No Device or Alert tables yet—keep it minimal.

### 1.4 FastAPI Endpoints (2–3 hrs)

#### POST /signup
- Input: name, postcode, dwelling_age, tenure, has_cooling, phone_number, sms_consent
- Output: resident_id (UUID or auto-increment)
- Action: Create Resident row, hash phone (store hash + last 4 digits only)
- Error: postcode required, sms_consent must be true

#### POST /report
- Input: resident_id, event_id, thermal_indicator (hot|cool|unbearable), adaptation_strategy (optional)
- Output: { "status": "ok", "results_url": "/results/{postcode}" }
- Action: Create Report row, tied to resident_id + event_id
- Error: resident_id must exist, thermal_indicator must be valid enum

#### GET /results/{postcode}
- Input: postcode
- Output: { "postcode": "3144", "total_reports": 12, "hot": 5, "cool": 4, "unbearable": 3, "strategies": ["AC", "went to library", ...] }
- Action: Aggregate all reports from that postcode (last 7 days), count by thermal_indicator, list distinct adaptation strategies
- Error: postcode not found → return 200 with 0 reports

#### GET /health
- Output: { "status": "ok" }
- Action: Health check (DB connection up, Twilio creds loaded)

### 1.5 Twilio Integration (1 hr)
- Create Twilio account (free trial has $15 credit)
- Load API SID + auth token into .env
- Write send_sms(resident_id, event_id) function:
  ```python
  def send_sms(resident_id, event_id):
      # Fetch resident from DB
      # Build link: http://localhost:8000/pwa?resident_id={id}&event_id={event_id}
      # Send SMS: "Heat alert in your area. How hot is your home? {link}"
      # Log result
  ```
- Test: send SMS to your own phone, verify link resolves to localhost

### 1.6 End-of-Day 1 Checkpoint
- `uvicorn main:app --reload` starts without errors
- POST /signup creates resident in SQLite
- GET /results/{postcode} returns data for test resident
- Send SMS to own phone, verify delivery
- SMS link navigates to http://localhost:8000/pwa?resident_id=1&event_id=event_20261001

---

## Day 2: PWA + End-to-End (4–5 hrs)

### 2.1 PWA Static Files (1 hr)
```
public/
├── index.html           # Landing page (signup form)
├── report.html          # Report form (fetched via link)
├── results.html         # Results view
├── app.js               # Client-side logic
├── styles.css           # Mobile-first CSS
└── manifest.json        # PWA metadata (optional for spike)
```

FastAPI serves static files:
```python
from fastapi.staticfiles import StaticFiles
app.mount("/pwa", StaticFiles(directory="public"), name="pwa")
```

### 2.2 Signup Form (index.html + app.js) (1.5 hrs)
**Form fields:**
- Name (text input)
- Postcode (text input or select)
- Dwelling age (dropdown: pre-1980 | 1980-2000 | post-2000)
- Tenure (radio: own | rent | other)
- Cooling available (yes/no checkbox)
- Phone number (text input, masked)
- SMS consent (checkbox: required, "I agree to receive heat alerts")

**On submit:**
- POST /signup with form data
- On success: show "Thank you! QR code to share" + unique signup link
- On error: show error message

### 2.3 Report Form (report.html + app.js) (1.5 hrs)
**URL params:** ?resident_id=X&event_id=Y (from SMS link)

**Pre-fill logic:**
- Parse URL params
- Fetch resident data (GET /resident/{resident_id} endpoint—add this to Day 1)
- Display: "Welcome back, [name]! Your postcode: [postcode]"

**Form:**
- "How hot is your home right now?" (3 buttons: Hot | Cool | Unbearable)
- "What's helping you stay safe?" (optional text area: "AC", "open windows", "went to library", etc.)

**On submit:**
- POST /report with resident_id, event_id, thermal_indicator, adaptation_strategy
- On success: show results page (/results/{postcode})
- On error: show error message

### 2.4 Results View (results.html) (1 hr)
**Display:**
- Postcode name + summary stats
- Bar chart or text: "X reported hot, Y cool, Z unbearable"
- "What's helping in your area?" (list of adaptation strategies + frequency)

**Simple implementation:**
- Fetch GET /results/{postcode}
- Render text summary (no fancy charts for spike)

### 2.5 CSS + Mobile UX (1 hr)
- Mobile-first (320px+), no desktop-only features
- Button-heavy (easy for older users)
- Large touch targets (44px minimum)
- Clear spacing, readable fonts
- No complex CSS Grid (keep simple)

### 2.6 End-of-Day 2: Full Loop Test
1. **Signup:** Open http://localhost:8000/pwa → fill form → submit
2. **SMS Alert:** Trigger send_sms(resident_id, event_id) manually or via Python REPL
3. **Click Link:** Copy SMS link to browser
4. **Report:** Form pre-fills with resident data → select thermal indicator → submit
5. **View Results:** See postcode aggregation
6. **Verify DB:** Inspect SQLite to confirm rows exist

---

## Thermal Indicator Scale (from HeatWave spec)

**Three-point subjective scale:**
- **Unbearable** — cannot be outside, urgent need for cooling, health at risk
- **Hot** — uncomfortable, need to take action (find shade, hydrate, reduce activity)
- **Cool** — manageable, not a concern for this event

**Rationale:** Simple (no numerical confusion), accessible to non-technical users, matches lived experience (not temperature). Consistent with adaptive resilience framing (how people *feel*, not what thermometer says).

**Adaptation strategies (open text):** Examples: "used AC", "went to library", "stayed indoors", "opened windows", "went to cooling center", "drank water", "took a cold shower". Freeform—no validation needed for spike.

---

## Key Assumptions to Test

1. **PWA deep linking:** Does URL param passing work smoothly? (resident_id from SMS link → pre-filled form)
2. **Twilio SMS delivery:** How long does SMS take? Does link click-through work?
3. **SQLite performance:** Is it fast enough for 500+ residents, 1500+ reports?
4. **Device UUID tracking:** Can you reliably get device_uuid from browser localStorage?
5. **Postcode aggregation:** How do you handle duplicate postcodes (same person reports twice from different locations)?

---

## Known Unknowns (to surface during spike)

- [ ] CORS issues (API on :8000, static files on :8000 — should be same origin, probably OK)
- [ ] Twilio sandbox limitations (may need to verify phone number first)
- [ ] SQLite locking (if multiple concurrent reports, will it handle it?)
- [ ] URL length (SMS links with resident_id + event_id — will they be short enough?)
- [ ] Mobile browser caching (will PWA cache form data across page reloads?)

---

## Throwaway Mindset

**You will delete this after Day 2.** Do NOT:
- Optimize for production
- Add error handling everywhere (basic checks OK, but don't overdo it)
- Build reusable components (write inline, duplicate code)
- Design for scaling (SQLite is fine, no database migration planning)
- Document extensively (inline comments only)

**DO:**
- Test assumptions ruthlessly
- Note what works and what's clunky
- Write down gotchas
- Start fresh Week 1 with newfound knowledge

---

## Success Definition

By end of Day 2 afternoon, you can:
1. Walk through signup → SMS → report → results in ~5 minutes
2. Identify one thing that felt clunky (and why)
3. Identify one thing that felt smooth (and why)
4. Have proof that the tech stack pieces fit together
5. Have enough confidence to start the real MVP Week 1 knowing what to expect

Then delete the spike code. Start fresh.
