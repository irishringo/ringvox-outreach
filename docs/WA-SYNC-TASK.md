# Ringvox WhatsApp outreach sync (scheduled task prompt)

Runs in the **Ringvox** cloud environment, twice a day. Keeps HighLevel (the CRM, master record of contacts) and the WhatsApp outreach page's database (Ringvox Supabase, schema `outreach`) in step. Read-only towards everything except the two writes described. Never print credentials. British English in anything written.

## Access (all through the environment's proxy, send no auth of your own)
- HighLevel: `https://services.leadconnectorhq.com`, location `foJKf5RVfBNriDFVQ51h`. Send headers `Version: 2021-07-28`, `Accept: application/json`, `User-Agent: curl/8.5.0`. Never use `/contacts/upsert` with `tags` (it replaces all tags); add tags with `POST /contacts/{id}/tags`, remove with `DELETE /contacts/{id}/tags`.
- Supabase SQL: `POST https://api.supabase.com/v1/projects/pnabqurwfloupeskrxbr/database/query` with JSON body `{"query": "..."}` (Supabase Management Ringvox secret). Use this for every database read and write. Escape strings properly (dollar-quoting is fine).

## Part 1: HighLevel to the outreach database (import)
1. Page through every contact in the location (`POST /contacts/search` with `{"locationId": "...", "pageLimit": 100, "page": n}`, or `GET /contacts/` with `startAfterId`), until done.
2. Keep a contact when ALL of these hold:
   - it has the tag `trades`
   - it has a phone number that is an Irish mobile (+353 83/85/86/87/88/89, or tag `phone-mobile`); Northern Ireland mobiles (+44 7) are fine too
   - it does NOT have any of the tags `customer`, `do-not-contact`, `review-not-a-trade`
3. For each kept contact, normalise the phone to digits only with country code (e.g. `353871234567`) and upsert into `outreach.wa_contacts` ON CONFLICT (phone):
   - always set: `hl_contact_id`, `business_name` (companyName, else contact name), `first_name` (firstName, only if it looks like a person's first name), `trade` (from the `trade-*` tag, capitalised), `county` (from the `county-*` tag or the county field), `website`, `source` = 'highlevel'
   - `email`: set from HighLevel only when the database row has none (never blank out an email the database has)
   - `prior_round`: 'round1' if tagged `round1-contacted`; else 'june' if the phone is in `outreach.prior_sends`; else leave as is. Set `prior_sent_at` from `outreach.prior_sends.sent_at` when present.
   - `priority`: 50 if never messaged before, 80 if 'june', 90 if 'round1' (only set on insert)
   - NEVER overwrite `step`, `next_due`, `last_sent_at`, `outcome`, `outcome_at`, `notes`, `skipped`
4. For rows already in the database whose HighLevel contact now has `customer` or `do-not-contact` (or has been deleted), set `skipped = true`, `skip_reason` = the reason. Do not delete rows.
5. Contacts in `outreach.prior_sends` (June WhatsApp sends) that exist in HighLevel: add the tag `wa-june-contacted` if missing.
6. Insert one row into `outreach.sync_runs` with direction 'import' and a one-line summary (kept, new, updated, skipped).
7. Housekeeping: delete the row `phone = '000test000' and source = 'probe'` and its events if it still exists (a test row).

## Part 2: outreach database to HighLevel (push)
Read `outreach.wa_events` where `hl_synced_at is null` (join `outreach.wa_contacts` for `hl_contact_id`), oldest first. For each event, do the HighLevel side, then set `hl_synced_at = now()` on that event. If a contact has no `hl_contact_id`, try to find it in HighLevel by phone; if still none, leave the event unsynced and mention it in the summary.

- `sent` (step 1/2/3): add tag `wa-sent-1` / `wa-sent-2` / `wa-sent-3`; add a note "WhatsApp message {step} of 3 sent by Colm, {date}". On step 1, move the contact's opportunity in the **Trades Outbound** pipeline to the stage "WhatsApp sent" (create the opportunity at that stage if none exists).
- `sent_undone`: remove the matching `wa-sent-{step}` tag; add a note "WhatsApp message {step} marked as not sent".
- `outcome`:
  - `interested`: tags `replied`, `wa-interested`; opportunity to "Replied / Interested"; create a task for the user assigned to the location "Ring {name}, interested on WhatsApp" due today; note.
  - `not_now`: tag `not-now`; opportunity to "Not now (nurture)"; note.
  - `stop`: tag `do-not-contact`; opportunity to "Lost / Unsubscribed"; note.
  - `not_on_whatsapp` / `wrong_number`: tag `wa-invalid`; note. (Leave the stage.)
  - `signed_up`: tag `signed-up`; opportunity to "Trial / Paying"; note. Do not add `customer` (CJ does that).
  - `cleared`: note "WhatsApp outcome cleared".
  Match stage names loosely (read the pipeline's stages with `GET /opportunities/pipelines?locationId=...`); if a stage doesn't exist, keep the current stage and say so in the summary.
- `email`: if the HighLevel contact has no email, set it (`PUT /contacts/{id}` with only `email`), add tag `email-captured`; if it already has a different one, just add a note with the new address.
- `note`: add (or update the latest WhatsApp-notes note of) a note "WhatsApp notes: {text}". Skip when the text is empty.
- `skip`: tag `wa-skipped`; note with the reason. If the reason is "not a trade", also add tag `review-not-a-trade`.

Insert one row into `outreach.sync_runs` with direction 'push' and a one-line summary (events pushed, failures). If anything failed, set `ok = false` and say what.

## Finish
Write nothing else. No messages to contacts, ever: this task only moves data between the two systems. End with a three-line summary.
