# Gym contact-form outreach — prototype

n8n workflow: Clay → Skyvern → HubSpot, plus reply capture.

## Import

1. n8n → Workflows → **Import from File** → `workflow.json`.
2. Create credentials:
   - `Skyvern API key` — Header Auth, `x-api-key: <your key>`
   - `HubSpot token` — Header Auth, `Authorization: Bearer <private-app-token>`
   - `Outreach inbox` — Gmail OAuth2 for the reply inbox
3. Env vars (Settings → Variables, or `.env`):
   - `ALLOWED_COUNTRIES` — e.g. `US,UK,DE,CA`
   - `N8N_BASE_URL` — public URL n8n is reachable at (for the Skyvern callback)
   - `OUTREACH_FROM_EMAIL` — the address used in form submissions
4. In HubSpot, create three custom company properties: `outreach_source`, `skyvern_task_id`, `skyvern_status`.
5. Activate the workflow.

## Three flows in one file

- **Flow A — Clay webhook in.** POST a batch of gyms to `/<base>/webhook/clay-gym-batch` shaped `{ "items": [ { company_name, website, contact_form_url, country, first_name? }, ... ] }`. Filters by country, renders the message, queues Skyvern, upserts the HubSpot company, logs the submission.
- **Flow B — Skyvern callback.** Skyvern POSTs to `/<base>/webhook/skyvern-callback` when the task finishes. Looks up the company by `skyvern_task_id` and appends the outcome (+ screenshot URL if Skyvern returned one).
- **Flow C — Reply capture.** Gmail poll on the outreach inbox. Matches replies to HubSpot **by sender domain** (not by the created contact, per the note in the brief). Unmatched replies fall into a bucket for manual review — wire to Slack/Sheet later.

## What's stubbed and what to tighten before real runs

- The Skyvern request body assumes the v1 `/api/v1/tasks` shape with `navigation_goal` + `navigation_payload`. Some tenants use `create-task-v2`; check and swap the URL if so.
- HubSpot company upsert here is a plain `POST /companies`; for true idempotency swap to the `upsert` endpoint keyed on `domain` (HubSpot's default dedupe property) once the custom property is created.
- Activity logging uses `notes` with `associationTypeId: 190` (note → company). That's the cleanest activity for a prototype; switch to the `engagements` API if you want it to appear as an "Email" or "Form submission" on the timeline.
- The message template is inline in **Render message**. Swap for a proper template when you've tested one version end-to-end.
- The unmatched-reply branch is inert. Decide: Slack notification, Sheet row, or HubSpot unassociated note.

## Source tag

Every activity this workflow writes carries `[source: contact-form-skyvern]` in the body and the company has `outreach_source = contact-form-skyvern`. Filter on either in HubSpot reports to isolate this channel.
