# Ringvox WhatsApp outreach sync (scheduled task prompt)

**Live routine (10 Oct 2026): "Ringvox WhatsApp sync (Ringvox env)", trig_015kG83UxNGtjs7tknu4SFhR, created in the Claude Code Routines page with the Ringvox environment, Sonnet 5.5, 06:47 and 18:47 UTC.** It replaced trig_01P1KqL6w21EtRgE6YLEJWmG, which lost the Ringvox environment when its model was changed through the API (disabled, not deleted). Edit this routine in the app (Routines, the routine, Edit), not through the API. The database is reached through the Supabase connector (the Supabase Management secret in the environment has no database permission).

Parts 3 and 4 below were added to the live routine on 10 Oct.

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

## Part 3: email reply check (every run)
Since = newest outreach.sync_runs row with direction 'reply-check' (or 7 days ago). Read HighLevel conversations for trades contacts with an inbound email since then, plus colm@ringvox.co in Zoho Mail (token refresh at accounts.zoho.eu with only grant_type=refresh_token, then mail.zoho.eu, account 747760000000002002). Classify the newest inbound message: interested, question, not now, stop, out of office, bounce, other.
- interested / question: tags replied, email-interested; stage Replied / Interested; task "Ring {name}: replied to the email" due today; note with the message and a suggested reply in Colm's voice (never sent).
- not now: tags replied, not-now; stage Not now. stop: tag do-not-contact; stage Lost / Unsubscribed. out of office: note. bounce: tag email-bounced, note, and clear the email in outreach.wa_contacts so a mobile-only contact goes back to the WhatsApp lane. other: note.
- Insert a sync_runs row direction 'reply-check'; anyone interested is named in the run's summary so the notification shows them.

## Part 4: one-off email search (only until a sync_runs row direction 'email-finder' with ok = true exists)
Budget 600 Outscraper lookups. Website, then Facebook page, then Google Places to find a website. Junk filter as in task 200. Found: HighLevel email + tags email-found, email-pass2 (and in-wa-sequence if already WhatsApped); outreach.wa_contacts email set. Every contact tried tagged email-search-done. Report: Dropbox ringvox-outreach/email-finder-<date>.md and .csv.
