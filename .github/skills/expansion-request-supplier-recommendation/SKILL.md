---
name: expansion-request-supplier-recommendation
description: Use when eventType is ExpansionRequest.ReadyForSupplierRecommendation. Reads the two Business Skills as instructions, ranks suppliers, gets human approval through Human_In_the_loop_approval, then creates and finalizes the Vendor Award by following the Finalize Vendor Award skill.
---

# Expansion Request Supplier Recommendation

## Core principle

Business skills are INSTRUCTIONS, not executable actions. They describe how to solve a problem. You carry out those instructions yourself using the data tools (query, create record, update record).

- Read a business skill with `fetch` on its skill path. That returns its instructions.
- NEVER call update, create, or delete on any `/skills/...` path. That overwrites the skill definition and destroys it.
- The `fetch/update/delete` operations listed for a skill describe how to manage the skill document. They are not a way to run it.
- Never look for an "execution surface" for a business skill. There isn't one, and none is needed.
- Do not invent `/customapis/...` paths.

## Do not stop early

Keep working until the procedure reaches one of these end states: (a) a Reject at step 4, (b) step 5c verification complete, (c) a tool error you report exactly, (d) an eligibility or duplicate-award stop, or (e) `Pending human approval` when no decision is returned. Do not end your turn after discovery, after reading a skill, or after ranking suppliers. Do not close with an offer such as "want me to proceed" or "I can monitor". The next required action after ranking is always the `Human_In_the_loop_approval` call.

## Environment

Use the `D365AITour005` environment. Do not ask for environment details.

## Business skills used (read-only)

Read both at the start. Their names are:

- `Recommend Supplier and Initiate Vendor Award`
- `Finalize Vendor Award`

Do not hand-build paths. Call `search_paths` with the skill name, then `fetch` the path it returns, exactly as returned.

If a live read is denied ("Access denied"), returns "not found", or fails for any reason, DO NOT STOP. Read the bundled fallback copy in this skill's folder instead and continue:

- `references/recommend-supplier-and-initiate-vendor-award.md`
- `references/finalize-vendor-award.md`

The bundled copies have the same content as the Work IQ skills. Say in your final response which source you used. A denied or failed skill read is never a reason to end the run.

Follow their instructions exactly. They define the scoring weights, the draft award fields, the commercial-term rules, and the verification steps.
## Tables: original set only

Use ONLY `aitour_expansionrequest` and `aitour_vendoraward` (plus `aitour_supplier`, invoices, and tasks). NEVER read from or write to any table whose name contains `cab` (for example `aitour_cabexpansionrequest` or `aitour_cabvendoraward`), even if Work IQ discovery, the Caldova Vendor Award app, or a search result lists them or calls them an alternate. If a lookup in the original tables finds nothing, report that. Do not fall back to a cab table.

## Verified schema (use these exact logical names)

Never guess a column. Do not query `aitour_budgetedamount`; it does not exist.

`aitour_expansionrequest`: `aitour_requestnumber`, `aitour_name`, `aitour_description`, `aitour_estimatedbudget` (the budget), `aitour_requeststatus` (100000001 = Ready_for_Recommendations), `aitour_servicecategory` (text), `aitour_location`, `aitour_targetdate`, `aitour_priority`.

`aitour_vendoraward`: `aitour_awardnumber` (format VA-2026-###), `aitour_name`, `aitour_scope`, `aitour_expansionrequestid`, `aitour_selectedsupplierid`, `aitour_awardamount`, `aitour_awarddate`, `aitour_effectivedate`, `aitour_awardstatus` (100000001 = Awarded).

`aitour_supplier`: `aitour_supplierid`, `aitour_name`, `aitour_suppliercode`, `aitour_servicecategory` (a single category such as "Drug product manufacturing"), `aitour_vettedstatus` (BIT, 1 = vetted), `aitour_addressline1`, `aitour_addressline2` (city and region), `aitour_addressline3` (country).

`aitour_invoice`: `aitour_invoiceid`, `aitour_invoicecode`, `aitour_supplierid` (lookup to supplier), `aitour_totalamount`, `aitour_invoicedate`, `aitour_supportreferences`.

`aitour_invoicelineitem`: `aitour_invoicelineitemid`, `aitour_invoiceid` (lookup to invoice), `aitour_description`, `aitour_amount`, `aitour_expecteddecisionstate` (text: matched, variance, disputed, escalated), `aitour_issuecategory`, `aitour_linetype`.

`aitour_vendoraward` choice values for `aitour_awardstatus`: Draft = 100000000, Awarded = 100000001, Processing_Complete = 100000002, Cancelled = 100000003. `aitour_awarddate` and `aitour_effectivedate` are date-only (use `YYYY-MM-DD`).

The request's `aitour_servicecategory` can list several categories separated by semicolons. The first listed is the lead category. Match suppliers on the lead category.

Working queries (verified):

- Vetted suppliers: `SELECT aitour_supplierid, aitour_name, aitour_suppliercode, aitour_servicecategory, aitour_addressline1, aitour_addressline2, aitour_addressline3 FROM aitour_supplier WHERE aitour_vettedstatus = 1`
- Commercial history for chosen suppliers: `SELECT i.aitour_supplierid, i.aitour_invoicecode, i.aitour_totalamount, l.aitour_expecteddecisionstate, l.aitour_description, l.aitour_amount FROM aitour_invoice i JOIN aitour_invoicelineitem l ON i.aitour_invoiceid = l.aitour_invoiceid WHERE i.aitour_supplierid IN ('<id1>','<id2>')`
- Existing awards for the request: `SELECT aitour_awardnumber, aitour_awardstatus FROM aitour_vendoraward WHERE aitour_expansionrequestid = '<expansionRequestId>'`
- Highest award number: `SELECT TOP 1 aitour_awardnumber FROM aitour_vendoraward ORDER BY aitour_awardnumber DESC`

Do not call `get_schema` or sample-query these tables to discover column names. They are listed above. Use `get_schema` only for a table not listed here (for example Task).

## How to read and write data (verified)

READ with the environment query action. This is the only read method to use for the tables in this skill:

- Tool: `do_action`
- `actionUrl`: the query operation path returned by `search_paths` (it ends in `/query` and lists the `action` operation). Use it exactly as returned. It is usually `environments/D365AITour005/query`.
- `jsonBody`: `{"querytext": "SELECT ... FROM aitour_expansionrequest WHERE aitour_requestnumber = '<requestNumber>'"}`

Use Dataverse logical table and column names, one SELECT, and an explicit column list or `SELECT *`. No subqueries, DISTINCT, HAVING, CASE, or CAST. JOINs on equality are allowed.

Example for step 1: `SELECT aitour_expansionrequestid, aitour_name, aitour_estimatedbudget, aitour_requeststatus, aitour_servicecategory, aitour_targetdate FROM aitour_expansionrequest WHERE aitour_requestnumber = '<requestNumber>'`

Do NOT use app-scoped table paths (anything under `environments/D365AITour005/apps/...`) to read these records. The Caldova Vendor Award app is wired to the cab tables, so that route leads to the wrong data. If a fetch returns no record data, use the query action above instead of switching to an app view.

WRITE with `create_entity` and `update_entity` on the environment table path. Records are addressed as `environments/D365AITour005/tables/<logical table name>/records/<record id>` for update. Call `get_schema` on the table first for exact column names and choice values.
## Inputs

The event must include `eventType`, `requestNumber`, and `status`. It may include `effectiveDate`.

Proceed only when `eventType` is `ExpansionRequest.ReadyForSupplierRecommendation`, `status` is `Ready for supplier recommendation`, and `requestNumber` is present. Otherwise stop and explain why nothing was done.

## Procedure

### 1. Recommend (read-only, no writes)

Follow the "Recommend suppliers" section of the Recommend Supplier and Initiate Vendor Award skill:

1. Query the Expansion Request by `requestNumber`. Confirm it exists and is ready for recommendation.
2. Confirm no Vendor Award already exists for it (`aitour_vendoraward.aitour_expansionrequestid`). If one exists, stop and report it. Do not create a duplicate.
3. Query vetted suppliers matching the request's service category, then their invoices and invoice line items.
4. Rank by the weights in the skill and choose the top supplier.

### 2. Prepare the commercial terms (read-only)

Follow the "Collect commercial terms" section of the Finalize Vendor Award skill:

- Proposed Award Amount: 95 percent of `aitour_estimatedbudget`. It is a suggestion and must not exceed the budget.
- Effective Date: use `effectiveDate` from the event if present. Otherwise use `aitour_targetdate` as the proposed date.
- Never derive Award Amount from supplier invoices.

### 3. Human approval (required, fresh every run)

Immediately call the attached tool `Human_In_the_loop_approval`. Do not construct Teams cards. Do not post a plain Teams message or a visual-only Adaptive Card. Do not use Work IQ Teams messages for approval. Teams delivery is owned by the tool.

A fresh approval is required for every event. Never reuse a prior approval from an earlier run, card, record, or transcript. Do not ask whether to call the tool. Calling it is mandatory.

Pass these fields:

- Approval title: `Supplier Recommendation Approval`
- Request number
- Recommended vendor (the supplier you ranked first)
- Budgeted amount (`aitour_estimatedbudget`)
- Award amount (the proposed amount from step 2)
- Reason: why the supplier was recommended. End the reason with: `Award amount is a suggested 95% of budget. Proposed effective date: <date>.` The tool has no effective-date field, so it travels in the reason.
- Vendor Award Id: `Pending` (the draft award is created only after approval)
- Options: `Approve`, `Reject`
- If the tool has destination fields: Team `Expansion Requests`, Channel `Requests`

Approval field formatting (the tool builds a Teams Adaptive Card from these strings; unsafe text makes Teams show "Card - access it on go.skype.com/cards.unsupported" and nobody can approve):

- Every field is plain text on a single line. No double quotes, backslashes, line breaks, markdown, bullet characters, emoji, or angle brackets. Do not use quotation marks around names.
- Amounts: digits with a dollar sign and commas only, for example `$8,930,000`. No words such as "approximately".
- Vendor: the supplier name only.
- Reason: ONE sentence, at most 180 characters, ending with a period. Put the proposed effective date in it in plain form, for example `Proposed effective date 2027-05-31.` Do not add the 95 percent explanation to the card.
- Vendor Award Id: the single word `Pending`.
- Mention risks such as a disputed line in your own final response, not in the card.
The approval is the explicit selection of the supplier and the confirmation of the commercial terms. Wait for the returned decision. It returns `Approve` or `Reject`.

If the tool is unavailable or fails, stop and report the failure. Do not post a plain Teams message and do not write anything. If no decision is returned in this execution, end with status `Pending human approval` and write nothing.

### 4. If the decision is Reject

Stop. Make no writes. Do not finalize. End with status `Rejected by human approver`.

### 5. If the decision is Approve

Do these steps yourself with the data tools, in order.

**5a. Create the Draft Vendor Award** (Recommend skill, "Create a draft Vendor Award"). Create one `aitour_vendoraward` record with the expansion request, the approved supplier, Award Status Draft, a name derived from the request name, a short scope derived from the request description, and the next available award number (query existing award numbers and add one to the highest). Leave Award Amount, Award Date, and Effective Date blank at creation. Use `get_schema` on the table first for the Draft choice value.

**5b. Finalize** (Finalize skill, "Finalize the award"). Update ONLY that Vendor Award record:

- Award Amount: the approved amount
- Effective Date: the approved date
- Award Date: today
- Award Status: Awarded

Do NOT update the Expansion Request. Do NOT create Tasks. A server-side plugin does that in the same transaction: it marks the request Awarded, copies the supplier, and creates the Legal, Procurement, and Finance tasks.

**5c. Verify** (Finalize skill, "Verify the result"):

1. The Vendor Award is Awarded with the correct amount and dates.
2. The Expansion Request is Awarded and has the selected supplier.
3. Exactly one open Task exists for each of Legal, Procurement, and Finance.

If the update fails or the plugin rejects it, report the exact error. Do not retry automatically after an ambiguous response.

## Safety

- Do not write anything before approval returns `Approve` for the current run.
- Never modify, create, or delete a business skill.
- Never create a second active Vendor Award for the same request.
- Do not offer to bypass approval.
- Do not claim completion unless the writes succeeded and step 5c verified them.
- If a tool call fails, report the failure clearly and stop.

## Final response

Short summary:

- Request number
- Recommended supplier
- Human decision: `Approve`, `Reject`, or `Pending`
- Vendor Award number and status, or pending
- Whether the three tasks exist
- Teams approval destination: `Expansion Requests` / `Requests`
- Current status: `Awarded after human approval`, `Rejected by human approver`, or `Pending human approval`

If something failed, say exactly what.
