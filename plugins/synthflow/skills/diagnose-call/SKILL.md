---
name: diagnose-call
description: Investigate why a specific Synthflow call (or a small set of similar calls) went wrong — failed or missed transfers, calls cut off, the agent going silent or repeating itself, actions or webhooks that didn't fire or failed, wrong data captured, or outbound calls that didn't connect. Use this whenever the user points at a call or a phone number and asks what happened or why, e.g. "why did this call drop", "the transfer didn't ring", "the booking didn't happen on this call", "the webhook never arrived", or "check the call from +1… this morning". For an audit of many recent calls, use the call-review skill instead.
---

# Diagnose Call

## Purpose

Find out what actually happened on a call, and why, from the evidence Synthflow exposes: the call record, transcript, timeline, executed actions, end-call reason, and webhook logs. Then say clearly what's confirmed, what's likely, and what can't be determined from these tools.

This playbook runs on the Synthflow MCP tools, whether they come from the plugin's bundled server or the official Synthflow connector. It is read-only.

## Tools this uses

- **`list_calls`** — find the call: by `lead_phone_number` (partial match), `model_id`, date range, `call_status`, `end_call_reason`, `duration_min`/`duration_max`.
- **`get_call_details`** — transcript, timeline, executed actions, end-call reason, labels, recording URL.
- **`get_call_details_batch`** — up to 25 calls at once, for comparing similar calls.
- **`list_webhook_logs`** — action and webhook executions, filterable by `call_id`, `model_id`, `status`, `webhook_type` (`custom_action`, `external_webhook`, `inbound_webhook`), `time_range` (up to `7d`).
- **`get_webhook_log_detail`** — full request, response, status code, duration, and error for one execution.
- **`get_agent`**, **`get_agent_actions`**, **`get_action`** — the configuration the call ran with.
- **`list_agent_versions`** — whether the agent changed around the time of the call.
- **`search_docs`** — how a feature is supposed to behave.

## What these tools can't show

Be upfront about this — it decides what you can conclude:

- **Audio:** recordings can't be played over MCP. Latency, interruptions, silence, and mispronunciation can't be confirmed from a transcript. Share the recording URL and ask the user to listen at the timestamp.
- **Telephony internals:** there's no SIP signalling, carrier error codes, RTP or media traces, or trunk registration status. When the evidence points at the phone network, say so and recommend opening a support ticket with the call id.
- **Flow Designer internals:** there's no per-node execution trace. You can see what the agent said and which actions ran, not which node it was in.
- **Webhook logs** only go back 7 days through `list_webhook_logs`.

## Step 1 — Find the call

Use whatever the user gave: call id, phone number, agent, time. With a phone number and an approximate time, `list_calls` with `lead_phone_number` and a narrow date range. If several calls match, list them (time, duration, status, agent) and confirm which one. Don't guess.

If the issue repeats ("all transfers failed today"), pick 3–5 representative calls and fetch them with `get_call_details_batch`.

## Step 2 — Read the call record

From `get_call_details`, note:

- Status, direction, duration, start time, agent, and **end-call reason**.
- The **timeline**: when the agent spoke, when actions ran, when the call ended, and who ended it.
- **Executed actions:** which ran, with what inputs, and what they returned or which error.
- The **transcript** around the moment it went wrong.

Write down a one-line timeline of what happened before forming a theory.

## Step 3 — Follow the symptom

**Transfer didn't connect or didn't ring**
1. Did the transfer action run? Check `executed_actions`. If it never ran, the cause is the trigger: compare the transcript with the action's `trigger` (`get_action`) and the prompt's transfer rules.
2. If it ran: check the destination number (E.164, correct number), the mode (cold or warm), and what the transcript shows after the transfer started — hold message, failure message, or silence.
3. Compare with other transfers to the same number (`list_calls` + batch). If all fail, suspect the destination or the carrier; if only some fail, look at the time of day, busy or no-answer, and the timeout.
4. Beyond this, it's telephony. Say what the record shows and recommend a support ticket with the call ids.

**Call cut off or ended early**
1. End-call reason and who hung up.
2. Duration: several calls ending at the **same duration** points to a limit — check the agent's `max_duration_seconds` / `max_duration_is_enabled` and idle settings (`allowed_idle_time_seconds`, idle reminders) in `get_agent`.
3. The last turns of the transcript: did the agent say goodbye (an end-call rule fired), or did it stop mid-sentence?
4. Did an action run just before the end and fail or time out?

**Agent went silent, repeated itself, or said something wrong**
1. Find the turn in the transcript.
2. Was an action running at that moment (silence while waiting on a slow API)? Check its duration in the webhook log, and whether it has a `start_message`.
3. Compare what the agent said with the prompt (`get_agent`) — a missing or conflicting instruction is the usual cause. Offer the `prompt-review` skill.
4. Silence and timing need the recording to confirm.

**Action or webhook didn't fire, or failed**
1. `executed_actions` on the call: did it run at all?
   - **Didn't run:** check it's attached to the agent with the expected timing (`get_agent_actions`), and whether the conversation met its trigger or description.
   - **Ran and failed:** `list_webhook_logs` with the `call_id`, then `get_webhook_log_detail`. Read the request (URL, method, body — were the variables filled?) and the response (status code, body, duration).
2. Common causes: a 4xx from the user's API (auth, wrong field), a timeout, an input variable the agent never collected, or an action changed recently without being re-initialized.
3. Post-call webhooks: `webhook_type: "external_webhook"`. Check the URL on the agent and the response the receiver gave.

**Wrong or missing extracted data**
1. The extractor's `return_value` on the call and the transcript: was the information actually said?
2. The extractor's question text and choices (`get_action`): ambiguous wording or missing choices are the usual cause.

**Outbound call didn't connect**
1. `call_status` (`failed`, `busy`, `no_answer`, `hangup_on_voicemail`) and duration.
2. Several failures from the same number around the same time suggests a number, trunk, or concurrency problem — that's telephony; recommend a support ticket with the call ids.

## Step 4 — Check for a recent change

`list_agent_versions` for the agent: was a new version published shortly before the problem started? If so, offer to compare it with the previous version (`get_agent_version_diff`) — a recent change is often the cause.

## Step 5 — Report

- **Call:** id, time, agent, direction, duration, status, end-call reason.
- **What happened:** a short timeline with timestamps.
- **Cause:** split into
  - **Confirmed** — shown directly by the record, transcript, or logs (quote or cite it);
  - **Likely** — consistent with the evidence but not proven;
  - **Can't be determined here** — needs the recording, telephony logs, or Synthflow support.
- **Fix:** the specific change, if there is one (prompt, action config, agent setting, destination number), and which skill to use for it (`safe-agent-change`, `build-action`). If it's a platform or telephony issue, what to include in the support ticket: call ids, times, numbers, and what you ruled out.
- **How to confirm the fix:** a test call or `regression-test` scenario that reproduces the problem.

## Rules

- **Evidence first.** Build the timeline from the record before forming a theory.
- **Label certainty.** Never state a "likely" cause as confirmed.
- **Don't speculate past the tools.** Say plainly when the answer is in telephony or audio data you can't see.
- **Read-only.** Don't change agents or actions while diagnosing. Offer the fix as a next step.
- **Protect caller data.** Quote only the transcript lines needed as evidence.
