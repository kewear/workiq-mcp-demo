---
name: expansion-request-supplier-recommendation
description: Use when an event has eventType ExpansionRequest.ReadyForSupplierRecommendation. Processes the Expansion Request supplier recommendation, sends an actionable Work IQ Teams approval message with Approve and Reject options, waits for the decision returned by the workflow, and then completes or rejects the award.
---

# Expansion Request Supplier Recommendation

## Required skill-found acknowledgement

When this skill is selected, the first assistant message must explicitly confirm that the skill was found and selected:

```text
Found skill: expansion-request-supplier-recommendation. I will process Expansion Request <requestNumber>, request human approval in Teams, wait for the approval decision, and then complete or reject the award.
```

Do not start by saying that you will search for Work IQ paths. Skill selection has already happened.

## Purpose

Process an Expansion Request business event when the request is ready for supplier recommendation. The skill recommends a supplier, initiates or proposes the Vendor Award process in Dataverse, sends a human approval message to Teams through Work IQ, waits for the returned approval decision, and then completes or rejects the award based on that decision.

Human approval is required before the Expansion Request or Vendor Award can be completed.

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

Use the configured `D365AITour` Dataverse environment. Do not ask the user or event payload for the Dataverse environment. The event contains business context only. The skill owns the environment and tool binding.

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
5. Treat responses such as `Skill updated successfully`, `acknowledged`, or `workflow initiated` as intermediate acknowledgements only. They are not completion.
6. After the Dataverse skill acknowledgement, fetch the Expansion Request, supplier recommendation, and Vendor Award details again if tools are available.
7. Capture the recommended supplier, award amount if available, recommendation rationale, and proposed Vendor Award identifier if available.
8. Use the Work IQ MCP server to post an actionable Teams message with options.
9. Wait for the workflow to return the selected option from the Teams message.
10. If the decision is `Approve`, finalize or complete the Vendor Award and mark the Expansion Request as awarded or completed using the available Dataverse action or record update.
11. If the decision is `Reject`, mark the supplier recommendation or Expansion Request as rejected and do not finalize the Vendor Award.
12. Return the final outcome.

## Teams approval message with options

The Teams approval step is not an informational notification. It must be a Work IQ message with options so the workflow can wait for and pass back the human decision.

Use Work IQ to post the message to:

- Team: `Expansion Requests`
- Channel: `Requests`

The message must include:

- Expansion Request number
- Recommended supplier, or `Pending` if not available yet
- Award amount, or `Not available yet`
- Reason for recommendation, or `Pending`
- Vendor Award identifier, if created, or `Pending`
- Clear prompt asking the human approver to choose an action

The options must be exactly:

- `Approve`
- `Reject`

Before posting, resolve the Team ID and Channel ID. If exactly one matching team and channel is found, post the options message. If the destination is ambiguous or not found, return an error and do not post.

## Approval decision handling

After posting the Work IQ Teams message with options, wait for the selected option that the workflow returns.

If the selected option is `Approve`:

1. Confirm the request is still eligible to be awarded.
2. Complete or finalize the Vendor Award using the available Dataverse action or record update.
3. Mark the Expansion Request as awarded, approved, or completed using the available Dataverse action or record update.
4. Return status `Awarded after human approval`.

If the selected option is `Reject`:

1. Confirm the request exists.
2. Mark the supplier recommendation or approval state as rejected using the available Dataverse action or record update.
3. Do not finalize the Vendor Award.
4. Return status `Rejected by human approver`.

If the Work IQ options message is posted but no decision is returned in the current workflow execution, return status `Pending human approval` and do not claim the award is complete.

## Safety rules

- Do not finalize the Vendor Award before the human selects `Approve`.
- Do not complete the Expansion Request before the human selects `Approve`.
- Do not offer to bypass approval.
- Do not treat `Skill updated successfully` as the final outcome.
- Do not create duplicate Vendor Awards if one already exists for the request.
- Do not post to Teams if the team or channel is ambiguous.
- Do not claim completion unless the human approval decision has been received and processed.
- If a tool call fails, report the failure clearly and stop.

## Success response

Return a short summary with:

- Request number
- Recommended supplier, or pending status
- Human decision: `Approve`, `Reject`, or `Pending`
- Vendor Award identifier, or pending status
- Teams approval destination
- Current status: `Awarded after human approval`, `Rejected by human approver`, or `Pending human approval`
