---
name: model-router
description: Route every piece of AI work to the cheapest model and lowest effort that can do it well, sized to how urgent the job is. The running model settles urgency first, triages the job, writes tight specs, then dispatches worker subagents with an explicit model (and effort) on every call, and escalates only on defined signals. Use before dispatching any subagent or workflow in a multi-step task, and whenever the user says "save tokens", "use a cheaper model", "worker agents", "minimum effective dose", "route this", "no rush", "background job", "overnight", "budget", "usage limits" or mentions token cost. Also use when running on a cheap model and hitting repeated failures, ambiguity or an architectural choice; it defines when and how to escalate. Works for Claude (Claude Code, Cowork, the API) and maps to ChatGPT/Codex, Gemini and Grok tiers.
---

# model-router

Smartest brain for the thinking, cheapest capable brain for the doing, and no faster or pricier than the job's urgency needs.

The most expensive mistake in a top-model session is dispatching a subagent **without choosing its model**: it silently inherits the session's expensive model and you pay frontier rates for mechanical work. The second is running a no-rush job at urgent speed and burning through your plan's limits. Decide urgency, model and effort before every dispatch.

## Step 0: urgency gate (before any dispatch)

| | **Budget mode** (default) | **Priority mode** |
|---|---|---|
| When | Anything without a deadline: clean-ups, migrations, tagging, research backlogs, "no rush", "overnight", "background" | It ships today, a customer is waiting, or the user says "urgent", "ASAP", "quality over cost" |
| Spend | Stay inside the plan's included usage. Never go into paid overage without asking. | Normal routing rubric |
| Deterministic work | Script it: zero model tokens | Same |
| Per-item judgement at volume (captioning, tagging, classifying, extracting) | Fast tier at **low effort**. Pilot one batch first; escalate only the items that fail checks | Workhorse tier |
| Verification | Mechanical checks, then a 5–10% sample on a cheap model. Never the top model over every batch | Sampled review on a stronger model |
| Top model | One diagnosis or decision agent at most, never workers | As the rubric says |
| Concurrency | ≤ 3 agents at once, paced. It's fine if it runs all night | Parallel |

How to decide:
1. The user's words decide it. Anything admin- or storage-shaped with no deadline is budget mode.
2. If it isn't clear, ask once before dispatching: *"Is this urgent (today or customer-facing), or a background job I should run cheaply within your plan, even if it takes longer?"* Don't infer urgency from how big or important the job sounds.
3. Before launching more than ~10 agents or a long workflow, state the rough cost first, and check usage headroom if a usage tool exists.
4. If the user says **pause**, stop every background agent immediately and note where each one stopped.

## Two dials, not one: model and effort

Current models have an **effort** setting (`low`, `medium`, `high`, `xhigh`, `max`) that controls how long they think and how many tool calls they make. It is the cheapest lever after the free ones (scripts, caching), and it often beats switching models:

- **Workers on mechanical tasks run at `low`.** Fewer, more consolidated tool calls and terser output.
- **`high` is the sweet spot for most real work.** `xhigh` for hard coding and long agentic runs. `max` only when measurement shows the level below falls short.
- **Before splitting a job across models, try one strong model at lower effort.** On the newest models, lower effort often matches the previous generation at high effort. One model also means one prompt cache (caches are per-model), and no hand-off overhead.
- **Judge cost per finished task, not per request.** A cheap call that needs three retries isn't cheap.
- Some models default to `medium` (for example Claude Opus 5.5 and Haiku 5.5), others to `high`. Set it explicitly when it matters.

## Cost ladder (Claude API, per 1M tokens, checked October 2026)

Prices are written as "USD n" deliberately: a dollar sign followed by a number can be swallowed when a skill is invoked with arguments.

| Tier | Model | Input / output | Relative cost |
|---|---|---|---|
| Fast | Haiku 5.5 | USD 0.10 / 0.50 (prompts up to 100K; 0.50 / 2.50 above) | 1× |
| Workhorse | Sonnet 5.5 | USD 2 / 10 | 20× |
| Strong | Opus 5.5 | USD 4 / 20 | 40× |
| Frontier | Fable 5.1 | USD 10 / 50 | 100× |

The gaps are now huge: one frontier-priced batch job costs about 100× the same job on the fast tier. Two more free levers: **prompt caching** (cache reads cost a small fraction of normal input; keep stable content first and don't edit earlier turns) and the **Batch API** (50% off for anything that can wait hours). Prices move, the ratios are the point; check the provider's pricing page when it matters.

## Routing rubric: task shape → tier

For **priority mode**. In budget mode, drop every row one tier (fast tier is the floor), except the brand-copy rule below.

| Task shape | Tier | Examples |
|---|---|---|
| Deterministic or programmatic | **No model: a script** | Inventories, size or hash matching, dedupe, renames, moves, format conversion, counts |
| Mechanical, fully specified | Fast, low effort | Header normalisation, link checks, running commands and reporting output, extraction with a known structure, per-item captioning, tagging or classifying at volume |
| Standard implementation from a clear spec | Workhorse | Code to spec, tests, bulk edits, summaries from a detailed brief, scoped research sweeps |
| Judgement inside a bounded problem | Strong | Debugging with an unknown cause, multi-file changes that need design sense, nuanced customer-facing copy, sampled review of another agent's important output |
| Architecture, ambiguity, decomposition, high stakes | Frontier (or the session model) | Deciding what to build, contradictory requirements, anything irreversible, legal or money-touching |

Tiebreaker: if a wrong result would be caught cheaply (a grep sweep, a failing test, an ID check), route down a tier. If a wrong result ships to a customer or production, route up, but only for the items that ship.

Brand-copy rule: public, brand-facing writing gets the workhorse tier as a minimum, even in budget mode. Fast-tier copy costs more in rework than it saves. Internal metadata (file maps, photo captions) is fine on the fast tier.

## Dispatch syntax

**Claude Code / Cowork.** The model override is the `model` parameter on the Agent tool, not `subagent_type` (which picks the agent's toolkit). They compose:

```
Agent({
  description: "Normalise CSV headers, batch 2",
  subagent_type: "general-purpose",
  model: "haiku",          // haiku | sonnet | opus | fable
  run_in_background: true,
  prompt: "..."            // self-contained, see Worker prompt rules
})
```

Where the harness exposes an `effort` option on agents or workflow steps, set `low` for mechanical workers. In a Workflow script: `agent(prompt, {model: 'haiku', effort: 'low'})`. In budget mode, cap concurrency yourself: process items in chunks of three, not one pipeline over a hundred items. If a harness has no `model` parameter, define custom agents in `~/.claude/agents/` (for example `worker-haiku.md` with `model: haiku` in the frontmatter) and dispatch those.

**Claude API.** Set `model` per request and `output_config: {effort: "low"}` for worker calls. Non-urgent fan-out goes through the Batch API. For "cheap executor, smart reviewer" on a single request, use the advisor tool (a workhorse executor with a stronger advisor).

**Other providers.** The same pattern works anywhere a request names its model and reasoning level:

| Tier | Claude | ChatGPT / Codex | Gemini | Grok |
|---|---|---|---|---|
| Frontier | Fable | the top reasoning model | the top Pro model, high thinking | the top reasoning model |
| Strong / workhorse | Opus, Sonnet | the standard flagship | Pro | the standard model |
| Fast | Haiku | the mini or nano tier | Flash or Flash-Lite | the fast tier |

Use each provider's reasoning-effort or thinking setting as the second dial. Codex and similar coding agents let you pick both the model and the reasoning effort per task; route the same way.

## The protocol (routing down: you're on the top model)

1. **Urgency gate:** budget or priority mode. Ask if unclear.
2. **Triage in one pass.** Split the request into scriptable work (do it with code), thinking work (decisions, specs, anything ambiguous) and doing work (everything that can run from a spec). Write a spec, not an essay.
3. **Keep thinking, ship specs.** The expensive model writes work orders: exact file paths, requirements, output format, acceptance criteria. It doesn't do the batch work itself.
4. **Pilot, then dispatch.** Run one batch, check it mechanically, then dispatch the rest with an explicit model (and effort) on every call. Priority mode: all independent dispatches at once. Budget mode: ≤ 3 at a time.
5. **Verify cheaply.** Mechanical checks first (grep sweeps, exit codes, counts, ID alignment), then a sample: 10–15% on a stronger model in priority mode, 5–10% on a cheap one in budget mode. Re-reading every output on the top model wipes out the saving.
6. **Finish at the top.** Final synthesis, voice and the message to the user stay with the session model.

## Escalation triggers (routing up: you're on a cheap model)

A cheap model can't reliably judge its own limits, so escalation runs on mechanical signals. Escalate when any of these fire:

- **Two failed attempts** at the same subtask. Different failure modes mean a structural misunderstanding; stop retrying.
- The spec needs a **choice the instructions don't settle**: naming something public, picking an architecture, resolving contradictory requirements.
- The task touches **money, legal, deletion, publishing or brand-facing output** with no explicit approval on record.
- The user's words include **design, architect, strategy, why, rethink**.

Escalating means spawning one stronger agent whose only job is the diagnosis or decision. It returns a spec, not the finished task. In budget mode, escalate only the failing items. Then carry on cheaply against that spec. Don't move the whole session up a tier.

## Worker prompt rules

Workers start with **zero conversation context**. Every prompt must stand alone:

- Exact file paths and item lists, never "find the drafts".
- Paste in the constraints that matter (spelling standard, protected names, "don't touch code blocks or URLs"). Workers don't see your project instructions.
- Demand a **structured, terse return**, such as a per-item status table. Worker output becomes input for the expensive model, so verbosity is paid for twice. For big jobs, workers write results to files and return counts.
- Prefer deterministic tools inside the worker. A word-list substitution beats model judgement for spelling; `curl -w '%{http_code}'` beats "check the link looks fine".

## When NOT to delegate

- One synthesis deliverable the top model would rewrite anyway (one strategy doc, one email).
- Anything a couple of shell commands can do. Run it inline.
- Small jobs: under ~10 items or a few minutes of work. Every worker carries its own system prompt and context, often tens of thousands of tokens, so below that size the overhead beats the saving.
- Final assembly of brand-facing output. Workers draft, the session model finishes.

## Surfaces without subagents

In a plain chat (claude.ai, the ChatGPT app), there are no worker dispatches. Apply minimum effective dose in one brain: triage first, keep deliberation short, don't re-derive settled decisions, and pick the lighter model or lower thinking setting for simple questions. For genuine fan-out work, move it to Claude Code, Cowork or an API pipeline.

## Turning it off

If the user says "router off", "stay on the top model", "full fat" or "quality over cost, don't route", stop down-routing for the rest of the session and honour it without argument. Budget mode's "don't go into paid overage" still applies unless they say otherwise.

## Red flags

| Smell | Fix |
|---|---|
| Big fan-out with no urgency on record | Ask "urgent or background?" first. Default to budget mode. |
| Agent call with no `model` in a top-model session | The worker inherits frontier pricing. Add the parameter. |
| Workhorse tier tagging thousands of items on a no-rush job | Fast tier at low effort. Pilot, then escalate only the failures. |
| A strong model verifying every batch | Mechanical checks, then a 5–10% sample on a cheap model. |
| 10+ agents in parallel on a background job | ≤ 3 at a time, paced. |
| A model doing what a script can do | Write the script. |
| `subagent_type: "opus"` | That isn't a model override. Use `model: "opus"`. |
| Splitting a job across three models before trying one model at lower effort | Measure the simpler option first. |
| "I'll just do the batch work myself, it's faster" | You're the 100× brain doing 1× work. Spec it and dispatch it. |
| Third retry of a failing fix on a cheap model | Two strikes: escalate to one stronger diagnosis agent. |
