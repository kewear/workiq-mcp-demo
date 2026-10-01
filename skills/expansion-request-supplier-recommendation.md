# Expansion Request Supplier Recommendation

## Purpose

Process an Expansion Request business event when the request is ready for supplier recommendation. The skill recommends a supplier, initiates or proposes the Vendor Award process in Dataverse, and posts a Teams approval request through Work IQ.

The Teams approval is the gate before Vendor Award finalization.

## Event handled

`ExpansionRequest.ReadyForSupplierRecommendation`

## Inputs

The event payload must include:

- `eventType`
- `requestNumber`
- `status`

Example payload:

```json
{
  "eventType": "ExpansionRequest.ReadyForSupplierRecommendation",
  "requestNumber": "EXP-2026-004",
  "status": "Ready for supplier recommendation"
}
```

## Environment binding

Use the configured `D365AITour` Dataverse environment.

Do not ask the user or event payload for the Dataverse environment. The event contains business context only. The skill owns the environment and tool binding.

## Eligibility checks

Proceed only when all of the following are true:

- `eventType` equals `ExpansionRequest.ReadyForSupplierRecommendation`
- `status` equals `Ready for supplier recommendation`
- `requestNumber` is present

If any eligibility check fails, stop and return a concise explanation of why no action was taken.

## Business process

1. Find the Expansion Request by `requestNumber`.
2. Confirm the request exists.
3. Confirm the request is ready for supplier recommendation.
4. Run the Dataverse skill `cr2d6_skill_recommend_supplier_and_initiate_vendor_award`.
5. Confirm the supplier recommendation step succeeded.
6. Capture the recommended supplier, award amount if available, recommendation rationale, and proposed Vendor Award identifier if available.
7. Post a Teams approval request using Work IQ.
8. Stop after the Teams approval request is posted. Do not finalize the Vendor Award.

## Approval gate rule

Teams approval is required before Vendor Award finalization. Posting the Teams approval request is mandatory after the supplier recommendation step succeeds.

Do not wait for finalized Vendor Award records before posting the approval. If the Vendor Award identifier, recommended supplier, award amount, or rationale are not available yet, include `Pending` or `Not available yet` for those fields and still post the Teams approval request.

The process is not complete until the Teams approval request has been posted or the Teams post fails with a clear error.

## Teams approval destination

- Team: `Expansion Requests`
- Channel: `Requests`

The Teams approval message must include:

- Expansion Request number
- Recommended supplier, or `Pending` if not available yet
- Award amount, or `Not available yet`
- Reason for recommendation, or `Pending`
- Vendor Award identifier, if created, or `Pending`
- Approval options: `Approve`, `Reject`

Before posting, resolve the Team ID and Channel ID. If exactly one matching team and channel is found, post the approval request. If the destination is ambiguous or not found, return an error and do not post.

## Safety rules

- Do not finalize the Vendor Award.
- Do not create duplicate Vendor Awards if one already exists for the request.
- Do not post to Teams if the team or channel is ambiguous.
- Do not claim completion unless the Dataverse recommendation step and Teams post both succeeded.
- If a tool call fails, report the failure clearly and stop.

## Success response

Return a short summary with:

- Request number
- Recommended supplier, or pending status
- Vendor Award identifier, or pending status
- Teams approval destination
- Current status