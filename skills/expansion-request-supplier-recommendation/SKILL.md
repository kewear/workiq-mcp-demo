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

Read both at the start:

- `environments/D365AITour005/skills/Recommend%20Supplier%20and%20Initiate%20Vendor%20Award`
- `environments/D365AITour005/skills/Finalize%20Vendor%20Award`

Follow their instructions exactly. They define the scoring weights, the draft award fields, the commercial-term rules, and the verification steps.

## Tables: original set only

Use ONLY `aitour_expansionrequest` and `aitour_vendoraward` (plus `aitour_supplier`, invoices, and tasks). NEVER read from or write to any table whose name contains `cab` (for example `aitour_cabexpansionrequest` or `aitour_cabvendoraward`), even if Work IQ discovery, the Caldova Vendor Award app, or a search result lists them or calls them an alternate. If a lookup in the original tables finds nothing, report that. Do not fall back to a cab table.

## Verified schema (use these exact logical names)

Never guess a column. Do not query `aitour_budgetedamount`; it does not exist.

`aitour_expansionrequest`: `aitour_requestnumber`, `aitour_name`, `aitour_description`, `aitour_estimatedbudget` (the budget), `aitour_requeststatus` (100000001 = Ready_for_Recommendations), `aitour_servicecategory` (text), `aitour_location`, `aitour_targetdate`, `aitour_priority`.

`aitour_vendoraward`: `aitour_awardnumber` (format VA-2026-###), `aitour_name`, `aitour_scope`, `aitour_expansionrequestid`, `aitour_selectedsupplierid`, `aitour_awardamount`, `aitour_awarddate`, `aitour_effectivedate`, `aitour_awardstatus` (100000001 = Awarded).

For any other table or column (suppliers, invoices, invoice line items, the Draft status value, tasks), call `get_schema` first and use only what it returns.

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
