---
name: centcom-pydantic-ai
description: Connect Pydantic AI deferred tools and approval-required tool calls to Contro1 decisions, callbacks, and audit evidence.
user_invocable: true
---

# Contro1 + Pydantic AI Skill

Use this skill when integrating Contro1 approval workflows into a Pydantic AI codebase.

## Rules

- Mark risky tools with `requires_approval=True`.
- Convert `DeferredToolRequests` into Contro1 approval requests.
- Use the Pydantic AI run ID or conversation ID as `correlation_id`.
- Include the tool call ID in `external_request_id` so retries are idempotent.
- Call Control Map before high-risk deferred tools to confirm required roles, quorum, separation of duties, and fallback routing are satisfiable.
- Return deferred tool results only after the Contro1 decision has been verified.
- Treat rejected, cancelled, timed_out, invalid signatures, and unknown request IDs as fail-closed for production actions.
- Log approved follow-up actions with `in_reply_to` pointing at the Contro1 request.

## Required configuration

```bash
CENTCOM_API_KEY=cc_live_your_key
CENTCOM_BASE_URL=https://api.contro1.com/api/centcom/v1
CENTCOM_WEBHOOK_SECRET=whsec_your_secret
CENTCOM_CALLBACK_URL=https://your-app.example.com/webhooks/contro1
```

## 1. Short example: mark a tool for approval

```python
from pydantic_ai import Agent

agent = Agent("openai:gpt-5")

@agent.tool_plain(requires_approval=True)
def issue_refund(customer_id: str, amount: int) -> str:
    return billing.refund(customer_id, amount)
```

## 2. Short example: create a Contro1 request

Create one Contro1 request per approval-required tool call:

```python
request = await centcom.create_request(
    type="approval",
    question=f"Approve Pydantic AI tool: {call.tool_name}?",
    context={"args": call.args},
    callback_url=os.environ["CENTCOM_CALLBACK_URL"],
    required_role="manager",
    external_request_id=f"pydantic-ai:{run_id}:{call.tool_call_id}",
    correlation_id=run_id,
    metadata={
        "integration": "pydantic-ai",
        "tool_name": call.tool_name,
        "tool_call_id": call.tool_call_id,
    },
)
```

Return approval results by tool call ID only after the signed Contro1 decision is verified.

## 3. Full deferred approval handler

A production handler should create the approval request, wait for a verified signed decision, and return the deferred result.

```python
async def handle_deferred_approvals(run_id: str, requests):
    approvals = {}
    for call in requests.approvals:
        request = await centcom.create_request(
            type="approval",
            question=f"Approve Pydantic AI tool: {call.tool_name}?",
            context={"args": call.args},
            callback_url=os.environ["CENTCOM_CALLBACK_URL"],
            required_role="manager",
            external_request_id=f"pydantic-ai:{run_id}:{call.tool_call_id}",
            correlation_id=run_id,
            metadata={"integration": "pydantic-ai", "tool_name": call.tool_name},
        )

        decision = await wait_for_signed_decision(request["id"])
        approved = bool(decision.get("response", {}).get("approved"))
        approvals[call.tool_call_id] = approved

    return requests.build_results(approvals=approvals)
```

## 4. If routing fails, check who is available

Most integrations do not need Control Map in the normal approval path. If a request cannot be routed, times out unexpectedly, or your app wants to show a clear operational error, call Control Map to see who is currently available.

```python
preview = await centcom.post("/requests/control-map", {
    "approval_requirements": {
        "required_roles": ["manager"],
        "required_approvals": 1,
    },
    "metadata": {
        "integration": "pydantic-ai",
        "tool_name": call.tool_name,
        "tool_call_id": call.tool_call_id,
    },
})

print(preview["satisfiable"])            # can this request be routed?
print(preview.get("on_shift_capacity"))  # who is currently available?
print(preview.get("fallback_reviewers")) # who can receive fallback routing?
print(preview.get("warnings"))           # why routing may fail
```

## 5. Log autonomous and post-approval tool actions

Every approval request already stores the reviewer decision in Contro1. Use audit records for low-risk tools that did not need approval, and optionally after an approved tool completes if you want to record what your Pydantic AI system actually did next.

```python
await centcom.log_action(
    action="pydantic_ai.tool_allowed",
    summary="Ran read-only account lookup",
    source={"integration": "pydantic-ai", "run_id": run_id},
    outcome="success",
    severity="info",
    correlation_id=run_id,
)
```

For a post-approval tool result, link the audit record back to the approval request:

```python
await centcom.log_action(
    action="pydantic_ai.tool_completed",
    summary=f"Completed approved tool {call.tool_name}",
    source={"integration": "pydantic-ai", "run_id": run_id},
    outcome="success",
    correlation_id=run_id,
    in_reply_to={"type": "request", "id": request_id},
)
```

## 6. Get evidence

Use evidence when compliance or incident review needs proof of what happened.

```python
evidence = await centcom.get(f"/requests/{request_id}/evidence")
timeline = await centcom.get(f"/cases/{run_id}")
```

- Request evidence shows one reviewed tool call: context, policy, reviewer, decision, callback, timestamps, and final response.
- Case timeline shows all approvals and audit records that share the same `correlation_id`.

## Reference links

- Website: https://contro1.com
- Documentation: https://contro1.com/docs/pydantic-ai-human-approval
- Repo: https://github.com/contro1-hq/centcom-pydantic-ai
- Pydantic AI deferred tools: https://pydantic.dev/docs/ai/tools-toolsets/deferred-tools
- Contro1 webhooks: https://contro1.com/docs/webhooks
