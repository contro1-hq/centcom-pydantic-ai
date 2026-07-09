# centcom-pydantic-ai

Human approval skill for Pydantic AI deferred tools, connecting approval-required tool calls to Contro1 decisions, callbacks, and audit evidence.

Website: https://contro1.com

Documentation: https://contro1.com/docs/pydantic-ai-human-approval

Repository description:

Human approval skill for Pydantic AI deferred tools, connecting approval-required tool calls to Contro1 decisions, callbacks, and audit evidence.

## Installation / Usage

Copy `skills/centcom-pydantic-ai.md` into the skill library used by your coding agent or implementation assistant.

Use it when adding Contro1 approval handling to Pydantic AI tools marked with `requires_approval=True` or to deferred tool requests that need an external human decision.

## What this skill helps with

- Creating approval requests from Pydantic AI deferred tool calls.
- Optional Control Map previews for high-risk role routing and quorum workflows.
- Using `external_request_id` for idempotent tool-call review.
- Using `correlation_id` to keep a Pydantic AI run timeline together.
- Handling signed callback verification before returning deferred tool results.
- Producing audit-ready evidence for approved, denied, timed-out, and autonomous actions.

## Security note

Production approvals must go through Contro1 APIs and signed webhooks.
