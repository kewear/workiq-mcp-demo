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

If the agent starts by saying it will discover endpoints, trigger a recommend-suppliers action, update status, monitor for records, or proceed to finalize later, that is the wrong flow. The correct flow is to follow this GitHub process skill and call Work IQ Teams with options immediately after the supplier recommendation business skill runs.

## Purpose

Process an Expansion Request business event when the request is ready for supplier recommendation. The skill recommends a supplier, initiates or proposes the Vendor Award process in Dataverse, sends a human approval message to Teams through Work IQ, waits for the returned approval decision, and then completes or rejects the award based on that decision.

Human approval is required before the Expansion Request or Vendor Award can be completed.

## Non-negotiable execution checkpoints

This skill is not complete after the Dataverse business skill runs. The required execution order is:

1. Find the Expansion Request.
2. Invoke the supplier recommendation business skill through the Work IQ MCP server.
3. Post a response-capable Teams approval through Work IQ so the human decision can be returned to the workflow.
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

The award finalization business skill is approval-gated. It may be invoked only after the Work IQ Teams options response returns `Approve`. Never invoke the award finalization business skill while the decision is pending, missing, rejected, ambiguous, or failed.

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
4. Invoke the Dataverse business skill `cr2d6_skill_recommend_supplier_and_initiate_vendor_award` through the Work IQ MCP server.
5. Treat tool status responses as invocation acknowledgements only. They are not completion, and they must not be reported as the final outcome.
6. After the supplier recommendation business skill acknowledgement, continue to the Teams options step. Fetch the Expansion Request, supplier recommendation, and Vendor Award details again if tools are available, but do not skip Teams if those details are incomplete.
7. Capture the recommended supplier, award amount if available, recommendation rationale, and proposed Vendor Award identifier if available.
8. Post a response-capable Teams approval with `Approve` and `Reject` options through Work IQ.
9. Wait for the workflow to return the selected option from the Teams message.
10. If the decision is `Approve`, invoke the award finalization business skill through the Work IQ MCP server, then confirm the Expansion Request or Vendor Award is awarded or completed.
11. If the decision is `Reject`, mark the supplier recommendation or Expansion Request as rejected and do not finalize the Vendor Award.
12. Return the final outcome.

## Teams approval message with options

The Teams approval step is not an informational notification. It must be response-capable so the workflow can wait for and pass back the human decision.

Do not use a Teams Adaptive Card with `Action.Submit` when the card is posted through a normal channel message. In this runtime, that renders buttons but Teams returns "That action isn't supported here" when the user clicks them.

Use one of these response-capable patterns:

1. Prefer the Work IQ approval or post-with-options API if it is available. It must create Teams buttons and return the selected option to the workflow.
2. If using a normal Teams channel message via Work IQ `create_entity`, the Adaptive Card actions must be callback-capable `Action.OpenUrl` buttons pointing to Logic Apps approval callback URLs supplied by the workflow, one for `Approve` and one for `Reject`.

If neither a Work IQ approval/post-with-options API nor callback URLs are available, stop with:

```text
Blocked: response-capable Teams approval is not available. Do not post a visual-only card.
```

For a Teams channel message with callback URLs, first resolve the destination:

1. `fetch` `/me/joinedTeams?$select=id,displayName` and select the exact team `Expansion Requests`.
2. `fetch` `/teams/{teamId}/channels?$select=id,displayName` and select the exact channel `Requests`.
3. `create_entity` with `parentUrl` `/teams/{teamId}/channels/{channelId}/messages`.

The `jsonBody` must include both `body` and an Adaptive Card attachment. The posted card must render buttons like this:

- `Approve`
- `Reject`

The approval card title must be:

```text
Supplier Recommendation Approval
```

The approval card body must include:

- Request Number
- Recommended Supplier
- Award Amount
- Reason
- Vendor Award Id

The options must be exactly `Approve` and `Reject`.

Use this body shape only when approval callback URLs are available from the workflow. Replace `APPROVE_CALLBACK_URL` and `REJECT_CALLBACK_URL` with the actual URLs supplied by Logic Apps.

Graph-backed Teams messages require a matching attachment placeholder in the HTML body. The `<attachment id="approvalCard"></attachment>` placeholder must match the attachment `id` value exactly.

```json
{
  "body": {
    "contentType": "html",
    "content": "Supplier recommendation for Expansion Request EXP-2026-004. Please review and choose Approve or Reject.<br/><attachment id=\"approvalCard\"></attachment>"
  },
  "attachments": [
    {
      "id": "approvalCard",
      "contentType": "application/vnd.microsoft.card.adaptive",
      "contentUrl": null,
      "content": "{\"$schema\":\"http://adaptivecards.io/schemas/adaptive-card.json\",\"type\":\"AdaptiveCard\",\"version\":\"1.4\",\"body\":[{\"type\":\"TextBlock\",\"size\":\"Large\",\"weight\":\"Bolder\",\"text\":\"Supplier Recommendation Approval\"},{\"type\":\"FactSet\",\"facts\":[{\"title\":\"Request Number\",\"value\":\"EXP-2026-004\"},{\"title\":\"Recommended Supplier\",\"value\":\"Pending\"},{\"title\":\"Award Amount\",\"value\":\"Not available yet\"},{\"title\":\"Reason\",\"value\":\"Pending\"},{\"title\":\"Vendor Award Id\",\"value\":\"Pending\"}]},{\"type\":\"TextBlock\",\"wrap\":true,\"text\":\"Select an action to proceed.\"}],\"actions\":[{\"type\":\"Action.OpenUrl\",\"title\":\"Approve\",\"url\":\"APPROVE_CALLBACK_URL\"},{\"type\":\"Action.OpenUrl\",\"title\":\"Reject\",\"url\":\"REJECT_CALLBACK_URL\"}],\"msteams\":{\"width\":\"full\"}}"
    }
  ]
}
```

Do not call `create_entity` with only a `body` field. A plain channel message without an approval card does not satisfy this skill.

Do not post an Adaptive Card that uses `Action.Submit` through normal Teams channel message creation. That produces unsupported buttons in this runtime.

This call must be made through the Work IQ MCP server, not by asking the user to send a Teams message and not by using a non-Work-IQ connector. The visible evidence of success is that the run contains a Work IQ MCP call for Teams/channel messaging after the supplier recommendation business skill call.

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

Before posting, resolve the Team ID and Channel ID using the exact fetch calls above if the selected approval pattern requires them. If exactly one matching team and channel is found, post the approval card with response-capable buttons. If the destination is ambiguous or not found, return an error and do not post.

## Approval decision handling

After posting the Work IQ Teams message with options, wait for the selected option that the workflow returns.

If the selected option is `Approve`:

1. Confirm the request is still eligible to be awarded.
2. Invoke the business skill that finalizes or awards the approved Vendor Award through the Work IQ MCP server.
3. Confirm the Expansion Request or Vendor Award is awarded, approved, or completed.
4. Return status `Awarded after human approval`.

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
- Do not make additional Business Applications or Dataverse calls after the supplier recommendation business skill until a response-capable Work IQ Teams approval has been posted.
- Do not post a plain Teams channel message.
- Do not post a visual-only Adaptive Card with `Action.Submit` through normal Teams channel message creation.
- Do not ask the user whether to post the Teams approval message. Posting it is mandatory.
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
