# Stop Renting the Brain and the Muscle from the Same Vendor

## How we cut our agent-coding LLM bill by ~70% without touching quality

Our setup is probably familiar to anyone running coding agents in production: **Opus writes the plan, Sonnet executes it.** It works great. It also costs us roughly **$1,500–$2,000 a month** — stable, every month, like rent.

That number finally annoyed me enough to do something about it. Here's the full analysis, including one idea that turned out to be a dead end.

## The key realization: plan and execute have different economics

The plan step is **low-volume, high-leverage**. A bad plan poisons everything downstream; a good one makes execution almost mechanical. The execute step is **high-volume, low unit value** — lots of tokens, each one individually cheap to get wrong and retry.

So the question was never "which single model is cheapest." It's two separate questions:

1. What's the cheapest planner we can get away with? → **Keep Opus.** Don't touch the brain.
2. What's the cheapest executor that's good enough? → This is where all the money is.

## Dead end #1: a Mac Studio for local DeepSeek R1

My first instinct was the classic one: buy a Mac Studio with 256GB unified memory, run DeepSeek R1 locally, stop paying per token.

It doesn't work. Three facts killed it:

- **DeepSeek R1 (671B) at Q4_K_M is 404GB.** It does not fit in 256GB (roughly 240GB usable after OS overhead). Even Q3 quants (290–320GB) don't fit. You'd be stuck at emergency-only 2-bit quantization, at which point it's no longer "comparable to Sonnet."
- **Apple discontinued the 512GB Mac Studio option** in early 2026 (DRAM shortage), and raised the 96GB→256GB upgrade from $1,600 to **$2,000**. So the machine that *could* run R1 no longer exists new, and the 256GB config is now $5,999 before tax (~$6,570 in California).
- What *does* fit in 256GB: **Qwen3-235B-A22B at Q4_K_M is only 142GB**, and GLM-4.5 (355B MoE) fits at ~200GB. Both are strong — roughly last-generation flagship level — but neither is truly Sonnet-equivalent, especially on long agentic chains.

The Mac Studio isn't a bad machine. It's just not an R1 machine. If you buy it, buy it for Qwen3-235B, not for R1.

## What actually worked: unbundle the stack

We landed on a three-tier architecture. The principle: **rent the brain from the best vendor, buy the muscle from the cheapest one.**

| Tier | Role | Model | Price (per 1M tokens) |
|------|------|-------|----------------------|
| Brain | Planning | Opus (unchanged) | $15 / $75 |
| Muscle | Execution | GLM-4.6 API | $0.60 / $2.20 |
| Pen | Documentation | Qwen3-235B (local) or DeepSeek chat API | $0 marginal / $0.28 / $0.42 |

**Why GLM-4.6 for execution:** it's a 355B MoE explicitly tuned for agentic coding, at roughly a quarter of Sonnet's price ($2/$10). The API is OpenAI-compatible, so switching is mostly a base-URL change. We validated on a slice of traffic first — pass rates held, and the execution bill dropped from ~$1,200/month to ~$270.

**Why not just use DeepSeek for everything:** DeepSeek's API is even cheaper (R1 at $0.55/$2.19, chat at $0.28/$0.42), but for tool-heavy agent execution we found GLM's agent tuning worth the small premium. Your mileage may vary — test both.

**The planner stays on Opus.** Planning tokens are a minority of the bill and the highest-leverage tokens in the pipeline. Saving a few hundred dollars a month here while risking plan quality is a bad trade.

## The docs surprise: it's the input, not the model

We also assumed we needed a good "writing model" for documentation. Turns out that was the wrong diagnosis. Every model writes mediocre docs when it only sees diffs — the output is grammatically fine and informationally empty, because the model can't see *why* decisions were made.

The fix wasn't a better model, it was a better input: our Opus plans already contain design rationale. We added a fixed "design decisions" section to the plan template (what was chosen, what was rejected, why), and feed plan + diff together into the docs step. With that input, even the cheapest tier writes acceptable docs. The bottleneck was information, not prose.

## Beyond cost: matching models to agent subtasks

The three-tier setup above is really a special case of a more general principle: **an agent loop is not one model call, it's five different jobs**, and each one needs a different kind of intelligence.

| Subtask | What it needs | Our pick |
|---------|--------------|----------|
| Planning | Deep reasoning, long-horizon trade-offs | Opus (frontier, keep) |
| Execution | Tool-call reliability, instruction following | GLM-4.6 (mid-tier, agent-tuned) |
| Verification | Checking output against a rubric | Cheap model or self-check |
| Context compression | Summarizing long trajectories | Cheapest available |
| Routing | Classifying task difficulty | Heuristics or tiny model |

A few patterns that fell out of this:

**Start cheap, escalate on signal.** Not every task needs the planner. Simple, well-specified tasks go straight to the execution tier; only when the executor stalls (repeated test failures, tool-call loops, judge rejections) does the task escalate to Opus for replanning. The escalation triggers are mechanical — no judgment call required.

**Verification is underrated and cheap.** A smaller model checking the executor's work against explicit criteria catches a surprising amount — wrong files touched, tests skipped, requirements missed. The rubric matters more than the model's raw intelligence here. This is also where most of our human interventions used to happen; the judge tier absorbed a large share of them.

**Compress, don't truncate.** Long agent trajectories blow up context and cost. Summarizing the trajectory with a cheap model between iterations keeps the planner's context dense and relevant. This is pure cost savings with no quality trade-off — the summary just needs to preserve decisions and dead ends.

**Log the routing, retro the mistakes.** Every task records which tier handled which subtask. In our weekly review, the most valuable category turned out to be *routing errors*: tasks that went to the execution tier but needed the planner, and vice versa. Each one becomes a routing rule. The router is the only part of the system that compounds — everything else is just inference.

The meta-lesson: model selection isn't a one-time architecture decision. It's a **routing policy** that you keep tuning, and the tuning data comes free from your own logs.

## The math

Rough monthly numbers for our workload (~$1,750/month baseline):

- Keep Opus planning: ~$400–500 (unchanged, worth it)
- Execution Sonnet → GLM-4.6: ~$1,200 → ~$270 (**saves ~$930**)
- Docs → local/cheap tier: small absolute savings, but essentially free

**Total: ~$1,750 → ~$700–800/month.** No hardware purchased, no infra to operate. If we later add the Mac Studio for the highest-volume execution subtasks, payback is 3–4 months — but only *after* the API switch, not before. (Do the free optimization first; hardware second.)

## Takeaways

1. **Split plan and execute economically.** They're different workloads with different quality bars. Optimize them separately.
2. **Don't buy hardware for a model that doesn't fit on it.** Check quantized sizes before checking prices — 404GB does not go into 256GB.
3. **The cheapest model that passes your eval wins.** "Good enough for execution" is a much lower bar than "good enough for planning." Test with a traffic slice, don't theorize.
4. **For docs, fix the input before upgrading the model.** Rationale in, quality out.

The broader lesson: LLM cost optimization isn't about finding one cheap model. It's about matching each pipeline stage to the cheapest tier that clears its quality bar — and being honest about which stages actually need the frontier.
