# BookLeaf: Author Royalty Dashboard & Automated Notification System

Assignment 2 (Full-Stack Developer). Built with **Bubble.io** (app, database, auth), **n8n** (automation backend) and **Google Gemini** (AI-written royalty summary).

Authors log in to see their own books and royalty figures. Admins see every author and can trigger an AI-generated royalty summary, which is calculated in n8n, written by Gemini, returned to Bubble and stored in a Notification Log.

**Submission status note:** The Bubble UI and individual-summary workflow are configured, but final end-to-end AI generation must be re-tested with a valid Gemini API key. The bulk trigger and separate callback path are not verified. This README distinguishes configured functionality from verified behavior.

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

## Setup and configuration

> **Gemini API configuration:** The exported workflow uses a placeholder for the API key. Before testing AI generation, add a valid Gemini API key to the `x-goog-api-key` header in the n8n HTTP Request node. Do not include the key in the exported workflow.

### Prerequisites

- A Bubble.io app with the data types and pages described above.
- An n8n instance that can receive public HTTPS webhooks.
- A Google Gemini API key with access to the configured Gemini model.
- Access to the Bubble editor and the n8n workflow.
- Test Author and Admin accounts linked to the appropriate `User` records.

### 1. Configure n8n

1. Open n8n and import the provided workflow JSON.
2. Open **AI - Draft Royalty Email (Gemini Primary)**.
3. Confirm the HTTP Request node uses the intended Gemini API endpoint and `Content-Type: application/json`.
4. Add your Gemini API key to the `x-goog-api-key` header using n8n credentials or a protected environment variable where supported. Do not leave the value blank when testing.
5. Run a test execution and inspect the Gemini node output. A green overall execution alone is not enough: confirm the node returned generated text and not an API error.
6. Check the Code and Prepare Result + Status nodes with a representative author payload. Verify totals against the supplied book records.
7. Activate/publish the workflow and copy its **production** webhook URL. Do not use the temporary `/webhook-test/` URL for the deployed Bubble app.
8. If enabling the separate Bubble callback path, configure its Bubble endpoint and credentials separately. The callback node is disabled in this submission and should not be assumed to work.

### 2. Configure Bubble (when using this supplied Bubble app)

The Bubble app already contains the UI, data types, sample records, and workflows. **Do not recreate the database or pages** when you have been given editor access to this app.

1. Open the supplied app from the Bubble dashboard and open the editor.
2. Go to **Plugins → API Connector**.
3. Open the existing **n8n Royalty - Generate Royalty Summary** call.
4. In the call's URL field, paste the **active Production URL** shown by the n8n Webhook node for the workflow you intend to use. The URL format is `https://<your-n8n-host>/webhook/bookleaf-royalty-summary`. Do not use `/webhook-test/` for the deployed app.
5. Keep the existing HTTP method, JSON body, and parameter mappings unchanged unless the imported workflow expects a different payload. Initialize/test the call only after the n8n workflow is active and its Gemini API key is configured.
6. Save the API Connector settings, then test the **individual Generate Royalty Summary** action from an author detail page.
7. Confirm the result page shows the returned summary and the matching `NotificationLog` status/content. If the request fails, inspect the latest n8n execution and the API Connector response before changing the body.
8. If the n8n workflow is running on the original supplied n8n instance, keep its current Production URL. If the workflow was imported into another n8n instance, use that new instance's Production URL instead.

#### URL-change rule: what to update and when

- **If you only add the reviewer as a Bubble collaborator:** do not change any URL. Inviting a collaborator normally does not change the Bubble app URL or the n8n webhook URL.
- **If the Bubble app URL stays the same:** do not change Bubble URLs in n8n. Keep the existing setup.
- **If the Bubble app's hostname changes or the app is replaced with a different Bubble app:** the n8n webhook URL does not need to change just because Bubble changed. The Bubble API Connector should continue pointing to the active n8n Production URL. The exported workflow has a hard-coded Bubble callback fallback in the `Calculate Royalty Facts` Code node and a URL in the disabled `Bubble Callback - Save Notification Log` node. The callback node is disabled in this submission, so do not change these just for the current synchronous response flow. If the callback is enabled later, update both Bubble callback URL references to the new app's correct endpoint and configure the required Bubble API workflow/authentication.
- **If the n8n host or webhook path changes:** update the URL in Bubble under **Plugins → API Connector → n8n Royalty - Generate Royalty Summary** to the active Production URL shown by that n8n Webhook node. Do not change it if the n8n URL is unchanged.
- **If the reviewer imports the workflow into their own n8n instance:** use that instance's Production URL in the supplied Bubble app's API Connector. The URL will usually differ because the host is different, even if the webhook path stays `bookleaf-royalty-summary`.

**If you are configuring a different/new Bubble app instead:** install the Bubble **API Connector** plugin via **Plugins → Add plugins → API Connector**, then recreate the call with the same method, request body, and dynamic fields from the supplied app. Import/recreate the required data types, pages, workflows, sample records, and user-role relationships separately; changing the webhook URL alone does not recreate the app.

### 3. Test the integration

- **Input validation:** send a representative author with their full book list; inspect the actual input received by n8n.
- **Royalty calculation:** compare earned, paid, pending, copy counts, production/published counts, eligibility, and due date with the stored records.
- **Gemini generation:** confirm the Gemini node returns generated text. Test an invalid/missing key only as a deliberate failure test.
- **Bubble result:** confirm the summary page displays the returned content and that the corresponding `NotificationLog` is updated.
- **Failure path:** confirm an API error produces a visible `Failed` status and a useful error message rather than an empty summary marked successful.
- **Bulk trigger:** this was not verified end to end because the Bubble free-plan workflow limit was reached. Do not present it as tested.
- **Security:** confirm credentials are not included in the exported workflow or repository.

### 4. If the reviewer uses a separate Bubble app

This section applies only if the reviewer is not using the supplied Bubble app and instead creates/uses a separate app.

- Recreate the required data types, pages, workflows, and `User.role` / `User.author` relationships in that app, or use a supported app-copy/transfer process.
- In **Plugins → API Connector**, point the existing summary call to the Production webhook URL of the n8n instance they will run.
- Update any hard-coded Bubble app hostname in callback URLs only if the callback path is enabled. The callback node is disabled in this submission, so the synchronous response path is the intended integration.
- Import/create sample data and test users separately. Do not assume that copying the workflow JSON transfers Bubble database records or user accounts.

## Repository contents

```text
README.md
n8n/workflow.json     (credentials removed)
screenshots/          (optional)
```

## Access and demo

- Live Bubble preview: https://coderakhil786.bubbleapps.io/version-test?debug_mode=true
- n8n workflow: BookLeaf Royalty Summary - Gemini Primary.json
- Screen recording: [[add link]](https://drive.google.com/file/d/1-fPAX_SAkbzk6WQqkMRR5az3GpkHovp0/view?usp=sharing)
- Test accounts: Admin and author test accounts are available for reviewing the application. Test credentials should be shared directly with the reviewer, not stored in this public repository.
Sensitive keys are not included in this repository.

