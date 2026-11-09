---
name: call-report
description: Build a call-volume and outcome report for a Synthflow agent, workspace, or agency — counts, durations, statuses, end-call reasons, call reasons from information extractors, transfer and booking results, and trends over a period. Use this whenever the user wants a daily, weekly, or monthly report, a call-reason breakdown, KPIs for a customer, an operations dashboard dataset, or numbers like "how many calls did we get", "what are people calling about", or "what share of calls were transferred". For a quality audit of individual calls, use the call-review skill instead.
---

# Call Report

## Purpose

Turn a period of calls into accurate numbers and a short narrative: volume, outcomes, durations, why people called, and what happened as a result. The output is a report a customer or manager can read, with every number traceable to how it was counted.

`call-review` answers "what's going wrong on these calls?". This skill answers "what happened across these calls?". Use both together when a report should include quality findings.

This playbook runs on the Synthflow MCP tools, whether they come from the plugin's bundled server or the official Synthflow connector.

## Tools this uses

- **`get_call_analytics`** — counts by status, duration metrics, sentiment, end-call reasons, and a daily breakdown. Window up to **120 days**. Cheapest source; start here.
- **`list_calls`** — individual calls with filters: `model_id`, `call_status`, `type_of_call`, `from_date`/`to_date`, `duration_min`/`duration_max`, `end_call_reason`, `labels`, `call_mode` (`production`/`test`), `exclude_simulations` (default true). `page_size` max 50.
- **`get_call_details_batch`** — transcripts, timelines, and executed actions for **up to 25 calls per request**.
- **`list_calls_by_extractor_result`** — calls whose information-extractor result matches a value.
- **`get_agent_actions`**, **`get_call_details`** — find the exact extractor names and fields before filtering on them.
- **`list_agents`**, **`list_subaccounts`** — scope.

## Step 0 — Pin the scope

Confirm, without re-asking what the user already said:

1. **Scope.** One agent (`model_id`), one workspace, or an agency. For an agency-wide report, pass `workspace_ids: ["*"]` to `get_call_analytics` and `list_calls` instead of looping per subaccount.
2. **Period and timezone.** Exact `from_date` and `to_date` in ISO-8601. "Last month" means the full calendar month. Ask which timezone the customer reports in if days or months must line up with theirs; otherwise state that you used UTC.
3. **Call set.** Default to **production calls only, simulations excluded** (`call_mode: "production"`, `exclude_simulations: true`). Say so in the report. Include test calls only if asked.
4. **Questions the report must answer.** Volume and outcomes are the default. Ask whether they also want call reasons, transfer results, booking results, or a comparison with the previous period.

## Step 1 — Headline numbers from analytics

Call `get_call_analytics` for the period (split into ≤120-day windows if needed). Use it for totals, status breakdown, duration stats, end-call reasons, sentiment, and the daily series.

With `workspace_ids: ["*"]`, results come back **per workspace, not merged**. Sum counts yourself. Don't add up averages or percentages across workspaces — recompute them from the counts, or report them per workspace.

If the user only needs headline numbers, stop here and report. Don't fetch individual calls you don't need.

## Step 2 — Breakdowns that need individual calls

Only when the question needs per-call data:

**Call reasons and other extractor values.** Information extractors store a value per call. To count calls by that value:

1. Look up the exact extractor `name` and the `question.identifier` under its `return_value` — from `get_agent_actions`, or from one sample call's `executed_actions` via `get_call_details`. Don't guess them; they're exact keys, not display labels.
2. Call `list_calls_by_extractor_result` once per value (for example each choice of a single-choice extractor), paging through all results. The value's type must match what's stored (boolean, string, or number).
3. Calls with no value for the extractor are their own bucket ("not captured"). Report it — a large one usually means the extractor or the prompt isn't working.

**Transfers, bookings, and other action outcomes.** These live in each call's executed actions:

1. Page through `list_calls` for the period (`page_size: 50`) to collect call ids. **Keep going until a page returns fewer results than the page size.** With `workspace_ids: ["*"]`, paging is per workspace — keep paging until every workspace is exhausted.
2. Fetch details with `get_call_details_batch`, 25 ids at a time. Check `failed_call_ids` and `call_ids_truncated` on every response and retry or report the missing ones — a short result isn't a complete one.
3. Count from `executed_actions`: action triggered, succeeded, failed.

For large periods (more than a few hundred calls), tell the user how many calls you'll fetch before you start. Offer a sample instead of the full set if it's in the thousands, and label sampled numbers as estimates.

## Step 3 — Check the numbers

Before reporting:

- **De-duplicate by `call_id`**, especially when pages or workspaces overlap.
- **Reconcile totals.** The count of calls you fetched should match the analytics total for the same filters. If it doesn't, say by how much and why (simulations, test calls, date boundaries, failed fetches).
- **Separate non-conversations.** Report no-answer, busy, voicemail, and very short calls (under about 5 seconds) separately, so they don't skew outcome rates.
- **Every percentage has its denominator.** "42% transferred (126 of 300 connected calls)".

## Step 4 — Report

Lead with a short summary, then tables:

1. **Summary:** scope, period and timezone, call set (production only, simulations excluded), total calls, connected calls, and the 2–3 most important findings.
2. **Volume and outcomes:** by status, with counts and percentages. Add the previous period alongside when a comparison was requested.
3. **Duration:** average and median for connected calls; flag a long tail if there is one.
4. **Call reasons** (if requested): one row per extractor value, including "not captured".
5. **Action outcomes** (if requested): transfers and bookings — attempted, succeeded, failed.
6. **Daily or weekly trend:** a compact table, or offer a chart.
7. **Method notes:** filters used, anything excluded, any sampled or estimated numbers, any calls that failed to fetch.

Offer a CSV-style table if the user wants to paste the data elsewhere.

## Rules

- **Don't invent metrics.** If no tool exposes a number (for example cost or latency), say so instead of estimating.
- **Analytics first.** Only fetch individual calls for the breakdowns that need them.
- **Complete or labelled.** Page to the end, or label the number as a sample.
- **Show your denominators.** No percentage without its count.
- **Read-only.** This skill never changes agents or actions. If the numbers point at a problem, suggest `call-review` to investigate specific calls, or `safe-agent-change` to fix the agent.
