# Retell AI To GoHighLevel n8n Blueprint Runbook

This is a proof-of-approach asset for voice-agent automation proposals. It is not connected to live buyer systems and contains no secrets.

Files:

- `proof/retell_ghl_workflow_demo.html`
- `proof/retell_ghl_n8n_blueprint.json`

## What The Blueprint Demonstrates

The workflow receives a Retell call outcome webhook, normalizes the payload, classifies the outcome route, and maps it to GoHighLevel actions.

Routes covered:

- booked call
- reschedule request
- human handoff
- qualified but not booked
- review required because identity or booking data is missing

## 48-Hour Buyer Milestone

For a real buyer, the first milestone should be:

One Retell call outcome path connected to one GoHighLevel location, with contact upsert, opportunity or task creation, appointment handling where allowed, duplicate prevention, sample payload tests, and handoff documentation.

## Buyer Inputs Needed

- Retell webhook event sample for at least three call outcomes.
- GoHighLevel location ID.
- Pipeline ID and stage IDs.
- Calendar ID if direct appointment creation is in scope.
- Default user/owner ID for handoff tasks.
- Rules for booked, reschedule, not-qualified, and human-handoff outcomes.
- Preferred confirmation behavior.

## Variables To Replace

- `{{GHL_API_TOKEN}}`
- `{{GHL_LOCATION_ID}}`
- `{{GHL_PIPELINE_ID}}`
- `{{GHL_BOOKED_STAGE_ID}}`
- `{{GHL_CALENDAR_ID}}`
- `{{GHL_DEFAULT_USER_ID}}`
- `{{CONTACT_ID_FROM_PREVIOUS_STEP}}`

The last placeholder depends on how the buyer's GHL contact upsert response is shaped. In production, the workflow should map the returned contact ID from the upsert node into task and appointment nodes.

## Safety Notes

- Do not create GHL appointments if required identity or time fields are missing.
- Use Retell `call_id` as the idempotency key so Retell retries do not create duplicate GHL records.
- Keep raw payload logging short and avoid storing sensitive transcript content unless the buyer approves it.
- Put low-confidence or ambiguous outcomes into human review.
- Test with Retell sample payloads before switching live traffic.

## Proposal Framing

Use this line when submitting H55-style proposals:

> I already mapped a compact Retell-to-GHL n8n blueprint that covers booked, reschedule, human-handoff, and missing-data routes. I would adapt that to your exact Retell payloads, GHL pipeline stages, and calendar behavior during the first milestone.
