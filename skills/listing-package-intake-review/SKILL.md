---
name: listing-package-intake-review
description: Processes an executed NY (GRAR/NYSAR) listing document package for an upcoming listing in Brokermint / BoldTrail Back Office. Uploads the combined signed PDF, splits it into its component documents, files each on the matching "Listing Preparation and Documents" checklist task, updates the transaction fields from what the documents say, submits the documents for review, and approves or rejects each one. Use this whenever the user uploads a signed listing package (Exclusive Right to Sell, agency disclosure, fair housing disclosure, PCDS, listing attachment, delayed negotiation addendum, MLS input sheet, seller disclosure) and says things like "upload and split these", "file these to the checklist", "update the fields from these docs", "submit for review", "approve or deny", or "process this listing package" - even if they only name one of those steps.
---

# Listing Package Intake and Review

Takes one combined, e-signed listing package PDF for a listing that already exists in Back Office (status opportunity / pre-listing) and runs the full sequence:

1. Find the transaction and read its checklist
2. Upload the PDF and split it into component documents filed on the right tasks
3. Update transaction fields from the documents
4. Submit the documents for review
5. Approve or reject each document
6. Report back

The user's instruction authorizes the work. If the user only asks for some steps (for example "upload and split, don't review yet"), do only those steps.

## Tools and setup

- Tools come from the Back Office connectors (a "BoldTrail Backoffice" server and a "brokermint" server). They are usually deferred, so call `tool_search` with a few keywords to load them before use. Do not guess parameter names.
- Most BoldTrail tools need `account_id`. Get it from the brokermint `get_accounts_current` tool (field `id`).
- The brokermint server has a generic `backoffice_request` tool (method, path, body). It is the way to read and update transaction fields (see Step 3).
- You need a shell with outbound network access to POST the PDF. If you cannot reach the network, use the file-picker upload tool instead and tell the user they must pick the file.
- Read the uploaded PDF visually (the page images are in context). Count pages with `pdfinfo` if available.

## Hard rules

- Never delete documents, and never empty or remove the original combined upload. Leave it in Unsorted and say so.
- Never send the review email (`send_reviewed_email`) unless the user explicitly asks. If the user says "no email to <agent>", do not call it.
- Do not approve or reject task exemption requests unless asked. You may note whether the documents support them.
- Do not write to fields you are unsure about. Report them instead.
- Do not touch automation-trigger fields (e.g. any "Automation" dropdown) or sign/lockbox fields unless the user tells you to.
- Tell the user plainly about anything you could not do or could not confirm.

## Step 1: Find the transaction and read its state

1. Search by street name with the brokermint `get_transactions_search` (or BoldTrail `search_transactions`). Note the transaction `id`.
2. `list_all_transaction_tasks` (account_id, bm_transaction_id). Record the task id, checklist id, name, status and whether a document is already attached for each task. A task rejects a document if it already has one.
3. `list_transaction_participants` (full_info=1) and `list_transaction_comments` for context, especially earlier compliance comments about known problems (wrong ZIP, price mismatches, missing sellers).
4. `GET /v3/transactions/{id}` through `backoffice_request` to see current field values.

## Step 2: Identify documents and page ranges

Look at every page of the PDF. A typical combined package arrives in this order, but always verify against the actual pages and do not assume:

| Document | Form / version to expect | Checklist task |
|---|---|---|
| NYS Disclosure Form for Buyer and Seller (agency) | DOS-1736-f (Rev. 09/21), 2 pp | Upload Agency Disclosure |
| NYS Housing and Anti-Discrimination Disclosure | DOS-2156 (06/20), 2 pp | Upload Fair Housing Disclosure |
| Exclusive Right to Sell or Lease Contract | GRAR (c) 5/2025, 4 pp | Upload Exclusive Right to Sell (Pages 1-4) |
| Delayed Showing / Negotiation Addendum | NYSAMLS Rev. 6/2023, 1 p | Upload Delayed Negotiations Form |
| Brokerage property info sheet (utilities, improvements) | firm form, 1 p | Upload Supplemental Property Information |
| Seller Disclosure for Property Showings and Offer Submission | firm form, 1 p | Upload Seller Disclosure for Showings and Offer Submission |
| Property Condition Disclosure Statement | DOS-1614-f (Rev. 02/25), 7 pp | Upload PCD |
| Attachment to Exclusive Right to Sell | GRAR (c) 5/2025, 2 pp | Upload Contract Attachment |
| MLS Input Sheet (Single-Family RES) | NYS Alliance of MLSs, Rev. 12/30/2025, 5 pp | Upload MLS Input Sheet |

Write down the page range for each document and check the ranges add up to the PDF's page count. Not every package contains every document: only split what is present. If a document is present but there is no matching task, or a task already has a document, leave the split piece unsorted and tell the user.

The package may also include documents that belong to other tasks (lead paint form, rented property form, HOA docs, etc.). Map them by name to the checklist the same way. If a form version shown on the page differs from the list above, say so in your report; do not treat versions as interchangeable.

## Step 3: Upload and split

1. Call `upload_document_to_transaction_by_sending_file_bytes` with `account_id` and `bm_transaction_id` and no `task_id`. It returns a short-lived `upload_url`.
2. POST the file immediately (the URL expires):
   `curl -s -F "file=@/path/to/file.pdf;type=application/pdf" "<upload_url>"`
   The response contains the new document `id` and `pages`. Confirm the page count matches the PDF.
3. For each component, call `split_transaction_document` with `transaction_id`, `document_id` (the combined upload), `document_name`, `start_page`, `end_page`, and `task_id` of the matching task. The split goes straight onto the task. Name pieces like `NYS Agency Disclosure - <street address>`. The returned `pages` should equal end minus start plus one.
4. Do splits one at a time and keep a list of each new document id; you need them for review.

If the original package has `has_package: true` (a Back Office e-sign package), call `get_document_page_ranges` first to get the boundaries.

## Step 4: Update transaction fields

1. `GET /v3/transactions/schema` to get exact field labels, types and dropdown options.
2. Compare what the documents say against the current values from Step 1. Update with `PUT /v3/transactions/{id}` using field labels as keys (the same keys the GET returns). Send only the fields that change.
3. Date fields (`listing_date`, `expiration_date`) are 13-digit millisecond timestamps. Match the convention of the existing `listing_date` (midnight UTC for that calendar date) and verify with Python that the date round-trips. Dates must match the documents exactly, with no rounding or timezone drift.
4. Text fields are free text. Use plain, consistent wording (for example `10/13/2026 at 2:00 PM`).

Fields this process normally fills when the documents state them:

| Source | Field |
|---|---|
| ERTS and every other form (ZIP shown on signed forms) | zip |
| ERTS paragraph 3/4 | listing_date, expiration_date (do not change an existing listing_date that already matches) |
| ERTS owners / signatures | Seller, Seller 2 (and verify Property Owner, Property Owner 2) |
| ERTS price | Recommended List Price / price (should match $ in paragraph 2) |
| ERTS paragraph 9 / 10A | Seller Agent Commission, Buyer Agent Commission (verify, correct only if wrong) |
| Delayed Negotiation Addendum | Delayed Offers Due; Showings Start (only if the addendum delays negotiations only, then showings start at the list date) |
| MLS input sheet | Half baths and Full baths (when clearly legible), Year Built, School District, Included Items, Exclusions (from private remarks) |
| Attachment (paragraph T) | HOA: (No when not governed by an HOA) |

Leave these alone unless the user says otherwise: automation dropdowns, Yard sign, Lockbox, Lockbox Type, Occupancy Status, Bedrooms when the handwriting is ambiguous, Property Description, Showing Instructions, anything the documents do not state. List what you left alone and why.

After the PUT, read the response and confirm each changed field.

## Step 5: Submit for review

Call `submit_document_for_review` (account_id, document_id) for every split document that has `document_approval_required` on its task. Splitting alone does not queue a document. Check with `list_pending_review_documents` (type `["transaction"]`, bm_transaction_ids) if needed.

## Step 6: Review each document

For each document check: correct form and version; owner/party names match across all documents; property address and ZIP match; price and dates match the ERTS; all required signatures present for every owner and the listing agent; initials present where the form calls for them; dates beside signatures; no material blank fields where the form expects completion; nothing conflicting with other documents. Be strict but practical: do not invent deficiencies, and do not fail a document for unused optional sections.

Decisions:

- **Approve** when the document is complete, signed, and consistent. Put form name and version, who signed and when, and any minor non-critical notes in the comment.
- **Reject** for material problems: missing signature or initials on a required line, wrong or inconsistent address or ZIP, wrong parties, price or term conflicts, material blanks on a form that is supposed to be complete. The comment must say exactly what is wrong and how to fix it.
- **MLS Input Sheet is information-only.** The listing coordinator completes the MLS entry, so blank fields are not a deficiency. Approve it when it is signed and the names and address are consistent, every time. Do not reject it for blanks.
- A wrong ZIP (or other address typo) on a signed form is a rejection reason: the form must be regenerated from the corrected record and re-signed.

How to record decisions with `approve_review_document` / `reject_review_document`:

- Approvals: use `finalize: true` so the task goes straight to complete (approved) with no email.
- Rejections when the user said not to email the agent: use `finalize: false`. This leaves "review rejected (not sent)" and nothing is sent. Tell the user these are drafts and that sending the review (or asking you to finalize) is their call.
- Do not call `send_reviewed_email` unless asked.

Gotcha when changing a decision: a task in "review rejected (not sent)" cannot be approved with `finalize: true` directly, and resubmitting fails. Approve once with `finalize: false` (task becomes "review approved (not sent)"), then approve again with `finalize: true`. The earlier rejection comment stays on the task and cannot be deleted by the API; tell the user so they can remove it in Back Office.

Verify at the end with `get_transaction_checklist_task` (status should read "complete (approved)" for approvals).

## Step 7: Report

Give the user a short summary:

1. What was uploaded (document id, page count) and the table of documents filed to tasks.
2. Fields updated (old to new where useful) and fields deliberately left alone.
3. Review decisions: approved vs rejected, with the reason for each rejection.
4. What was not done (no email sent, original package left in Unsorted, exemptions not decided) and anything that needs a human decision.

Also flag cross-document findings that matter to the listing even if you approved: conflicting prices between the ERTS and an agent note, disclosures that must be echoed in MLS remarks (for example flood claims or electrical defects in the PCDS), delayed-negotiation dates that must appear in public and private remarks, pending exemption requests the documents do or do not support (for example a lead form is not needed for a home built after 1978).
