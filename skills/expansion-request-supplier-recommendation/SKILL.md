---
name: expansion-request-supplier-recommendation
description: Use when an event has eventType ExpansionRequest.ReadyForSupplierRecommendation or ExpansionRequest.SupplierRecommendationApprovalReceived. Processes the Expansion Request supplier recommendation, creates the required human approval gate, and completes the request only after an explicit approval event.
---

# Expansion Request Supplier Recommendation

## Required skill-found acknowledgement

When this skill is selected, the first assistant message must explicitly confirm that the skill was found and selected:

```text
Found skill: expansion-request-supplier-recommendation. I will process Expansion Request <requestNumber> and create the required human approval gate.
```

Do not start by saying that you will search for Work IQ paths. Skill selection has already happened.

## Purpose

Process Expansion Request business events that require supplier recommendation and human approval. This skill has two supported phases:

1. `ExpansionRequest.ReadyForSupplierRecommendation`: recommend a supplier, initiate or propose the Vendor Award process in Dataverse, and create an actionable Teams approval request through Work IQ.
2. `ExpansionRequest.SupplierRecommendationApprovalReceived`: complete or reject the Expansion Request only after the human approval decision is received.

Human approval is required before the Expansion Request or Vendor Award can be completed.

## Supported events

- `ExpansionRequest.ReadyForSupplierRecommendation`
- `ExpansionRequest.SupplierRecommendationApprovalReceived`

## Inputs

The supplier recommendation event payload must include:

- `eventType`
- `requestNumber`
- `status`

Example supplier recommendation event:

```json
{
  "eventType": "ExpansionRequest.ReadyForSupplierRecommendation",
  "requestNumber": "EXP-2026-004",
  "status": "Ready for supplier recommendation"
}
```

The approval decision event payload must include:

- `eventType`
- `requestNumber`
- `decision`
- `approver`

Example approval decision event:

```json
{
  "eventType": "ExpansionRequest.SupplierRecommendationApprovalReceived",
  "requestNumber": "EXP-2026-004",
  "decision": "Approve",
  "approver": "Kent Weare"
}
```

## Environment binding

Use the configured `D365AITour` Dataverse environment. Do not ask the user or event payload for the Dataverse environment. The event contains business context only. The skill owns the environment and tool binding.

## Phase 1: Supplier recommendation and approval request

Use this phase when `eventType` is `ExpansionRequest.ReadyForSupplierRecommendation`.

Eligibility checks:

- `status` equals `Ready for supplier recommendation`
- `requestNumber` is present

Business process:

1. Find the Expansion Request by `requestNumber`.
2. Confirm the request exists.
3. Confirm the request is ready for supplier recommendation.
4. Run the Dataverse skill `cr2d6_skill_recommend_supplier_and_initiate_vendor_award`.
5. Treat responses such as `Skill updated successfully`, `acknowledged`, or `workflow initiated` as intermediate acknowledgements only. They are not completion.
6. After the Dataverse skill acknowledgement, fetch the Expansion Request, supplier recommendation, and Vendor Award details again if tools are available.
7. Capture the recommended supplier, award amount if available, recommendation rationale, and proposed Vendor Award identifier if available.
8. Create a Teams approval request using Work IQ. This must be actionable, not just informational.
9. Stop with status `Pending human approval`.

## Human approval gate

The Teams approval request is mandatory and must be created after the supplier recommendation step succeeds or is acknowledged.

Do not wait for finalized Vendor Award records before creating approval. If the Vendor Award identifier, recommended supplier, award amount, or rationale are not available yet, include `Pending` or `Not available yet` for those fields and still create the approval request.

The process is not complete after `Skill updated successfully`. The process is only waiting for a human approval decision.

The Teams approval must allow the approver to choose:

- `Approve`
- `Reject`

If Work IQ supports actionable cards, approvals, buttons, or adaptive-card actions, use that mechanism. If only a Teams channel message is available, post a clear approval request message with the `Approve` and `Reject` options and return `Pending human approval`.

## Teams approval destination

- Team: `Expansion Requests`
- Channel: `Requests`

The Teams approval request must include:

- Expansion Request number
- Recommended supplier, or `Pending` if not available yet
- Award amount, or `Not available yet`
- Reason for recommendation, or `Pending`
- Vendor Award identifier, if created, or `Pending`
- Approval options: `Approve`, `Reject`
- Status: `Pending human approval`

Before posting, resolve the Team ID and Channel ID. If exactly one matching team and channel is found, create the approval request. If the destination is ambiguous or not found, return an error and do not post.

## Phase 2: Approval decision completion

Use this phase when `eventType` is `ExpansionRequest.SupplierRecommendationApprovalReceived`.

Eligibility checks:

- `requestNumber` is present
- `decision` is `Approve` or `Reject`
- `approver` is present

Business process for `Approve`:

1. Find the Expansion Request by `requestNumber`.
2. Confirm the request exists.
3. Confirm the request is waiting for supplier recommendation approval.
4. Finalize or complete the Vendor Award only if the approval decision is `Approve`.
5. Mark the Expansion Request or related approval state as approved or completed using the available Dataverse action or record update.
6. Return status `Completed after human approval`.

Business process for `Reject`:

1. Find the Expansion Request by `requestNumber`.
2. Confirm the request exists.
3. Mark the supplier recommendation approval as rejected using the available Dataverse action or record update.
4. Do not finalize the Vendor Award.
5. Return status `Rejected by human approver`.

## Safety rules

- Do not finalize the Vendor Award during the `ReadyForSupplierRecommendation` phase.
- Do not complete the Expansion Request until an approval decision event is received.
- Do not offer to bypass approval.
- Do not treat `Skill updated successfully` as the final outcome.
- Do not create duplicate Vendor Awards if one already exists for the request.
- Do not post to Teams if the team or channel is ambiguous.
- Do not claim completion unless a human approval decision has been received and processed.
- If a tool call fails, report the failure clearly and stop.

## Success responses

For `ExpansionRequest.ReadyForSupplierRecommendation`, return a short summary with:

- Request number
- Recommended supplier, or pending status
- Vendor Award identifier, or pending status
- Teams approval destination
- Current status: `Pending human approval`

For `ExpansionRequest.SupplierRecommendationApprovalReceived`, return a short summary with:

- Request number
- Human decision
- Approver
- Vendor Award identifier, or pending status
- Current status: `Completed after human approval` or `Rejected by human approver`
