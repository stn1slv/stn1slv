---
title: "Migrating off gpt-5: what luna changed, and what it cost to find out"
description: "Moving a ten-call LLM pipeline from gpt-5-mini and gpt-5-nano to gpt-5.6-luna: reasoning effort that does not transfer between models, a baseline that measured the wrong thing, prompt rules the old model had been ignoring, and the billing allowance that decided the model list."
published-at: TBD
author: Stanislav Deviatov
date: Sep-2026
language: en
---

# Migrating off gpt-5: what luna changed, and what it cost to find out

I expected to move an LLM pipeline off the gpt-5 family in minutes. Change three model names in the configuration, run the tests, done. It took most of a weekend, and almost all of that went into discovering that my measurements were worse than either model.

OpenAI is retiring gpt-5, gpt-5-mini, gpt-5-nano and gpt-5-pro on 10 December 2026. My pipeline ran on the two small ones, so the migration was not optional. I evaluated the gpt-5.4 family and gpt-5.6-luna as replacements.

The short verdict: gpt-5.6-luna is a very good model for this class of work, and the allowance it bills against makes it close to free at my volume. It also behaves differently enough from gpt-5-mini that a rename would have quietly changed the output.

## The setup

The pipeline takes in around 2,600 articles a month from a large set of sources and cuts them to roughly 120 candidates that a person then reviews by hand.

That cut is not one model call. It runs on a private engine of mine built on LangGraph: 12 nodes and 29 transitions between them, ten of which are separate model calls, each with its own prompt. Whatever survives the graph goes to a stronger model, and then a human deletes what should not ship.

One structural detail drives everything below. The first stage is a noise filter, not a quality judge, and its two error types cost completely different amounts. Something wrongly kept gets caught later. Something wrongly dropped is gone for good, because the URL is recorded as seen and never comes back.

## Where I landed

Nine of the ten model calls now run gpt-5.6-luna. One, the relevance gate at the very front, stays on gpt-5.4-nano at effort none, for reasons that took most of the weekend to establish. Effort is set explicitly on every call rather than left to the model default, mostly medium, none for summarisation, high for the borderline-score escalation.

On price, the rate card was the wrong thing to read first. About 99.6 percent of my requests bill under OpenAI's data-sharing programme, which grants two separate daily token allowances, and every model belongs to exactly one. Luna shares the 2.5M-a-day allowance with the nano and mini tier, while the stronger 5.6 models sit in a 250k allowance I would exhaust in a few hours. That decided the model list before quality did.

The day-to-day numbers are small. A weekday runs about 1.95M tokens on the old configuration and 1.15M on the new one, which is roughly $4 a month at flex rates and a few cents in practice, because it fits inside the allowance. Per thousand requests the models land far apart: $0.32 for gpt-5-nano, $0.52 for gpt-5.4-nano, $0.65 for luna, $1.49 for gpt-5-mini, and $3.29 for gpt-5.4-mini on a small sample.

One line in the card deserves a close read. Luna is the only model in my lineup that bills cache writes, at 25 percent above its own uncached input rate, and 57 percent of its input billed as writes against 41 percent as reads, because my traffic is spread thinly across the day rather than bunched into cache-friendly bursts. Caching still pays, by 19 percent against not caching, rather than the 90 percent the read discount suggests.

## My baseline was the thing I was trying to replace

I scored each candidate configuration by how closely it reproduced the previous model's output. That produced an alarming 31-article regression and sent me hunting a model defect that did not exist. The flaw is that my first stage produces candidates, and 40 to 60 percent of them get deleted later by a human, so scoring a new configuration against the old one's output measures agreement with a filter I already knew was mediocre. It rewards reproducing its mistakes.

Better ground truth was sitting in version control the whole time. That final human review deletes items from the candidate list, and the deletion lands as a commit. For the test month it took 122 candidates down to 72. Diffing that commit against its parent gives two labelled sets: what a human kept, and what a human threw away. Re-scored against those, 43 percent of the 31 turned out to be articles the human had deleted anyway.

Over a 200-row sample containing 72 human-kept and 49 human-pruned articles:

| configuration | human-kept recalled | human-pruned passed through |
|---|---|---|
| retired: gpt-5-mini and gpt-5-nano | 48 of 72 | 27 of 49 |
| candidate: gpt-5.4-mini on the judgement steps | 33 of 72 | 18 of 49 |
| shipped: luna on nine of ten calls | not measured | not measured |

A real loss of 15 human-kept articles, half the size of the regression I had been chasing. And localisable, because the same labels say which step killed each one:

| step | retired | candidate |
|---|---|---|
| topic filter | 2 | 16 |
| novelty check | 12 | 12 |
| content-type filter | 5 | 5 |
| relevance gate | 4 | 4 |
| elsewhere | 1 | 2 |

One column moved. That is a much better thing to own than a 31-article mystery.

The ground truth has two limits I should have written down earlier. It only contains articles the old pipeline passed, so a new configuration can never be credited for recovering something the old one dropped, and roughly 79 of the 200 rows carry no human label at all.

The run also surfaced a problem that has nothing to do with the migration. The novelty check lost the same 12 of 72 human-kept articles under both models, and it invents topic identifiers absent from the list it is given, a different fabricated identifier for each of its 24 rejections. Roughly 17 percent of the good articles, model-independent, and still open. A model swap is a good moment to find this kind of thing, because it is the only time anyone measures the steps individually.

If any part of your system ends in a human decision, that decision is your ground truth, and it is often already recorded somewhere: approval queues, moderation overrides, ticket reclassifications, edits to generated drafts.

## The new model was enforcing rules the old one ignored

With the loss localised to the topic filter, the reflex was to retune that prompt for the new model. That would have been actively harmful, because a later and stronger stage reads the same prompt files. Loosening a rule to suit a first-stage model degrades the stage doing the real judging, and no first-stage test would show it.

So instead of editing, I captured the model's stated reason for every article the topic filter dropped. About twenty calls.

The reasons were coherent, cited rules by name, and quoted the prompt accurately. One reproduced almost word for word a rule excluding AI cost optimisation "even when implemented at an API gateway", while rejecting an article a human had kept.

gpt-5-mini had simply been ignoring that rule. Two of the rules it ignored were stale: one had no exception for messaging patterns implemented inside an application, and one predated AI gateways existing as infrastructure worth including. I fixed those two and left the rest alone, in a commit separate from the model change so the two could be measured apart.

Of everything in this migration, that is the part I would most want another team to check first. A newer model rejecting more at a judgement step is often better instruction-following running into a prompt that has been drifting for a year. The test is cheap: if the model's stated reasons cite your real rules accurately, your prompt is stale and fixing it helps everywhere. If it misapplies rules that plainly do not fit, the prompt is fine and the model is the problem. Those are opposite conclusions, and a rejection count cannot tell them apart.

## Luna is stricter, and stricter is not the same as worse

The topic filter was one step. The relevance gate at the front of the graph was the other, and there luna rejected twice as many articles as gpt-5.4-nano. "This model is worse at this task" was the obvious read, and I nearly shipped a decision based on it.

That gate returns three separate judgements: is the article recent, is it in English, is its subject in scope. I had been reading only the combined result.

Split apart, at effort none, out of 198 cached articles: gpt-5.4-nano passed 166 and failed 32 on subject scope, 4 of which also failed the language check. Luna passed 134 and failed 64 on subject scope, with the same 4 language failures. The language check is identical, down to the same four articles, so the entire difference is the subject judgement.

Reading luna's stated reasons, they were coherent and mostly defensible. It was rejecting a Go worker-pool library, a voice-agent platform, a cloud database-migration post. It was not broken; it reads "primary focus" more strictly than my prompt intends.

That is a different problem with a different fix. A broken check is worth repairing, but a stricter and defensible reading of an under-specified prompt, sitting at the gate that feeds everything else, is worth routing around, which is why that one call did not migrate. An aggregate score tells you something changed and never tells you what, so if a step returns several sub-judgements, log them separately before forming a theory about the model.

## Effort is a property of a model and a task, not of a model alone

Having decided to keep gpt-5.4-nano at that gate, the remaining question was how hard to make it think. Reasoning models expose a `reasoning_effort` control, and I assumed it was a quality dial with a consistent direction, with each model having its own response to it.

Rejections when one step is fed all 198 cached articles in isolation, so that nothing upstream can vary:

| step | model | none | low | medium | high |
|---|---|---|---|---|---|
| relevance gate | gpt-5-nano (default effort) | | | 20 | |
| relevance gate | gpt-5.4-nano | 30 | | 55 | |
| relevance gate | gpt-5.6-luna | 66 | 61 | 68 | |
| topic filter | gpt-5.6-luna | 168 | 141 | 129 | 119 |

Two shapes in one table. gpt-5.4-nano nearly doubles across the range at the relevance gate while luna sits at 66, 61 and 68, which is noise rather than a trend. The same luna slides 49 articles at the topic filter and is still descending at the top of the range.

So "luna needs high effort" was not something I could learn once and reuse. It was true for luna on one task and meaningless for luna on another. Effort response belongs to a model and task pair, and each pair needs its own ladder. Reaching for a stronger model is no substitute for climbing it: on a separate 80-article probe at that gate, gpt-5.4-mini rejected 31 and luna 34, against 30 for gpt-5.4-nano at medium. Moving up a tier changed nothing. Dropping gpt-5.4-nano to effort none did.

There is a cost side too, though not the one I assumed. Output is where reasoning spends, but output is not what fills the allowance: it was 13 percent of my token usage on the old configuration and 2.6 percent on the new one, because the prompts are long and input dominates. What effort does control is the part that scales without bound. Moving the topic filter from medium to high bought 10 fewer rejections for roughly three times its reasoning tokens. I did not take that trade.

## Find out what your test set can actually resolve

Twice I drew conclusions from differences my measurement could not support.

The same configuration, run three times on the same 200 rows, rejected 37, 48 and 55 percent at the topic filter. That is an 18-point spread at fixed configuration and fixed input, far too wide to be sampling noise, so it is model nondeterminism. End to end, two full runs of one configuration differed by four items gained and four lost. Even the old configuration replayed against its own production output recovered only 75 of the 121 labelled articles, about 62 percent agreement with itself.

So a single run resolves something like 20 points at one step, and about 8 items end to end. I had called one model "genuinely better" on a 10-point single-step difference and had to retract it when the third run landed on the other side. The 15-article loss survives that floor and is concentrated in one step, which is why I trusted it.

What worked instead: attribute a specific change with a small deterministic probe of the exact items it targets, plus negative controls it must not move. Use the wide run only to catch large unintended damage elsewhere. One prompt correction taught me the other half of that lesson, flipping four target items past the step it fixed and moving end-to-end recall by almost nothing, because three of them died later for unrelated reasons.

## What I did not measure

The configuration I shipped was never measured end to end. I stopped that run early, so the complete numbers above describe the gpt-5.4-mini candidate. It has since run in production without errors, which is not the same as without quality loss.

There is one datapoint suggesting luna is worse than gpt-5.4-mini at the content-type filter, 9 human-kept articles lost against 5, at an effort level I failed to record. If the output thins out, that is the first place to look.

The topic filter ladder has no control run. Those rates come from a harness that feeds every article straight to the step, so they are inflated relative to real pipeline counts. I ran the equivalent control at the relevance gate and it validated cleanly, but did not repeat it for the topic ladder, so its shape is trustworthy and its absolute values are not.

One month, one corpus, one domain. The specific numbers will not transfer.

## Appendix: traps in the API surface

Default reasoning effort is not stable across generations. gpt-5 defaulted to medium, the 5.4 family defaults to none, and 5.6 defaults to medium again, so a bare rename changes how much every call reasons. The max level exists only on 5.6 models and fails on every request if you pair it with a 5.4 one. And any effort other than none, on a call that binds function tools, returns a 400 on chat completions and has to move to the responses endpoint; since luna defaults to medium, luna cannot use tools on chat completions at all.

None of these is recoverable at runtime. Retry middleware correctly skips 4xx, so an invalid pair does not fail once and loudly, it fails on every row of the run. Validate at process start, against the specific model. These specifics will age, so check them against current documentation before relying on any of them.

## The short version

A newer model enforcing a rule your old model ignored looks exactly like a regression until you read what it said. That is the part that turns a rename into a weekend, and it is the one I would check first.

If you have run a migration like this, I would be interested to hear whether your effort ladders behaved the same way. Mine were the part I was most confident about and most wrong about.
