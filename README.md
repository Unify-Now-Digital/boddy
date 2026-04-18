# Gym contact-form outreach

n8n workflow: Clay → Skyvern → HubSpot + reply capture. Three flows in one importable file.

## Import

1. n8n → Workflows → **Import from File** → `workflow.json`.
2. Create credentials:
   - **Skyvern API key** — Header Auth. Name: `x-api-key`. Value: `<your key>`.
   - **HubSpot token** — Header Auth. Name: `Authorization`. Value: `Bearer <private-app-token>`. Scopes: `crm.objects.companies.read`, `crm.objects.companies.write`, `crm.objects.notes.read`, `crm.objects.notes.write`.
   - **Outreach inbox** — Gmail OAuth2 on the mailbox used in form submissions.
3. Env vars (Settings → Variables):
   - `ALLOWED_COUNTRIES` — comma-separated ISO codes, e.g. `US,UK,DE,CA`.
   - `N8N_BASE_URL` — public HTTPS URL n8n is reachable at (Skyvern posts callbacks here). No trailing slash needed.
   - `OUTREACH_FROM_EMAIL` — the address used in form submissions (also used to filter your own domain out of reply capture).
4. In HubSpot → Settings → Properties → Company properties, create three custom properties (all **Single-line text**):
   - `outreach_source`
   - `skyvern_task_id`
   - `skyvern_status`
5. Make sure `domain` is set as the unique identifier on Company (it is by default).
6. Activate the workflow.

## Test

```
curl -X POST https://<your-n8n>/webhook/clay-gym-batch \
  -H 'Content-Type: application/json' \
  -d '{
    "items": [{
      "company_name": "Iron Temple Gym",
      "website": "https://irontemplegym.com/",
      "contact_form_url": "https://irontemplegym.com/contact",
      "country": "US",
      "first_name": "Alex"
    }]
  }'
```

Expected: HTTP 202 `{"ok":true,"accepted":1}` immediately. HubSpot company appears with `skyvern_status=queued`; Skyvern callback later flips it to `completed`/`failed` and appends an outcome note.

## The three flows

- **Flow A — Clay webhook in.** POST `{ "items": [ { company_name, website, contact_form_url, country, first_name? }, ... ] }` to `/<base>/webhook/clay-gym-batch`. Responds **202 immediately** (onReceived) so Clay isn't blocked. Per item: normalize domain → dedupe against existing HubSpot companies that already have a `skyvern_task_id` → country filter → render message → Skyvern queue → HubSpot batch upsert keyed on `domain` → log queued note → 200ms pause.
- **Flow B — Skyvern callback.** Skyvern POSTs to `/<base>/webhook/skyvern-callback` with `{task_id, status, screenshot_url?, failure_reason?}`. Looks up the company by `skyvern_task_id`, PATCHes `skyvern_status`, appends an outcome note. Unknown task_ids are dropped.
- **Flow C — Reply capture.** Gmail Trigger (simple mode) polls `is:unread -from:me` every minute. Parses `From:` header, excludes replies from our own domain, excludes common auto-reply subjects. Matches by **sender domain** against HubSpot company `domain`. On match: logs reply note + sets `skyvern_status=replied`. On miss: unmatched sink. **Every processed message is marked read** so it won't re-log.

## Before launching

- [ ] Confirm Skyvern endpoint path — `POST /api/v1/tasks/` is the v1 shape. If your tenant is v2 (`/api/v2/runs/tasks` or similar), update the URL in **Skyvern: queue task**.
- [ ] Confirm Skyvern's response field name. This workflow reads `task_id` at top level of the POST response; if yours returns `id` or nests it, update `HubSpot: upsert company` and `HubSpot: log queued`.
- [ ] Confirm Skyvern's webhook payload shape matches the assumed `{task_id, status, screenshot_url?, failure_reason?}`. Update **Flow B** field references if not.
- [ ] Test Clay's actual outgoing payload matches `{items: [...]}`. If Clay sends a bare array, wrap or change **Split gyms** to use the root.
- [ ] Seed one real gym through the flow end-to-end before the first batch.
- [ ] Wire the "Unmatched (manual review)" node to Slack / a Google Sheet / a HubSpot unassociated note. Currently a no-op.

## Source tag

Every note this workflow writes contains `[source: contact-form-skyvern]` and every company is tagged with `outreach_source = contact-form-skyvern`. Filter either in HubSpot reports to isolate this channel from email outreach.

## Known limitations

- Batch size: HubSpot free tier is 100 req/10s; this workflow does ~5 HubSpot calls per gym. At 200ms/item pacing we're ~25 req/s peak — OK for small batches (< 50 gyms) but will throttle on larger ones. Either increase the Wait amount or switch HubSpot calls to batch endpoints for scale.
- Association type ID `190` is HubSpot's default `note → company` (HUBSPOT_DEFINED). If your portal uses custom associations, update both note-creation nodes.
- Reply matching is domain-only. A gym that replies from Gmail (e.g. `owner@gmail.com`) won't match and falls into the unmatched bucket.
- Skyvern callbacks are not authenticated in this setup. For production, add a shared-secret query param on `webhook_callback_url` and verify it in Flow B.
