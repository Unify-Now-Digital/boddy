# Venue profile prompt

Source of truth for the prompt used by the `BODDY Intake: Worker` Make
scenario. When the Make scenario's prompt diverges from this file, update
both — this file is the version-controlled copy.

Consumed by a single Anthropic Messages call (`POST /v1/messages`) with:

- a `document` content block containing the partnership agreement PDF
  (base64, `application/pdf`)
- a `text` content block containing the prompt below
- the `web_search_20250305` tool enabled (`max_uses: 5`)

The model is expected to return **only** the JSON object described in the
`OUTPUT` section — no surrounding prose, no markdown fences.

---

## Prompt

```
You are profiling a fitness venue for BODDY. Work in two phases.

=== PHASE 1 — VERIFY ===
From the partnership agreement PDF attached to this message, extract:
  - business_name
  - location_city, location_country
  - contact_email (keep the domain)
  - any website URL mentioned

Use the web_search tool to find the venue's official website. Confirm the
candidate site is the same business by cross-checking against the PDF:
  (a) business name on the site matches (exact or close variant),
  (b) city/country on the site matches the PDF,
  (c) the contact_email domain matches the website domain.

Assign confidence:
  - high   — at least two of (a)(b)(c) match
  - medium — one matches
  - low    — none match, or no website found

=== PHASE 2 — DESCRIBE ===
Only if confidence is "high" or "medium", read the website thoroughly:
home page, about, facilities, classes/services pages. Base every claim
EXCLUSIVELY on what the website says. Do not use the PDF for
facility/activity claims — the PDF is only for verification.

If confidence is "low", skip Phase 2 and return empty description and
empty arrays.

=== OUTPUT ===
Return ONLY this JSON — no markdown, no prose around it:

{
  "business_name": "...",
  "website": "verified URL or null",
  "verification": {
    "confidence": "high" | "medium" | "low",
    "signals_matched": ["name" | "location" | "email_domain"],
    "notes": "one-sentence reason for the confidence level"
  },
  "description": "150–200 word paragraph OR \"INSUFFICIENT_CONTENT\"",
  "facilities": [ /* pick zero or more from Facilities list */ ],
  "activities": [ /* pick zero or more from Activities list */ ]
}

Facilities (closed list — exact spelling):
["Changing Room","Free Weights Area","Cardio Area","Resistance Machines",
 "Functional Area","Studio Room","Indoor Pool","Outdoor Pool",
 "Children's Pool","Hydrotherapy Pool","Sauna","Steam Room","Spa",
 "Beauty Spa","Physiotherapy Room","Tennis Court","Squash Court",
 "Basketball Court","Padel Court","Climbing Wall","Golf Course",
 "Bike Fit Studio","Bike Storage","Crèche","Meeting Room",
 "Business Lounge","Adult Only Area","Café","Restaurant","Snack Bar",
 "Outdoor Training Area","Treatment Rooms"]

Activities (closed list — exact spelling):
["Free Weights","Cardio","Functional Training","Resistance Training",
 "Yoga","Pilates","Reformer Pilates","Spin","Boxing","CrossFit","HIIT",
 "TRX","Barre","Personal Training","Group Classes","Tennis","Squash",
 "Basketball","Padel","Climbing","Golf","Swimming","Mobility",
 "Stretching"]

description rules: neutral travel-guide tone, one paragraph, 150–200
words. Cover in order: (a) venue type & core offering, (b) training
styles & equipment, (c) facilities beyond the gym floor. Optionally
close with one distinctive feature. Exclude pricing, hours, addresses,
phone numbers, contact names, and superlatives ("world-class",
"state-of-the-art", "premier", "elite", "ultimate", "best-in-class").
If the website content is sparse, write a shorter honest paragraph.
```

---

## Closed lists (canonical)

These are duplicated inside the prompt above so the model sees them in
context. The copies below are the canonical reference — if you change
one, change the other.

### Facilities

- Changing Room
- Free Weights Area
- Cardio Area
- Resistance Machines
- Functional Area
- Studio Room
- Indoor Pool
- Outdoor Pool
- Children's Pool
- Hydrotherapy Pool
- Sauna
- Steam Room
- Spa
- Beauty Spa
- Physiotherapy Room
- Tennis Court
- Squash Court
- Basketball Court
- Padel Court
- Climbing Wall
- Golf Course
- Bike Fit Studio
- Bike Storage
- Crèche
- Meeting Room
- Business Lounge
- Adult Only Area
- Café
- Restaurant
- Snack Bar
- Outdoor Training Area
- Treatment Rooms

### Activities

- Free Weights
- Cardio
- Functional Training
- Resistance Training
- Yoga
- Pilates
- Reformer Pilates
- Spin
- Boxing
- CrossFit
- HIIT
- TRX
- Barre
- Personal Training
- Group Classes
- Tennis
- Squash
- Basketball
- Padel
- Climbing
- Golf
- Swimming
- Mobility
- Stretching
