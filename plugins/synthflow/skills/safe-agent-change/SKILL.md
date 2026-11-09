---
name: safe-agent-change
description: Make a change to a live Synthflow voice agent safely — record a baseline, save a draft, review the diff, publish only on approval, verify the published state, and roll back if needed. Use this whenever the user wants to edit, update, fix, tweak, or publish an agent's prompt, greeting, voice, transcriber, LLM, actions, or any other setting, or wants to roll an agent back. Trigger on phrases like "update the prompt", "change the voice", "fix the agent", "publish the draft", "push this live", "roll back the agent", or "undo the last change".
---

# Safe Agent Change

## Purpose

Change a live agent without breaking what already works. Every edit goes through the same loop:

**baseline → draft → diff → approval → publish → verify** (→ rollback if verify fails)

Agents that skip steps in this loop are where most production incidents start: attached actions silently lost on republish, a prompt from one agent written over another, a failed update retried on top of a draft that had in fact been saved, or a draft published that changed more than the user asked for.

This playbook runs on the Synthflow MCP tools, whether they come from the plugin's bundled server or the official Synthflow connector.

## Tools this uses

- **`get_current_workspace`**, **`list_subaccounts`** — confirm which workspace the agent lives in.
- **`list_agents`**, **`search_agents_by_prompt`** — find the agent when the user hasn't given a `model_id`.
- **`get_agent`** — current settings and prompt.
- **`get_agent_actions`** — the actions attached to the agent (the baseline you must preserve).
- **`list_agent_versions`** — published versions and any existing draft.
- **`update_agent`** — saves changes as a **draft**. Nothing goes live.
- **`get_agent_version_diff`** — what the draft changes compared to live.
- **`publish_agent_draft`** — makes the draft live.
- **`rollback_agent_version`** — restores a previously published version.
- **`list_voices`** — required before any voice change.

Some tool descriptions mention `show_version_diff`. That UI tool doesn't exist over MCP — show the diff to the user yourself, in the chat.

## Step 0 — Pin the target

1. **Workspace.** If the agent may live in a subaccount, call `list_subaccounts` and pass that subaccount's `workspace_id` on every call below. Never guess or reuse a `workspace_id` from elsewhere — tools reject any workspace that isn't the current one or one of its subaccounts.
2. **Agent.** Resolve the exact `model_id`. If the user named an agent, use `list_agents` (or `search_agents_by_prompt`) and confirm the match when more than one agent could fit. Never guess.
3. **Environment.** If the user keeps separate dev/test and production agents, confirm which one this change is for. Default to the test agent when one exists and the user hasn't said otherwise.

## Step 1 — Record a baseline

Before changing anything, capture and keep in the conversation:

- `get_agent` — the current prompt and the settings you're about to touch.
- `get_agent_actions` — every attached action: name, `action_id`, trigger condition, and input-variable mapping.
- `list_agent_versions` — the version marked `is_current=true` (your rollback target) and whether a **draft already exists**.

**If a draft already exists,** stop and tell the user. Run `get_agent_version_diff` to show what it contains. Someone may have unpublished work in progress — your update would land on top of it, and publishing would ship their changes too. Ask whether to build on it or start from live; don't publish someone else's draft without them saying so.

## Step 2 — Save the draft

Call `update_agent` with **only the fields that change**. Every field in `update_data` is optional; anything you leave out is left alone.

Rules for the payload:

- **Prompt edits:** start from the prompt text returned by `get_agent` for **this same `model_id`** in Step 1, apply the edit, and send the whole result. Never send a prompt taken from another agent, a local file that might be stale, or an earlier read from before someone else changed it.
- **Voice changes:** call `list_voices` first, then send all six voice fields together: `voice_id`, `voice_name`, `voice_accent`, `voice_language_code`, `voice_preview_url`, `voice_is_recommended`. Sending only `voice_id` leaves the agent's voice settings inconsistent.
- **Don't touch `action_bindings`** unless the user asked to change which actions are attached. If you must, send the complete list built from your Step 1 baseline, with every existing trigger condition and input-variable mapping preserved — a partial list replaces the agent's actions with it.
- **Known publish-time constraints**, so the draft doesn't fail later:
  - `synthesizer_fallback_provider` must differ from `voice_provider`.
  - `transcriber_endpointing` must be `100` or `300`.

**If `update_agent` returns an error, do not retry blindly.** Some errors are returned after the draft was in fact saved. Run `get_agent_version_diff` first:

- The diff shows your change → the draft was saved. Continue to Step 3.
- The diff is empty or unchanged → the update didn't apply. Read the error, fix the payload (usually a validation problem with one field), and try once more with a smaller change. If it fails again, stop and report the exact error to the user.

## Step 3 — Review the diff

Call `get_agent_version_diff` with no `version_id` (it defaults to the draft compared with live). Show the user every changed field.

Check before asking for approval:

- **Only the intended fields changed.** Flag anything else — including fields the platform normalised (for example a `null` becoming a default value) — and say whether it matters.
- **Attached actions are intact.** If the diff shows actions removed or `attached_actions` empty while the live agent has actions, stop. Don't publish. Tell the user, and fix the draft (re-send `action_bindings` from the baseline) before going on.
- **Prompt changes match the request.** For prompt edits, quote the changed lines, not the whole prompt.

For a non-trivial prompt change, offer to run the `prompt-review` skill on the new prompt, or the `regression-test` skill against a test copy of the agent, before publishing.

## Step 4 — Publish on explicit approval

Call `publish_agent_draft` **only after the user has seen the diff and said yes** in this conversation. A request to "make the change" is not approval to publish — the draft is the change; publishing is a separate decision.

Pass a short `description` saying what changed and why (for example "Ask for confirmation before booking; fixes double bookings"). It shows up in the version history.

If publishing fails, report the exact error. Common causes are the constraints listed in Step 2.

## Step 5 — Verify the published state

Publishing succeeding isn't the same as the agent being right. Verify:

1. `list_agent_versions` — a new version is `is_current=true` and no draft remains.
2. `get_agent` — the changed fields hold the new values.
3. `get_agent_actions` — compare with the Step 1 baseline. **Every action that was attached before is still attached**, with the same trigger condition and input-variable mapping.

If any action went missing, tell the user straight away, and offer either to re-attach it (`attach_action_to_agent` with the baseline mapping) or to roll back.

Suggest a test call or simulation to confirm behaviour, especially for prompt and voice changes.

## Rolling back

Use this when verification fails, or when the user asks to undo a change. Always in this order:

1. `list_agent_versions` — call it **fresh**, even if you listed versions earlier. Copy `version_id` values verbatim from this output.
2. `get_agent_version_diff` with `version_id` = the target version and `compare_with` = the row with `is_current=true`. Show the user what will change.
3. `rollback_agent_version` with the target `version_id`, **only after the user confirms.**

Rollback creates a new version, doesn't delete history, and **clears any existing draft** — warn the user if one exists. You can't roll back to the version that's already current.

After rolling back, run Step 5 again.

## Report

End with a short summary:

- Agent and workspace changed.
- What changed (fields, and the key prompt lines).
- Published version number, or "draft saved, not published".
- Verification result: actions intact yes/no, fields confirmed yes/no.
- The version to roll back to if needed.

## Rules

- **Draft first, always.** Never treat `update_agent` as going live, and never publish without showing the diff.
- **Explicit approval to publish.** Approval to edit isn't approval to publish.
- **One agent at a time.** When changing several agents, run the full loop for each one. Don't batch publishes.
- **Preserve what you didn't touch.** Attached actions, voice settings, and anything else outside the request must come through unchanged — check it, don't assume it.
- **Never invent IDs.** `model_id`, `workspace_id`, and `version_id` come from a fresh tool result, never from memory or guesswork.
- **Stop and report rather than improvise.** If a tool can't do what's needed (for example, editing a Flow Designer graph node by node), say so and point the user to the Synthflow dashboard rather than working around it.
