# Make scenario blueprints — BODDY intake

Three best-effort Make blueprints. **Not guaranteed to import cleanly** —
see "Caveats" below. Build order: worker first (it has the webhook the
two trigger scenarios post to).

## Files

- `worker.blueprint.json` — `BODDY Intake: Worker`
  Webhook → HTTP (Anthropic Messages, PDF + web_search) → Set variable
  (parse JSON) → ClickUp create task.
- `gmail-fit-trigger.blueprint.json` — `BODDY Intake: .fit trigger`
  Gmail Watch Emails (Signed label, partnerships .fit) → iterate
  attachments → filter PDFs → HTTP POST to worker webhook.
- `gmail-tech-trigger.blueprint.json` — same as above, .tech account.

## Import steps

1. In Make: **Scenarios → Create new scenario → … → Import Blueprint**.
2. Start with `worker.blueprint.json`.
3. Open the **Webhook** module → click **Add** → create a new webhook
   (name it `boddy-intake-worker`). Copy the webhook URL.
4. Open the **HTTP** module → replace `__TODO_ANTHROPIC_API_KEY__` in
   the `x-api-key` header with your Anthropic key (or swap the module
   for the Anthropic Claude app if you've added your key there —
   note the Claude app doesn't expose `tools`, so this HTTP module
   must stay as HTTP for web search to work).
5. Open the **ClickUp** module → pick your ClickUp connection, then
   select Workspace / Space / Folder / List. Replace the four
   `__TODO_CLICKUP_*__` placeholders with the resolved IDs.
6. Save + activate the worker.
7. Import `gmail-fit-trigger.blueprint.json`. Open the **Gmail** module,
   pick the `.fit` connection, confirm the folder shows **Signed**.
   Open the **HTTP** module and paste the worker webhook URL from
   step 3 into `url` (replacing `__TODO_WORKER_WEBHOOK_URL__`). Save
   + activate.
8. Repeat step 7 for `gmail-tech-trigger.blueprint.json` with the
   `.tech` connection.

## Caveats

Make blueprints bake in account-private IDs that I cannot know:

- **Connection IDs** (`__IMTCONN__`) — every module that auths against
  Gmail, ClickUp, etc. will come in unselected. Click the connection
  dropdown on each and pick the right one.
- **Webhook ID** — the worker's webhook is a placeholder; you create a
  fresh one on import.
- **ClickUp List / Space / Folder / Team IDs** — placeholders; re-select.
- **Module version numbers** — pinned to what the blueprint author's
  account had at export time. If Make refuses a module version, delete
  that module and re-add its same-named replacement; all field values
  will be copied if you choose "Keep mapping".
- **Zone** — set to `eu1.make.com`. If your account is on `us1` or
  `us2`, edit the `zone` field in each file before import, or just
  ignore (zone is informational for import).
- **Expression syntax** — blueprint uses Make's `{{1.field}}` and
  `{{parseJSON(...)}}` syntax. If you renumber modules (e.g. insert
  a module between 1 and 2), re-check references.

If any file fails validation on import, the fallback is to build that
scenario by hand using the plan in `/root/.claude/plans/` — the prompt,
body JSON, and field mappings in these blueprints are the slow-to-type
parts, so even copy-pasting individual values out of these files saves
time.

## After import — sanity run

1. Activate the worker only.
2. POST a test payload to its webhook URL with `curl`:
   ```bash
   curl -X POST "$WORKER_WEBHOOK_URL" \
     -H "content-type: application/json" \
     -d '{
       "pdf_base64":   "<base64 of a real partner PDF>",
       "pdf_filename": "test.pdf",
       "sender_email": "test@example.com",
       "subject":      "Manual worker test"
     }'
   ```
3. Verify a ClickUp task appears with a 150–200 word description and
   non-empty facilities/activities arrays.
4. Then activate the two Gmail triggers.
