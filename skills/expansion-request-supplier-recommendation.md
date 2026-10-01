# Expansion Request Supplier Recommendation

## Purpose

Process an Expansion Request business event when the request is ready for supplier recommendation. The skill recommends a supplier, initiates the Vendor Award process in Dataverse, and posts a Teams approval request through Work IQ.

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
5. Confirm the Vendor Award was initiated.
6. Capture the recommended supplier, award amount if available, recommendation rationale, and Vendor Award identifier.
7. Use Work IQ to post a Teams approval request.

## Teams approval destination

- Team: `Expansion Requests`
- Channel: `Requests`

The Teams approval message must include:

- Expansion Request number
- Recommended supplier
- Award amount, if available
- Reason for recommendation
- Vendor Award identifier, if created
- Approval options: `Approve`, `Reject`

Before posting, resolve the Team ID and Channel ID. If exactly one matching team and channel is found, post the approval request. If the destination is ambiguous or not found, return an error and do not post.

## Safety rules

- Do not finalize the Vendor Award.
- Do not create duplicate Vendor Awards if one already exists for the request.
- Do not post to Teams if the team or channel is ambiguous.
- Do not claim completion unless the Dataverse action and Teams post both succeeded.
- If a tool call fails, report the failure clearly and stop.

## Success response

Return a short summary with:

- Request number
- Recommended supplier
- Vendor Award identifier
- Teams approval destination
- Current status