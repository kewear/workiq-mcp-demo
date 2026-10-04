---
name: finalize-vendor-award
description: Use after Human_In_the_loop_approval returns Approve for an Expansion Request supplier recommendation. Executes the Business Applications skill cr2d6_skill_finalize_vendor_award to finalize the Vendor Award.
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

## Required finalization action

Invoke the Business Applications business skill named:

`cr2d6_skill_finalize_vendor_award`

This is the required finalization action. The display name may be `Finalize Vendor Award`, but the logical skill name is `cr2d6_skill_finalize_vendor_award`.

Do not stop after discovering metadata for this skill. Metadata confirms the skill exists, but metadata is not execution.

The process is incomplete until one of these is true:

- `cr2d6_skill_finalize_vendor_award` executed successfully and final state was verified.
- The connected runtime explicitly returns that execution of `cr2d6_skill_finalize_vendor_award` is unavailable or failed.

## Required execution order

1. Find the Expansion Request by `requestNumber`.
2. Read related recommendation and Vendor Award context if not provided by the caller.
3. Confirm the human approval decision is `Approve`.
4. Discover the Business Applications execution surface for `cr2d6_skill_finalize_vendor_award`.
5. Execute `cr2d6_skill_finalize_vendor_award` with the request context.
6. Verify the finalization result by reading the Expansion Request, Vendor Award, or related task context.
7. Return the final status.

## Work IQ and Business Applications rules

Use Work IQ MCP for Business Applications discovery and actions.

Business skills are discoverable by name. Do not assume they are Dataverse Custom APIs. Do not invent `/customapis/...` paths.

Search for these finalization business skill names:

- `cr2d6_skill_finalize_vendor_award`
- `Finalize Vendor Award`

If discovery returns a metadata path for `cr2d6_skill_finalize_vendor_award` with only `fetch`, `update`, or `delete`, do not treat that as the final result. Continue looking for the Business Applications skill execution surface for `cr2d6_skill_finalize_vendor_award`.

If the runtime exposes a generic business-skill execution action that accepts a skill logical name, use it with `cr2d6_skill_finalize_vendor_award`.

If no execution surface is exposed after discovery, return this exact outcome:

`Approval received, but execution of cr2d6_skill_finalize_vendor_award is not exposed in this environment. Vendor Award was not finalized.`

Do not claim finalization succeeded unless `cr2d6_skill_finalize_vendor_award` executed successfully and the final state was verified.

## Safety rules

- Never finalize unless `approvalDecision` is `Approve`.
- Never finalize after `Reject`.
- Do not create, edit, update, publish, or overwrite business skill definitions.
- Do not invent a Vendor Award id.
- Do not report metadata discovery as finalization.
- If the execution surface is unavailable, report that execution is unavailable, not that the skill does not exist.
- If a tool call fails, report the failure clearly and stop.

## Success response

Return a short summary with:

- Request number
- Recommended supplier, if known
- Approval decision
- Vendor Award identifier, if known
- Finalization skill used: `cr2d6_skill_finalize_vendor_award`
- Finalization verification result
- Current status