# BookLeaf: Author Royalty Dashboard & Automated Notification System

Assignment 2 (Full-Stack Developer). Built with **Bubble.io** (app, database, auth), **n8n** (automation backend) and **Google Gemini** (AI-written royalty summary).

Authors log in to see their own books and royalty figures. Admins see every author and can trigger an AI-generated royalty summary, which is calculated in n8n, written by Gemini, returned to Bubble and stored in a Notification Log.

## What is built

**Author portal:** email/password login, "My Books" cards, per-book copies sold / earned / paid / pending, consolidated totals, and a Paid / Pending / Overdue badge per book.

**Admin portal:** author list with city, books, pending royalty and payout status; filters (city, payout status) and search; author detail page; "Generate & Send Royalty Summary" button; "Generate All Pending Summaries" bulk button; notification log with a summary detail page.

**n8n workflow** (`n8n/workflow.json`): Webhook, Calculate Royalty Facts (Code node), Gemini draft, Prepare Result + Status, Bubble callback (disabled, see limitations), Respond to Bubble.

## Architecture

```text
Bubble (author-detail page, button click)
   1. Create NotificationLog (status = Processing, random request_id)
   2. API Connector POST  --->  n8n Webhook
                                   |
                         Calculate Royalty Facts (Code node)
                         totals, payout status, eligibility, due date
                                   |
                         Gemini: writes the email from verified facts only
                                   |
                         Prepare Result + Status (Success / Failed)
                                   |
   3. JSON response  <---  Respond to Bubble
   4. Update NotificationLog (status, generated_content, error_message, completed_at)
   5. Open summary_detail page
```

## Database design (Bubble)

| Type | Fields |
|---|---|
| **Author** | author_id, name, email, phone, city, joined_date |
| **Book** | author (to Author), book_id, title, isbn, genre, publication_date, status, mrp, author_royalty_per_copy, total_copies_sold, total_royalty_earned, royalty_paid, royalty_pending, last_royalty_payout_date, print_partner, available_on (list of text) |
| **NotificationLog** | author (to Author), request_id, status, created_at, completed_at, generated_content, error_message |
| **User** | role (text: admin / author), author (to Author) |

**Why this structure:** Authors and Books are separate types linked by a relation (one author, many books), so an author's data is a single search on `Book.author`. The sample dataset gives royalty figures as per-book totals (no individual sales or payment transactions), so earned / paid / pending are stored on Book as given. The trade-off is that payout history is limited to the last payout date. If transaction data existed, I would add a Payment type linked to Book. NotificationLog is a separate type so every request has its own status and timestamps. Admin versus author access is decided by `User.role`, and `User.author` links a login to its Author record.

## Bubble and n8n integration

**Full payload instead of a Data API lookup.** The Bubble button sends the author and all of their books in one request. I chose this over having n8n query the Bubble Data API because (1) the data is already on the page, so no second network call can fail, (2) n8n does not need a Bubble API token or Data API privacy exceptions, and (3) the exact figures sent are the exact figures the admin is looking at. The cost is that the request is larger and Bubble has to build the book list correctly (see Challenges).

**Request body (API Connector):**
```json
{
  "request_id": "<request_id>",
  "callback_url": "https://<app>.bubbleapps.io/version-test/api/1.1/wf/royalty_summary_callback",
  "author": { "author_id": "...", "name": "...", "email": "...", "city": "..." },
  "books": [ { "book_id": "...", "title": "...", "total_royalty_earned": 0, "royalty_paid": 0, "royalty_pending": 0, "...": "..." } ]
}
```

**Calculation in code, not in the AI.** The Code node validates the payload, parses the book list, and calculates total earned / paid / pending, copies sold, published versus in-production counts, payout status (`Fully paid`, `Below minimum - rolls over`, `Pending payout`), eligibility (pending >= ₹1,000) and the expected payout date. The payout date is the end of the most recent quarter plus 45 days, or the next one if that date has passed (for example 14 Nov 2026 in October 2026). Per-book amounts are pre-formatted (`₹14,240`) so the AI never formats or computes numbers.

**Request ID.** Bubble creates the NotificationLog first with a random `request_id`, sends that same value to n8n, and n8n returns it unchanged. The log is updated from the n8n response.

## Prompt strategy

Gemini receives a system instruction plus the verified-facts JSON, with `temperature: 0.2`. The prompt:
- fixes the persona (Author Relations Manager) and a formal tone, in plain text with a fixed section structure;
- orders the sections: Subject, greeting by full name, Publication Overview, Title-wise Royalty Details, Royalty Statement, Payout Information, closing, signature;
- picks the Payout Information paragraph from the facts: fully paid (acknowledge), below ₹1,000 (rolls over, not overdue), eligible (expected by the supplied date, "not a guarantee");
- forbids inventing or recalculating figures, titles, dates or promises, and forbids giving royalty numbers for books still in production;
- handles edge cases explicitly: authors with no payout ("no payout recorded to date"), books in production, and zero-pending authors.

Outputs were checked by hand against the sample JSON for Sneha Kulkarni (3 books, 1 in production) and Meera Nair (2 books, partially paid): all totals matched.

## Payout badge rules (author portal)

| Badge | Rule |
|---|---|
| Paid (green) | `royalty_pending = 0` |
| Pending (yellow) | pending > 0 and last payout within 90 days |
| Overdue (red) A | pending > 0, a last payout date exists, and it is more than 90 days ago |
| Overdue (red) B | pending > 0, no payout ever, and published more than 90 days ago |

Note: with the sample data every last payout is in 2025, so no book shows yellow today. Books still in production show 0 pending and therefore "Paid". Small pending amounts below ₹1,000 can show Overdue by the assignment rule while the email correctly says "rolls over, not overdue". These are two separate rules.

## Error handling

- **Invalid or incomplete payload:** the Code node throws a clear error (missing author name/email, empty book list, invalid book JSON).
- **AI failure:** the Gemini node continues on error; "Prepare Result + Status" detects a missing response or API error and returns `status: Failed` with an `error_message`.
- **Bubble side:** the log is created as `Processing`, then updated with the returned status, content, error message and timestamp. The result is only written when n8n actually returned a status.

## Known limitations (stated honestly)

- **Bulk generation not verified end to end:** the “Generate All Pending Summaries” button is configured to schedule `generate_author_royalty_summary` on a list of Authors, using a random `request_id` for each scheduled run. However, Bubble’s free-plan workflow limit was reached during final testing, so I could not complete an end-to-end verification of the bulk button. I have contacted Bubble support about the limit. Until it can be tested again, I cannot claim that bulk generation works reliably.
- **Bubble callback:** the callback node and the `royalty_summary_callback` backend workflow were created, but the callback node is currently **disabled**. The intended path is for n8n to return the result synchronously and for Bubble to update the `NotificationLog` from the response. This does not demonstrate that the separate callback path works; it remains unverified/disabled.
- **Individual summary workflow:** the synchronous response path was configured to return the generated content and status to Bubble, where the NotificationLog is updated. Final testing was interrupted by the workflow limit, so the current live integration should be treated as not fully verified under the present plan limit.
- **Email sending:** not implemented (the assignment lists it as a bonus). The sample author emails are fictitious test data, so generated summaries are intended to be stored and shown in the Notification Log rather than sent to those addresses.
- **Security and failure-case checks:** before submission, verify the following in the current app/workflow. I am not claiming these are complete unless they have been tested:
  - Bubble privacy rules: authors can see only their own data, while admins can see all data.
  - Webhook authentication: n8n validates a header/shared secret and the secret is not exposed in the exported JSON.
  - Duplicate-click protection: repeated clicks do not create duplicate summary runs.
  - Failed-status handling: Bubble visibly records/shows a failure when n8n is unreachable or the AI request fails.

Because the free-plan workflow limit interrupted final testing, the bulk feature and the complete integration should be presented as **implemented/configured but not fully verified**, rather than as confirmed working features.
## Challenges and what I learned

- **Bubble was the steepest part.** I learned privacy rules, conditionals, repeating groups, the API Connector, backend workflows and scheduled workflows while building.
- **HTTP 422 "Failed to parse request body".** The request reached n8n but the book list was invalid JSON. Bubble does not show the final serialized body, so I debugged from the error position (line 10, column 14, the very start of the first book object) and the template. The multi-line template had a stray `]` and line-break/indent characters that were not valid JSON. Rebuilding it as one line with no indentation fixed it. I confirmed the fix from the n8n execution input, not from the Bubble UI.
- **Static versus dynamic.** The API Connector "Initialize" succeeding with a static sample proved nothing about the dynamic request. I now check the actual execution input in n8n first.
- **Payout date bug.** My first Code node used the end of the *current* quarter plus 45 days (Feb 2027), but the rule needs the most recent deadline still ahead (Nov 2026). I fixed it and checked it in the n8n output.
- **Bubble date conditions.** Date maths belongs on the date value (`date + days 90 < Current date/time`), not after `Current date/time`.

## Setup and moving to another account

**n8n:** import `n8n/workflow.json`, then recreate separately: the Gemini API key (credential or environment variable `GEMINI_API_KEY`; the exported file must not contain a key), the webhook authentication credential, `BUBBLE_API_TOKEN` (only if you enable the callback). Publish the workflow and use the new `/webhook/bookleaf-royalty-summary` URL.

**Bubble:** duplicate the app (or recreate the types above), then re-check: the API Connector URL and private headers, the hardcoded callback URL host (`<appname>.bubbleapps.io/version-test/...`), the Workflow API setting and API token, and the privacy rules. Data does not move with the app: export Things as CSV and import them again. Passwords are not exportable, so recreate the test logins and their `User.role` / `User.author` links.

## Repository contents

```text
README.md
n8n/workflow.json     (credentials removed)
screenshots/          (optional)
```

## Access and demo

- Live app: [add URL]
- Author test login: [email / password]
- Admin test login: [email / password]
- Bubble editor access: shared with the reviewer email provided
- n8n workflow export: `n8n/workflow.json`
- Screen recording: [add link]

Sensitive keys are not included in this repository.
