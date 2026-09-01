# Reasoning effort is not a property of the model

We budgeted one day to swap models in an LLM pipeline. It took a week, and almost nothing we learned was about the new models. It was about how badly we had been measuring the old ones.

OpenAI is retiring gpt-5, gpt-5-mini and gpt-5-nano on 10 December 2026. Our content pipeline ran on the last two, so the migration was not optional.

Here is what the measurements said, and what I would tell anyone running a similar system.

## The setup

The pipeline pulls around 2,600 articles a month from RSS feeds and cuts them to roughly 120 candidates for a curated newsletter.

That cut is not one model call. It is a chain of six, each with its own prompt: a relevance gate, a content-type filter, a topic filter, a novelty check, scoring, and summarisation. Whatever survives goes to a stronger model, and then a human deletes what should not ship.

One structural detail drives everything below. The first stage is a noise filter, not a quality judge, and its two error types cost completely different amounts. Something wrongly kept gets caught later. Something wrongly dropped is gone for good, because the URL is recorded as seen and never comes back.

## 1. Effort is a property of a model and a task, not of a model

Reasoning models expose a reasoning_effort control. We assumed it was a quality dial with a consistent direction, and that each model had its own response to it.

Two measurements, both on the same 198 articles, with each step run in isolation so nothing upstream could vary.

At the relevance gate, rejections out of 198:

- gpt-5-nano, the model we were replacing: 20
- gpt-5.4-nano at effort none: 30, and at medium: 55
- gpt-5.6-luna at none, low, medium: 66, 61, 68

One model nearly doubles across the effort range. The other is flat, because 66 / 61 / 68 is noise rather than a trend.

Now the same luna model on a different task in the same pipeline, the topic filter:

- effort none: 168
- low: 141
- medium: 129
- high: 119

A 49-article slope, still descending at the top of the range.

So "this model needs high effort" was not something we could learn once and reuse. It was true for luna at one task and meaningless for luna at another.

If you tune effort, tune it per model per task. And do not carry a setting across a model swap expecting the behaviour to come with it. We learned that half the expensive way: carrying medium across a model change at the relevance gate roughly doubled rejections and cost seven net inclusions on an 80-article sample, while we were still convinced the model was the variable.

There is a cost side too. Output tokens are where reasoning spends, and on a high-volume step they dominate the bill. Our topic filter alone runs about 309k tokens a day. Moving it from medium to high bought 10 fewer rejections for roughly three times the tokens. We did not take that trade.

## 2. Your baseline is probably the thing you are trying to replace

This invalidated a week of conclusions, and it is the mistake I would most expect other teams to be making.

We scored each candidate configuration by how closely it reproduced the previous model's output. That produced an alarming 31-article regression and sent us hunting a model defect that did not exist.

The flaw is obvious once you see it. Our first stage produces candidates, and 40 to 60 percent of them get deleted later by a human. Scoring a new configuration against the old one's output measures agreement with a filter we already knew was mediocre. It rewards reproducing its mistakes.

Better ground truth was sitting in version control the whole time. Every month a human deletes items from the draft before publication, and that deletion is a commit. For our test month it took 122 candidates down to 72. Diffing that commit against its parent gives two labelled sets: what a human kept, and what a human threw away.

Re-scored against those, 43 percent of the regression evaporated. Fifteen of 35 "lost" articles were ones the new models correctly dropped and the human had deleted anyway.

The honest numbers, over 200 articles containing 72 human-kept and 49 human-pruned:

- Old configuration: recalled 48 of 72 kept, and passed through 27 of 49 that were pruned
- New configuration: recalled 29 of 72, and passed through 15 of 49

A real regression, but a third smaller than the one we had been chasing. And now localisable. Broken down by which stage killed each human-kept article, old against new: topic filter 2 to 16, novelty check 12 to 12, content-type filter 5 to 5, relevance gate 4 to 4.

Every stage flat except one. That table ended the investigation.

If any part of your system ends in a human decision, that decision is your ground truth, and it is often already recorded somewhere: approval queues, moderation overrides, ticket reclassifications, edits to generated drafts. Using your previous model's output instead measures the wrong thing, and it produces confident conclusions you later delete.

## 3. Decompose the metric before you diagnose the model

When luna rejected twice as many articles at the relevance gate, "this model is worse at this task" was the obvious read, and we nearly shipped a decision based on it.

That gate actually returns three separate judgements: is the article recent, is it in English, is its subject in scope. We had been reading only the combined result.

Split apart, at effort none, out of 198: gpt-5.4-nano passed 166, failed 32 on subject scope and 4 on language. Luna passed 134, failed 64 on subject scope and the same 4 on language.

The language check is identical, down to the same four articles. The entire difference is the subject judgement.

And when we read luna's stated reasons, they were coherent and mostly defensible. It was rejecting a Go worker-pool library, a voice-agent platform, a cloud database-migration post. It was not broken. It reads "primary focus" more strictly than our prompt intends.

That is a different problem with a different fix. A broken check is worth repairing. A stricter but reasonable reading of an under-specified prompt, sitting at the gate that feeds everything else, is worth routing around. We kept the older model at that one step and moved the other nine to luna.

An aggregate score tells you something changed. It never tells you what. If a step returns several sub-judgements, log them separately before forming a theory.

## 4. Read what the model said before you rewrite the prompt

With the loss localised to one prompt, the reflex was to retune that prompt for the new model. In our case that would have been actively harmful, because a later and stronger stage reads the same prompt files. Loosening a rule to suit a weaker model degrades the stage doing the real judging, and no first-stage test would show it.

So instead of editing, we captured the model's stated reason for every article it dropped. About twenty calls.

The reasons were coherent, cited rules by name, and quoted the prompt accurately. One reproduced almost word for word a rule excluding AI cost optimisation "even when implemented at an API gateway", while rejecting an article a human had kept.

The old model had been under-applying rules the new one enforces. Two of those rules were genuinely stale: one had no exception for messaging patterns implemented inside an application, and one predated AI gateways existing as infrastructure worth covering.

That reframes the work entirely. These were not accommodations for a new model. They were scope corrections the rest of the pipeline needed anyway.

The test is cheap enough to build into any prompt-tuning loop. If the model's stated reasons cite your real rules accurately, your prompt is stale and fixing it helps everywhere. If it misapplies rules that plainly do not fit, the prompt is fine and the model is the problem. Those are opposite conclusions, and a rejection count cannot tell them apart.

## 5. Find out what your test set can actually resolve

Twice we drew conclusions from differences our measurement could not support.

The same configuration, run three times on the same 200 articles, rejected 37, 48 and 55 percent at one stage. Two full runs of one configuration differed by four items gained and four lost.

So a single 200-article run resolves something like 20 points, not 10. We had called one model "genuinely better" on a 10-point difference and had to retract it when the third run landed on the other side.

What worked instead: attribute a specific change with a small deterministic probe of the exact items it targets, plus negative controls it must not move. Use the big run only to catch large unintended damage elsewhere.

A related surprise. One prompt correction flipped four target items past the stage it fixed, and moved end-to-end recall by zero, because three of the four then died at later stages for unrelated reasons. When several stages reject the same category on different grounds, fixing one buys nothing you can measure downstream.

## 6. Two operational traps

The first is a transport constraint. Every step in our pipeline binds function tools, and OpenAI rejects that combination on the chat completions endpoint with any reasoning effort other than none. You get a 400 telling you to use the responses endpoint instead.

We verified it across two model families, so it is not one generation's quirk. It also means a model whose own default effort is medium cannot use tools on chat completions at all, even if you never pass the parameter.

This matters more than a normal error because it is unrecoverable at runtime. Retry middleware skips 4xx, so a bad model and effort pair fails every request in a run rather than failing once, loudly. We now validate effort values at process start. A related one caught us on the same switch: one service tier value is accepted on the old endpoint and rejected on the new one, so a config value that had been fine for months became a per-request failure.

The second is billing. Roughly 99.5 percent of our requests bill under a data-sharing programme granting two separate daily allowances, and each model belongs to exactly one. Our workload saturates the larger allowance on peak days while the smaller sits idle, which looks like spare capacity until you check the arithmetic. One step alone needs about 124 percent of that smaller allowance.

That single number eliminated an entire tier of models from our highest-volume step, regardless of quality or price. If your provider has anything similar, the allowance structure may constrain model choice more than the rate card does.

## What we did not measure

Three limits, because several conclusions depend on them.

The configuration we shipped was never measured end to end. We stopped the full run of the final layout early. The complete numbers above describe an earlier configuration. It has since run in production without errors, which is not the same as without quality loss.

The topic filter ladder has no control run. Those rates come from a harness that feeds every article straight to the step, so they are inflated relative to real pipeline counts. We ran the equivalent control at the relevance gate and it validated cleanly. We did not repeat it for the topic ladder, so its shape is trustworthy and its absolute values are not.

One month, one corpus, one domain. The specific numbers will not transfer.

## The short version

Tune reasoning effort per model per task, and measure it rather than inheriting it.

Log a step's sub-judgements separately, or you will diagnose the wrong component.

Find the human decision in your system and score against that, not against your previous model's output.

Before rewriting a prompt for a new model, read what the model said about the items it dropped. A stale rule and a misreading need opposite fixes.

Establish what your test set can resolve before you trust a difference it reports.

If you have run a migration like this, I would be interested to hear whether your effort ladders behaved the same way. Ours were the part I was most confident about and most wrong about.
