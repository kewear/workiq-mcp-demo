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

## Environment

Use the `D365AITour005` environment. Do not ask for environment details.

## Business skills used (read-only)

Read both at the start:

- `environments/D365AITour005/skills/Recommend%20Supplier%20and%20Initiate%20Vendor%20Award`
- `environments/D365AITour005/skills/Finalize%20Vendor%20Award`

Follow their instructions exactly. They define the scoring weights, the draft award fields, the commercial-term rules, and the verification steps.

## Inputs

The event must include `eventType`, `requestNumber`, and `status`. It may include `effectiveDate`.

Proceed only when `eventType` is `ExpansionRequest.ReadyForSupplierRecommendation`, `status` is `Ready for supplier recommendation`, and `requestNumber` is present. Otherwise stop and explain why nothing was done.

## Procedure

### 1. Recommend (read-only, no writes)

Follow the "Recommend suppliers" section of the Recommend Supplier and Initiate Vendor Award skill:

1. Query the Expansion Request by `requestNumber` (`aitour_expansionrequest`, column `aitour_requestnumber`). Confirm it exists and is ready for recommendation.
2. Confirm no Vendor Award already exists for it (`aitour_vendoraward`, column `aitour_expansionrequestid`). If one exists, stop and report it. Do not create a duplicate.
3. Query vetted suppliers matching the request's service category, then their invoices and invoice line items.
4. Rank by the weights in the skill and choose the top supplier.

### 2. Prepare the commercial terms (read-only)

Follow the "Collect commercial terms" section of the Finalize Vendor Award skill:

- Proposed Award Amount: 95 percent of the Expansion Request estimated budget. Label it a suggestion. It must not exceed the estimated budget.
- Effective Date: use `effectiveDate` from the event if present. Otherwise use the Expansion Request target date as the proposed date. Label it proposed.
- Never derive Award Amount from supplier invoices.

### 3. Human approval (required, fresh every run)

Immediately call the attached tool `Human_In_the_loop_approval`. Do not construct Teams cards and do not use Work IQ Teams messages for approval.

A fresh approval is required for every event. Never reuse a prior approval from an earlier run, card, record, or transcript.

Include in the approval request:

- Request number
- Recommended supplier
- Estimated budget
- Proposed award amount (labelled as a suggestion)
- Proposed effective date (labelled as proposed)
- Reason for recommendation
- Options: `Approve`, `Reject`

The approval is the explicit selection of the supplier AND the confirmation of the commercial terms. Wait for the returned decision.

### 4. If the decision is Reject

Stop. Make no writes. Report that the recommendation was rejected.

### 5. If the decision is Approve

Do both steps yourself with the data tools, in this order:

**5a. Create the Draft Vendor Award** (Recommend skill, "Create a draft Vendor Award"). Create one `aitour_vendoraward` record with the expansion request, the approved supplier, Award Status Draft, a name derived from the request name, a short scope derived from the request description, and the next available award number (query existing award numbers and add one to the highest). Leave Award Amount, Award Date, and Effective Date blank at creation. Use `get_schema` on the table first for exact column names and the Draft and Awarded choice values.

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

- Do not write anything before approval returns `Approve`.
- Never modify, create, or delete a business skill.
- Never create a second active Vendor Award for the same request.
- Never claim success unless the writes succeeded and step 5c verified them.

## Final response

Short summary: request number, recommended supplier, approval decision, Vendor Award number and status, whether the three tasks exist, and current status. If something failed, say exactly what.
