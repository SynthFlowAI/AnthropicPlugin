---
name: regression-test
description: Build and run simulated-caller tests against a Synthflow voice agent, then report which scenarios pass or fail with transcript evidence. Use this whenever the user wants to test an agent before or after a change, check that a prompt fix works, reproduce a failure from a real call, build a test suite, or re-run existing simulations. Trigger on phrases like "test the agent", "run the simulations", "regression test", "make sure this didn't break anything", "build a test suite", "reproduce this call", or "re-run the tests".
---

# Regression Test

## Purpose

Catch behaviour regressions with simulated callers before real customers hit them. A simulation is an AI caller with a persona prompt who talks to the agent for up to N turns; each case passes or fails against its success criteria.

The loop:

**plan cases → build or reuse a suite → run → collect results → report**

Use it before publishing a non-trivial change (see the `safe-agent-change` skill), after publishing to confirm a fix, or to reproduce a failure found by the `call-review` skill.

This playbook runs on the Synthflow MCP tools, whether they come from the plugin's bundled server or the official Synthflow connector. Simulations place no phone calls.

## Tools this uses

- **`list_agents`**, **`get_agent`**, **`get_agent_actions`** — the agent under test, its prompt, and what it can do.
- **`list_simulation_suites`**, **`list_simulation_cases`** — reuse existing tests before writing new ones.
- **`create_simulation_suite`** — `name`, `model_id`, `language` (default `en-US`).
- **`create_simulation_case`** — `name`, `prompt` (the caller persona), `success_criteria` (list), `call_success_type` (`all` or `any`), `suite_id`.
- **`execute_simulation_suite`** — `suite_id`, `target_agent_id`, `max_turns` (10–50, default 20). Returns a session id.
- **`get_simulation_session`** — status, transcripts, and per-criterion results for that run.
- **`list_simulations`** — past runs, filterable by agent, session, and status.
- **`get_call_details`** — when turning a real failed call into a test case.

## Step 0 — Pin the target

1. **Agent and workspace.** Resolve the exact `model_id` (and subaccount `workspace_id` if needed, via `list_subaccounts`). Never guess IDs.
2. **What version is being tested.** Simulations run against the agent's configuration as Synthflow serves it. Don't assume a run picks up an unpublished draft. To test a change before it reaches production, apply it to a **test copy** of the agent (a separate agent with the same prompt and actions) and run the suite against that copy.
3. **Side effects.** Simulated conversations can trigger the agent's attached actions — bookings, CRM writes, SMS, webhooks. Check `get_agent_actions`. If the agent has actions that write to real systems, tell the user before running, and recommend a test copy with those actions detached or pointed at a sandbox.

## Step 1 — Plan the cases

Reuse first: `list_simulation_suites` for this agent and `list_simulation_cases` for the suite. If a suite already covers the behaviour, re-run it instead of creating duplicates.

When new cases are needed, aim for a small, focused set — **5 to 10 cases** for one change is usually enough. Cover:

- **The change itself.** One case that directly exercises the new or fixed behaviour.
- **The happy path.** The agent's main job done cleanly (book, qualify, route, answer).
- **Neighbouring behaviour.** Flows the change could plausibly break — anything that shares a prompt section, action, or variable with what changed.
- **Edge callers.** Interruptions, corrections ("no, it's 4-5-1 not 4-1-5"), spelled-out names, numbers and addresses, off-topic questions, a caller who wants a human.
- **Real failures.** For each problem call the user cares about, pull it with `get_call_details` and write a persona that reproduces the caller's behaviour from the transcript.

Show the user the planned cases (name plus a one-line description each) before creating more than a handful.

## Step 2 — Write each case well

**Persona prompt** (`prompt`): describe who the caller is and what they do, in the second person, concretely:

- Their goal, and the details they'll give when asked (name, date, order number…).
- How they behave: patient or rushed, gives everything at once or only when asked, corrects themselves, interrupts.
- When they stop: "End the call once the booking is confirmed or after you've been offered a transfer."
- Use the agent's language. Create the suite with the matching `language` (for example `de-DE`, `es-ES`).

**Success criteria** (`success_criteria`): each one a single, checkable statement about the **agent's** behaviour:

- Good: "The agent reads the phone number back before booking." · "The agent offers a transfer when the caller asks for a human." · "The agent does not state a price."
- Bad: "The call goes well." · "The agent is helpful and polite."

Use `call_success_type: all` unless the criteria are genuine alternatives.

Don't include `base_agent_id` when creating custom cases.

## Step 3 — Run

Call `execute_simulation_suite` with the `suite_id` and the `target_agent_id`. Keep `max_turns` at 20 unless the scenario needs a long conversation (multi-step booking, long intake), then raise it to 30–40.

**If the call errors or times out,** the run may still have started. Check `list_simulations` filtered by `target_agent_id` before running again, so you don't start a duplicate batch.

## Step 4 — Collect results

Call `get_simulation_session` with the returned session id.

Simulations take a while to finish. **Don't poll in a tight loop:** check once, tell the user how many are complete, and check again after a pause (30–60 seconds). After about 10 checks, stop and report what has finished so far, with the session id so the user can check later.

For each simulation, read:

- Pass or fail, and which criteria failed.
- The transcript around the failure — the turn where the agent went wrong.

A failing case isn't always the agent's fault. Before calling it a regression, check whether:

- the persona didn't do what the criteria assume (for example, never asked for a human);
- the criterion is ambiguous or judged something the transcript can't show;
- the run hit `max_turns` before the scenario could finish.

Report those as **test problems**, separately from agent failures, and offer to fix the case.

## Step 5 — Report

Lead with a table, one row per case:

| Case | Result | Failed criteria | Evidence |
|---|---|---|---|

- **Evidence:** a short transcript quote from the turn that failed — not a paraphrase.

Then a short summary:

- Agent and version tested, and the suite name.
- Passed N of M. Agent failures and test problems counted separately.
- For each agent failure: what went wrong and the likely cause (prompt section, action, setting).
- When comparing against an earlier run (`list_simulations`), call out anything that **used to pass and now fails** first — that's the regression.
- One or two concrete next steps.

## Rules

- **Reuse before creating.** Don't duplicate suites or cases that already exist for the agent.
- **No real-world side effects without the user knowing.** Check attached actions before running.
- **Test what will ship.** If the user wants to test an unpublished change, use a test copy of the agent; don't assume the draft is what runs.
- **Evidence over assertion.** Every reported failure cites the transcript.
- **Separate agent failures from bad tests.** Don't "fix" the agent to satisfy a broken test case.
- **Don't auto-fix.** After the report, offer to draft the fix (with `prompt-review` and `safe-agent-change`), then re-run the same suite to confirm it.
