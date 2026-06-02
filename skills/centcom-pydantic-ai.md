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

## Approval-required tool pattern

```python
from pydantic_ai import Agent

agent = Agent("openai:gpt-5")

@agent.tool_plain(requires_approval=True)
def issue_refund(customer_id: str, amount: int) -> str:
    return billing.refund(customer_id, amount)
```

## Deferred approval handling

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

## Audit logging

After an approved tool executes, log the result in the same case:

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

## Reference links

- Website: https://contro1.com
- Documentation: https://contro1.com/docs/pydantic-ai-human-approval
- Repo: https://github.com/contro1-hq/centcom-pydantic-ai
- Pydantic AI deferred tools: https://pydantic.dev/docs/ai/tools-toolsets/deferred-tools
- Contro1 webhooks: https://contro1.com/docs/webhooks
