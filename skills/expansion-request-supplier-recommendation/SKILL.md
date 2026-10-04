---
name: expansion-request-supplier-recommendation
description: Use when an event has eventType ExpansionRequest.ReadyForSupplierRecommendation. Processes the Expansion Request supplier recommendation, calls Human_In_the_loop_approval with Approve and Reject options, waits for the decision returned by the workflow, and then completes or rejects the award.
---

# Expansion Request Supplier Recommendation

## Skill selection rule

When this skill is selected, it is acceptable to briefly confirm that the skill was found:

```text
Found skill: expansion-request-supplier-recommendation. I will process Expansion Request <requestNumber>, request human approval in Teams, wait for the approval decision, and then complete or reject the award.
```

That acknowledgement is not a valid final response. Do not return it as the whole result. Continue immediately to the supplier recommendation Custom API, then call `Human_In_the_loop_approval`.

Do not start by saying that you will search for Work IQ paths. Skill selection has already happened.

If the agent starts by saying it will discover endpoints, trigger a recommend-suppliers action, update status, monitor for records, or proceed to finalize later, that is the wrong flow. The correct flow is to follow this GitHub process skill, invoke the supplier recommendation Custom API, call `Human_In_the_loop_approval`, and only then finalize on `Approve`.

## Purpose

Process an Expansion Request business event when the request is ready for supplier recommendation. The skill recommends a supplier, initiates or proposes the Vendor Award process in Dataverse, calls the workflow tool `Human_In_the_loop_approval`, waits for the returned approval decision, and then completes or rejects the award based on that decision.

Human approval is required before the Expansion Request or Vendor Award can be completed.

## Non-negotiable execution checkpoints

This skill is not complete after the Dataverse business skill runs. The required execution order is:

1. Find the Expansion Request.
2. Invoke the supplier recommendation business skill through the Work IQ MCP server.
3. Call `Human_In_the_loop_approval` so the human decision can be returned to the workflow.
4. Wait for the returned option if the workflow provides one.
5. If the returned option is `Approve`, invoke the award finalization business skill through the Work IQ MCP server.
6. If the returned option is `Reject`, mark the request or recommendation rejected and do not finalize the award.

Do not send a final assistant response after step 2. After the supplier recommendation business skill is invoked, immediately proceed to the Work IQ Teams options call. A response that only says the supplier recommendation business skill succeeded is incomplete and must be treated as a failed run.

The required Teams call must produce buttons that can be acted on in Teams and return the selected decision to the workflow. Do not substitute a plain Teams channel message, a visual-only Adaptive Card, a Dataverse update, or a suggestion to fetch details later.

After the supplier recommendation business skill returns, do not make any more Business Applications or Dataverse calls until the response-capable Teams approval has been posted.

Never end the run with any of these responses:

- "Want me to monitor for the award record?"
- "Want me to proceed to run Finalize Vendor Award?"
- "No new Vendor Award records created yet."
- "Supplier recommendation initiated" without also posting the Teams approval options message.

Those responses skipped the required Work IQ Teams approval call.

## Skill invocation rules

Business skills are runtime actions. Do not create, edit, update, publish, or overwrite skill definitions as part of this process.

Use Work IQ MCP to invoke business skills only:

- Invoke the supplier recommendation business skill: `cr2d6_skill_recommend_supplier_and_initiate_vendor_award`
- After Teams approval, invoke the business skill that finalizes or awards the vendor. If the exact logical name is not already known, discover the available business skills and select the one whose purpose is to finalize or award the approved Vendor Award.

### Executable business skill paths

The Business Applications `/skills/{skillName}` resource is metadata only. Do not try to execute a business skill by fetching or updating `/businessapps/environments/{environmentId}/skills/{skillName}`.

Invoke the business skills as Dataverse Custom APIs through Work IQ MCP `do_action`:

- Supplier recommendation action URL:
  `/businessapps/environments/D365AITour005/customapis/cr2d6_skill_recommend_supplier_and_initiate_vendor_award`
- Award finalization action URL:
  `/businessapps/environments/D365AITour005/customapis/cr2d6_skill_finalize_vendor_award`

Before invoking either Custom API, call `get_schema` once on the exact Custom API action URL with `operationType: "action"` to confirm the request body shape and parameter casing. Then call `do_action` on that same Custom API action URL.

For supplier recommendation, the request body must include the event's `requestNumber` and a short business description. Use schema-confirmed field names. If the schema exposes compatible names, use:

```json
{
  "RequestNumber": "EXP-2026-004",
  "Description": "Recommend a supplier and initiate the Vendor Award for Expansion Request EXP-2026-004."
}
```

If the schema uses different casing, preserve the schema-confirmed casing. Do not omit the description if the schema requires it.

For award finalization, invoke `/businessapps/environments/D365AITour005/customapis/cr2d6_skill_finalize_vendor_award` only after the Teams approval decision is `Approve`.

The award finalization business skill is approval-gated. It may be invoked only after the Work IQ Teams options response returns `Approve`. Never invoke the award finalization business skill while the decision is pending, missing, rejected, ambiguous, or failed.

Do not use Work IQ `ask` to decide whether the GitHub process skill or finalization skill exists after approval. The approval decision is already the control signal. On `Approve`, call the exact finalization Custom API path above with Work IQ MCP `do_action`.

Do not report tool metadata as the business outcome. The business outcome must be one of: Teams approval requested, awarded after human approval, rejected by human approver, or a clear failure.

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
4. Invoke the Dataverse supplier recommendation Custom API through Work IQ MCP `do_action` at `/businessapps/environments/D365AITour005/customapis/cr2d6_skill_recommend_supplier_and_initiate_vendor_award`.
5. Treat tool status responses as invocation acknowledgements only. They are not completion, and they must not be reported as the final outcome.
6. After the supplier recommendation business skill acknowledgement, continue to the approval step. Fetch the Expansion Request and recommendation details again if tools are available, but do not skip approval if some generated details are incomplete.
7. Capture the recommended supplier, the Expansion Request budgeted amount, recommendation rationale, and proposed Vendor Award identifier if available.
8. Call the workflow tool `Human_In_the_loop_approval` to request human approval with `Approve` and `Reject` options.
9. Wait for `Human_In_the_loop_approval` to return the selected option.
10. If the decision is `Approve`, invoke the award finalization Custom API through Work IQ MCP `do_action` at `/businessapps/environments/D365AITour005/customapis/cr2d6_skill_finalize_vendor_award`, then confirm the Expansion Request or Vendor Award is awarded or completed.
11. If the decision is `Reject`, mark the supplier recommendation or Expansion Request as rejected and do not finalize the Vendor Award.
12. Return the final outcome.

## Human-in-the-loop approval

The approval step is not an informational notification. It must be response-capable so the workflow can wait for and pass back the human decision.

Use the workflow tool named `Human_In_the_loop_approval`. Do not hand-roll Teams Adaptive Cards through Work IQ for this approval gate unless the tool is unavailable and the user explicitly asks for a fallback.

Teams delivery/rendering is owned by the workflow tool. The agent must provide the business fields below to `Human_In_the_loop_approval`; it must not post, format, or manage the Teams approval card directly.

Call `Human_In_the_loop_approval` immediately after the supplier recommendation business skill is acknowledged. The tool input must include:

- Approval title: `Supplier Recommendation Approval`
- Request number
- Recommended vendor/supplier. Use the actual recommended supplier when available. If the supplier recommendation output is incomplete, re-read the Expansion Request and related supplier fields before falling back to `Pending`.
- Budgeted amount for the Expansion Request, or `Not available yet` if the request does not expose it.
- Award amount, if different from the budgeted amount, or `Not available yet`
- Reason for recommendation, or `Pending`
- Vendor Award identifier, if created, or `Pending`
- Options: `Approve`, `Reject`

For `EXP-2026-004`, prior successful runs identified the recommended supplier as `Atlas Regional Manufacturing` and the Expansion Request budgeted amount as `$9,400,000`. Use live data when available, but do not omit these fields from the approval request.

If the tool has destination fields, use:

- Team: `Expansion Requests`
- Channel: `Requests`

The approval tool must return one of:

- `Approve`
- `Reject`

If `Human_In_the_loop_approval` is not available or fails, stop and report the failure clearly. Do not post a plain Teams message and do not finalize the award.

## Approval decision handling

After calling `Human_In_the_loop_approval`, wait for the selected option that the workflow returns.

If the selected option is `Approve`:

1. Confirm the request is still eligible to be awarded.
2. Call `get_schema` once on `/businessapps/environments/D365AITour005/customapis/cr2d6_skill_finalize_vendor_award` with `operationType: "action"` to confirm parameter casing.
3. Call Work IQ MCP `do_action` on `/businessapps/environments/D365AITour005/customapis/cr2d6_skill_finalize_vendor_award`.
4. Include the approved request number and approval decision in the request body using schema-confirmed field names. If the schema exposes compatible names, use:

```json
{
  "RequestNumber": "EXP-2026-004",
  "Decision": "Approve",
  "Description": "Finalize the Vendor Award for approved Expansion Request EXP-2026-004."
}
```

5. Do not use Work IQ `ask` or skill metadata fetch as a substitute for the finalization `do_action`.
6. If the finalization Custom API call fails because the endpoint is not exposed, report that exact failure. Do not claim the award is finalized.
7. Confirm the Expansion Request or Vendor Award is awarded, approved, or completed.
8. Return status `Awarded after human approval`.

If the selected option is `Reject`:

1. Confirm the request exists.
2. Mark the supplier recommendation or approval state as rejected using the available Dataverse action or record update.
3. Do not finalize the Vendor Award.
4. Return status `Rejected by human approver`.

If the Work IQ options message is posted but no decision is returned in the current workflow execution, return status `Pending human approval` and do not claim the award is complete.

## Safety rules

- Do not finalize the Vendor Award before the human selects `Approve`.
- Do not invoke the award finalization business skill before the human selects `Approve`.
- Do not invoke the award finalization business skill when the human selects `Reject`.
- Do not complete the Expansion Request before the human selects `Approve`.
- Do not offer to bypass approval.
- Do not create, update, or publish skill definitions during this process.
- Do not report tool metadata as the final business outcome.
- Do not stop after the Dataverse business skill succeeds.
- Do not make additional Business Applications or Dataverse calls after the supplier recommendation business skill until `Human_In_the_loop_approval` returns a decision or fails.
- Do not post a plain Teams channel message.
- Do not post a visual-only Adaptive Card with `Action.Submit` through normal Teams channel message creation.
- Do not ask the user whether to call `Human_In_the_loop_approval`. Calling it is mandatory.
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
