---
name: build-action
description: Create, update, test, and attach Synthflow agent actions over MCP — custom HTTP actions, information extractors, transfers, SMS, real-time booking (Cal.com, GoHighLevel), IVR navigation, custom evaluations, and call-summary emails. Use this whenever the user wants to give an agent a new capability, connect it to an API or CRM, extract data from calls, add a transfer or SMS, set up booking, change an action's URL, body, or variables, or fix an action that isn't firing or is failing. Trigger on phrases like "add an action", "call my API during the call", "extract the caller's email", "set up a transfer", "connect booking", "update the webhook URL", or "the action isn't working".
---

# Build Action

## Purpose

Actions are what an agent can do beyond talking: call an API, extract data after the call, transfer, send an SMS, book, navigate an IVR, score the call, or email a summary. This skill builds them in the order that works:

**plan → check prerequisites → create → initialize/test → attach → verify on a call**

The MCP action tools have their own field names, which differ from the REST API and from the dashboard labels. Most failed action calls come from using the wrong field names, so follow the shapes below exactly, and read an existing action with `get_action` before changing it.

This playbook runs on the Synthflow MCP tools, whether they come from the plugin's bundled server or the official Synthflow connector.

## Tools this uses

- **`list_actions`**, **`get_action`** — find existing actions; read exact field names before any update.
- **`list_integrations`** — required before booking or any provider-backed action.
- **`test_api_call`** — call the user's endpoint directly to see the real response before building on it.
- **`create_action`** — `name`, `type`, `action_data`.
- **`update_action`** — `action_slug`, `update_data`, `confirmed`.
- **`initialize_action`** — sends a test request for a **custom** action and activates it. Required after creating or changing one.
- **`attach_action_to_agent`** / **`detach_action_from_agent`** — `model_id`, `action_slug`, `attachment_type` (`before`, `during`, `after`), `confirmed`.
- **`get_agent_actions`** — what's attached to an agent now.
- **`list_webhook_logs`**, **`get_webhook_log_detail`** — request and response of real action executions, for debugging.

Some tool descriptions mention `confirm_action` and `show_form`. Those UI tools don't exist over MCP:

- **Confirmation:** state the change in the chat (action, agent, timing, what changes) and get the user's explicit yes before calling a tool with `confirmed: true`.
- **Secrets:** never ask the user to paste an API key or token into the chat, and never put one in `action_data`. Create the action without the credential and have the user add it in the action's settings in the Synthflow dashboard.

## Step 0 — Plan

1. **Reuse first.** `list_actions` with a search term. If a similar action exists, prefer updating or reusing it over creating a duplicate. An action can be attached to several agents.
2. **Pick the type and timing.**

| Need | `type` | Usual `attachment_type` |
|---|---|---|
| Call an HTTP API (lookup, create ticket, write to CRM) | `custom` | `during`, or `before` to load data before the call |
| Pull a value out of the conversation (email, reason, intent) | `info` | `after` |
| Hand the caller to a person or number | `transfer` | `during` |
| Text the caller | `sms` | `during` |
| Book in Cal.com or GoHighLevel | `booking` | `during` |
| Navigate a phone menu with DTMF | `ivr` | `during` |
| Score the call against criteria | `custom_eval` | `after` |
| Email a summary of every completed call | `email` | `after` |

3. **Side effects.** If the action writes to a real system (creates orders, bookings, tickets, sends messages), say so and agree how to test it safely — a sandbox URL, a test record, or a test copy of the agent.

MCP-server actions (`type: "mcp"`) can't be created here — they're configured in the dashboard under Settings → MCP.

## Step 1 — Prerequisites

- **Booking and provider-backed actions:** call `list_integrations`. If the provider (Cal.com, GoHighLevel, …) is missing or `is_configured` is false, stop and tell the user to connect it on the Integrations page first.
- **Custom actions:** call the endpoint with `test_api_call`, using realistic example values, to see the actual response. Build the action's inputs and the agent's use of the response from what comes back, not from assumptions.

## Step 2 — Create with the right shape

Pass these as `action_data`. Use exactly these field names — aliases from the REST API or older docs (`phone`, `instructions`, `transfer_mode`, `variables_during_the_call`, `json_body_stringified`, `YES_NO`, `OPEN_QUESTION`, `booking_title`, `timezone`) are ignored or rejected.

**`custom` — HTTP API**

```json
{
  "url": "https://api.example.com/orders/lookup",
  "http_method": "post",
  "description": "Look up an order by its number when the caller asks about an order.",
  "prompt": "Use this when the caller gives an order number and wants its status.",
  "body": { "order_number": "{order_number}" },
  "input_variables": [
    { "name": "order_number", "type": "string", "description": "Order number the caller gives", "example": "ORD-12345" }
  ]
}
```

- `http_method` is lowercase. `body` is an object, with `{placeholders}` matching `input_variables` names.
- `description` and `prompt` decide **when** the LLM calls the action — make them specific.
- Keep descriptions short; some fields are limited to 256 characters.

**`info` — information extractor**

```json
{ "question": { "type": "choice", "identifier": "call_reason", "text": "Why did the caller call?", "choices": ["booking", "billing", "support", "other"] } }
```

- `question.type` is `open`, `yesno`, or `choice` (lowercase).
- `identifier` is snake_case and becomes the key in call results. Choose it carefully — reports filter on it (see `call-report`).
- Optional `question.examples` for `open` and `choice`.

**`transfer`**

```json
{ "phone_number": "+14155550123", "trigger": "The caller asks for a human or the agent can't resolve the issue.", "transfer_target_type": "single_target", "call_transfer_info": { "mode": "warm_transfer_with_context" } }
```

- `phone_number` in E.164. `mode` is `cold_transfer`, `warm_transfer`, or `warm_transfer_with_context`.

**`sms`**

```json
{ "content": "Thanks for calling. Book here: https://example.com/book", "instructions": "Send after the caller agrees to receive the booking link." }
```

**`booking`** — always nested under `config`:

```json
{ "config": { "type": "calcom", "event_type_id": "123456", "booking_email": "bookings@example.com", "lead_timezone": "America/Chicago", "days": 7, "slots_per_day": 3, "min_hours_diff": 1 } }
```

- GoHighLevel: `"type": "ghl"`, with `calendar_id` and `title` required; optional `status` (`confirmed`/`pending`), `location_id`, `assigned_user_id`.
- Scheduling fields for both: `first_appt_date` (YYYY-MM-DD), `days`, `slots_per_day`, `min_hours_diff`, `lead_timezone` (IANA).

**`email`**

```json
{ "recipients": ["ops@example.com"] }
```

For `ivr` and `custom_eval`, check an existing action of that type with `get_action`, or search the docs with `search_docs`, before creating one — don't guess their fields.

## Step 3 — Initialize and test

For **custom** actions, call `initialize_action` after every create or change, with `example_variables` for each input. The action isn't active until this succeeds.

Check the response:

- The status is 2xx and the body contains what the agent needs.
- If it fails, read the error, compare the request with what worked in `test_api_call`, fix with `update_action`, and initialize again. Common causes: wrong method, a body placeholder that doesn't match an input variable name, a missing auth header (have the user set it in the dashboard), or a slow endpoint (consider `failure_timeout` and a `start_message` so the caller isn't left in silence).

`initialize_action` calls the real endpoint. If it writes data, use test values and tell the user.

## Step 4 — Attach

1. `get_agent_actions` for the target agent — note what's attached already, so you don't duplicate an action or remove one by mistake.
2. Tell the user which action you'll attach, to which agent, and when (`before`, `during`, or `after`), and get a yes.
3. `attach_action_to_agent` with `confirmed: true`.
4. `get_agent_actions` again — the action is attached once, with the right timing.

If the agent's prompt must mention the action (when to use it, what to say while it runs), make that change through the `safe-agent-change` skill, and check that attached actions survive the publish.

## Updating an existing action

1. `get_action` first — always. Use its exact field names.
2. Show the user the fields that will change, get a yes, then `update_action` with only those fields in `update_data` and `confirmed: true`.
3. For custom actions, `initialize_action` again.
4. Remember an action may be attached to several agents — the change affects all of them. Say so when it applies.

If `update_action` returns `Could not edit action …`, re-read it with `get_action`, check you're using the stored field names (`http_method`, `body`, `input_variables`, `start_message`, `failure_message`, `failure_timeout`), and retry once with a smaller change.

## Step 5 — Verify on a real call or simulation

After a test call or simulation (see `regression-test`):

- `get_call_details` → `executed_actions`: the action ran, with the inputs you expected.
- For custom actions, `list_webhook_logs` with `webhook_type: "custom_action"` and the `call_id`, then `get_webhook_log_detail` for the request and response.
- For extractors, the value appears under the extractor's `return_value` with your `identifier`.

## Rules

- **Read before you write.** `get_action` before every update; `get_agent_actions` before every attach or detach.
- **Exact field names.** Use the shapes above; don't mix in REST or dashboard names.
- **No secrets in chat or payloads.** Credentials go in the dashboard.
- **Explicit yes before `confirmed: true`.** For attach, detach, and update.
- **Integrations before booking.** Stop if the provider isn't connected.
- **Initialize every custom action change.**
- **Say what you can't do.** If a capability isn't exposed over MCP (MCP-server actions, editing per-agent input mappings without re-attaching, Flow Designer node wiring), say so and point to the dashboard.
