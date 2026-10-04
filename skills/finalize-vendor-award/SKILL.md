---
name: finalize-vendor-award
description: Use after Human_In_the_loop_approval returns Approve for an Expansion Request supplier recommendation. Finalizes the Vendor Award only through an exposed executable business skill/action.
---

# Finalize Vendor Award

## Purpose

Finalize an approved Vendor Award for an Expansion Request after human approval has returned `Approve`.

This skill is approval-gated. It must never run before approval.

## Inputs

The caller should provide:

- `requestNumber`
- `approvalDecision`
- `recommendedSupplier`, if available
- `vendorAwardId`, if available
- `awardAmount`, if available
- `reason`, if available

## Eligibility checks

Proceed only when all of the following are true:

- `approvalDecision` equals `Approve`
- `requestNumber` is present

If the approval decision is `Reject`, missing, ambiguous, or failed, do not finalize and return a concise explanation.

## Environment binding

Use the configured `D365AITour` Dataverse environment.

Do not ask the user or event payload for Dataverse environment details. The skill owns the environment and tool binding.

## Required execution order

1. Find the Expansion Request by `requestNumber`.
2. Read related recommendation and Vendor Award context if not provided by the caller.
3. Confirm the human approval decision is `Approve`.
4. Discover the finalization business skill by name.
5. Invoke the finalization business skill only if an executable action/tool is exposed.
6. Verify the finalization result by reading the Expansion Request, Vendor Award, or related task context.
7. Return the final status.

## Work IQ and Business Applications rules

Use Work IQ MCP for Business Applications discovery and actions.

Business skills are discoverable by name. Do not assume they are Dataverse Custom APIs. Do not invent `/customapis/...` paths.

Award finalization business skill names to search for:

- `Finalize Vendor Award`
- `cr2d6_skill_finalize_vendor_award`

To run a business skill, use only an executable action/tool path that is actually exposed in the current run. Valid executable surfaces include an attached workflow tool, a returned Business Applications action path, or another concrete action path returned by Work IQ discovery with an `action` operation.

If discovery returns only skill metadata with `fetch`, `update`, or `delete`, do not treat that as execution.

If no executable finalization action is exposed, return this outcome:

`Approval received, but no executable finalization action is exposed in this environment. Vendor Award was not finalized.`

Do not claim finalization succeeded unless an executable finalization action completed successfully and the final state was verified.

## Safety rules

- Never finalize unless approvalDecision is `Approve`.
- Never finalize after `Reject`.
- Do not create, edit, update, publish, or overwrite business skill definitions.
- Do not invent a Vendor Award id.
- Do not report metadata discovery as finalization.
- If a tool call fails, report the failure clearly and stop.

## Success response

Return a short summary with:

- Request number
- Recommended supplier, if known
- Approval decision
- Vendor Award identifier, if known
- Finalization action used
- Finalization verification result
- Current status