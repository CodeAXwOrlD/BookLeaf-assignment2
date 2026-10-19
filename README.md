# BookLeaf — Author Royalty Dashboard & Automated Notification System

A full-stack no-code automation project built with **Bubble.io, n8n, and Google Gemini** to help authors track book royalties and enable administrators to generate royalty summaries through an automated workflow.

## Overview

BookLeaf provides an author-facing dashboard and an administrative interface for managing author and book information, monitoring royalty balances, and initiating royalty summary generation.

The project combines Bubble.io for the application interface and data management with n8n for workflow orchestration and AI-assisted summary generation.

## Key Features

### Author Dashboard
- Author-specific book listings.
- Royalty earned, paid, and pending balance visibility.
- Book publication status and sales information.
- Last royalty payout information.
- Royalty summary generation workflow.

### Admin Dashboard
- Centralized author and book overview.
- Author and book statistics.
- Pending royalty overview.
- Author-specific detail pages.
- Actions to initiate royalty summary generation.

### Automated Royalty Summary Workflow
- Receives author and book information through an n8n webhook.
- Processes royalty-related data.
- Uses Google Gemini to generate an AI-assisted royalty summary.
- Prepares a structured workflow response.
- Supports a Bubble callback integration for updating notification records.

### Notification Tracking
The application uses a `NotificationLog` data type to track royalty summary requests, generated content, processing status, timestamps, and error information.

## Technology Stack

| Technology | Purpose |
|---|---|
| Bubble.io | Application UI, database, authentication, and workflows |
| n8n | Workflow automation and API orchestration |
| Google Gemini | AI-assisted royalty summary generation |
| REST API / Webhooks | Communication between Bubble and n8n |
| JSON | Structured request and response payloads |

## Architecture

```text
Author / Admin
      |
      v
  Bubble.io
  Dashboard + Database
      |
      | Royalty summary request
      v
  n8n Webhook
      |
      v
  Royalty Facts Processing
      |
      v
  Google Gemini
      |
      v
  Prepare Summary + Status
      |
      +------> Bubble Callback
      |        NotificationLog Update
      |
      v
  Webhook Response
      |
      v
  Bubble Application
```

## Data Model

The application uses the following core data types:

- **Author:** Author ID, name, email, phone, city, and joining date.
- **Book:** Book ID, title, ISBN, genre, publication details, status, MRP, sales, earned royalty, paid royalty, pending royalty, and last payout date.
- **NotificationLog:** Author reference, request ID, status, creation and completion timestamps, generated content, and error message.
- **User:** User role and associated author record for role-based application flows.

## Royalty and Payout Rules

The system is designed around the assignment's royalty and payout requirements:

- Minimum payout threshold: ₹1,000.
- Balances below the minimum threshold roll over to a future payout.
- Royalty summaries should reflect earned, paid, and pending amounts.
- Payout status indicators should communicate production, pending, and overdue conditions.
- Quarterly payout deadlines follow the assignment's specified schedule.

Royalty figures and payout eligibility should be calculated from the underlying book and author data rather than invented by the AI-generated summary.

## Workflow Setup

### 1. Configure Bubble.io

1. Create the required data types and fields.
2. Add author and book records.
3. Configure author and admin application flows.
4. Create the royalty summary workflow.
5. Configure the API Connector request to send the author details, book list, request ID, and callback URL.

### 2. Configure n8n

1. Import the exported workflow JSON into n8n.
2. Configure the webhook endpoint.
3. Configure the Google Gemini API credential.
4. Configure the Bubble callback authentication.
5. Activate the workflow and verify the correct webhook URL.
6. Test the workflow with representative author and book records.

### 3. Configure the Bubble Callback

Create the backend API workflow `royalty_summary_callback` with the appropriate request parameters.

The callback should locate the corresponding notification record using its request ID and update its status, generated content, error details, and completion timestamp.

### 4. Validate the Integration

Test the complete Bubble-to-n8n-to-Bubble flow before relying on the workflow in a live environment.

Verify that successful requests update the correct notification record and that failed requests can be diagnosed using the returned status and error information.

## Project Structure

```text
BookLeaf-Royalty-Dashboard/
├── README.md
├── n8n/
│   └── workflow.json
└── screenshots/
    └── dashboard.png
```

The `screenshots/` directory is optional. Add screenshots of the actual application after final UI verification.

## Testing Checklist

- [ ] Author login and author-specific book visibility.
- [ ] Admin dashboard and author details.
- [ ] Royalty totals and payout status calculations.
- [ ] Dynamic Bubble-to-n8n webhook request.
- [ ] AI-assisted summary generation.
- [ ] Bubble callback and NotificationLog updates.
- [ ] Failed-request handling.
- [ ] Multiple books for one author.
- [ ] Final workflow export and documentation.
- [ ] Demo recording and application access verification.

## Configuration and Security

- Store API credentials in the appropriate credential or secret-management mechanism.
- Do not commit API keys, bearer tokens, passwords, or other secrets to GitHub.
- Use test credentials and representative data for development.
- Verify Bubble privacy rules before sharing the application with reviewers.
- Use the production webhook URL only after the workflow has been activated and tested.

## Demo and Deliverables

**Application URL:** [Add Bubble application URL]

**n8n Workflow Export:** `n8n/workflow.json`

**Demo Recording:** [Add recording link]

**Test Access:** [Provide reviewer access details securely]

## Current Status

The core Bubble dashboard, author/book data setup, author detail views, and initial n8n workflow have been configured. The dynamic webhook request, callback updates, and complete end-to-end behavior must be verified before the integration can be considered submission-ready.

## Future Improvements

- Email delivery of generated royalty summaries.
- More detailed payout history.
- Improved retry and duplicate-request handling.
- Additional audit and reporting capabilities.

---

**Built as part of the BookLeaf Full-Stack Developer Assignment — Author Royalty Dashboard & Automated Notification System.**
