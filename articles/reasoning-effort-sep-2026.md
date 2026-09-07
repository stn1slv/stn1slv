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

Two sample sets appear in this article and they are not the same. The per-step measurements use a cache of 198 articles fed straight into one step in isolation, so nothing upstream can vary. The end-to-end numbers use a 200-row sample run through the whole graph.

## Where I landed

Nine of the ten model calls now run gpt-5.6-luna. One, the relevance gate at the very front, stays on gpt-5.4-nano at effort none, because luna rejects roughly twice as much there as gpt-5.4-nano does at effort none, and more at every other level.

Effort is set explicitly on every call: medium for the content-type filter, the three fast-track gates, the topic filter and scoring; low for the novelty check; none for summarisation; high for the borderline-score escalation, so that escalation is still a real step up now that model tier is no longer an axis I vary there.

One caveat that belongs here rather than in a footnote. The end-to-end numbers in section 2 below describe an earlier candidate configuration that ran gpt-5.4-mini on the judgement steps, not the configuration I shipped. The luna evidence is narrower: the two effort ladders in section 1, the decomposition in section 3, and the reasons luna gave for its rejections. The shipped layout has run in production without errors, which is not the same as without quality loss.

On price, the per-token rate card turned out to be the wrong thing to read. In the month I pulled the usage export, 99.6 percent of my requests billed under OpenAI's data-sharing programme, which grants two separate daily token allowances, and every model belongs to exactly one. The large allowance is 2.5M tokens a day and covers the nano and mini tier along with gpt-5.6-luna; the small one is 250k a day and covers the mid-size and larger models. On the old configuration I ran about 1.95M tokens on a weekday, so the busy end of the week sat close to the large allowance and one day in thirty went past it at 2.70M.

That decided the model list before quality did. Luna sits in the large allowance, so putting a 5.6-generation model on nine calls costs nothing on most days, while the stronger 5.6 models sit in an allowance I would exhaust in a few hours. If your provider has anything like this, it will constrain model choice more than the rate card does.

The irony is that measuring the migration cost far more than running the pipeline does. The Sunday I ran the harnesses burned 73.1M tokens across 10,901 requests, 1.68 times the other thirty days of the month combined, and almost all of it on the paid flex tier rather than the free allowance, because a burst that size does not wait politely in the daily budget. A week of production traffic is cheaper than one afternoon of evaluating it.

## 1. Effort is a property of a model and a task, not of a model alone

Reasoning models expose a `reasoning_effort` control. I assumed it was a quality dial with a consistent direction, and that each model had its own response to it.

At the relevance gate, rejections out of the 198 cached articles:

- gpt-5-nano, the model I was replacing: 20
- gpt-5.4-nano at effort none: 30, and at medium: 55
- gpt-5.6-luna at none, low, medium: 66, 61, 68

One model nearly doubles across the effort range. The other is flat, because 66 / 61 / 68 is noise rather than a trend.

Now the same luna model on a different task in the same pipeline, the topic filter:

- effort none: 168
- low: 141
- medium: 129
- high: 119

A 49-article slope, still descending at the top of the range.

So "luna needs high effort" was not something I could learn once and reuse. It was true for luna on one task and meaningless for luna on another. Effort response belongs to a model and task pair, and each pair needs its own ladder.

Reaching for a stronger model is not a substitute for measuring that ladder. On a separate 80-article probe at the relevance gate, gpt-5.4-mini rejected 31 and luna 34, against 30 for gpt-5.4-nano at medium. Moving up a tier changed nothing; dropping gpt-5.4-nano to effort none did.

There is a cost side too, though not the one I assumed. Output is where reasoning spends, but output is not what fills the allowance: it was 13 percent of my token usage on the old configuration and 2.6 percent on the new one, because the prompts are long and input dominates. What effort does control is the part that scales without bound. The topic filter alone runs about 309k tokens a day, and moving it from medium to high bought 10 fewer rejections for roughly three times its reasoning tokens, with the ladder steps already shrinking, 27 then 12 then 10. I did not take that trade.

## 2. My baseline was the thing I was trying to replace

This invalidated most of the first day's conclusions, and it is the mistake I would most expect other teams to make during a deprecation migration.

I scored each candidate configuration by how closely it reproduced the previous model's output. That produced an alarming 31-article regression and sent me hunting a model defect that did not exist. The flaw is that my first stage produces candidates, and 40 to 60 percent of them get deleted later by a human, so scoring a new configuration against the old one's output measures agreement with a filter I already knew was mediocre. It rewards reproducing its mistakes.

Better ground truth was sitting in version control the whole time. That final human review deletes items from the candidate list, and the deletion lands as a commit. For the test month it took 122 candidates down to 72. Diffing that commit against its parent gives two labelled sets: what a human kept, and what a human threw away. Re-scored against those, 43 percent of the 31 turned out to be articles the human had deleted anyway.

Over the 200-row sample, which contains 72 human-kept and 49 human-pruned articles:

- Old configuration: recalled 48 of 72 kept, and passed through 27 of 49 that were pruned
- Candidate configuration with gpt-5.4-mini on the judgement steps: 33 of 72, and 18 of 49

A real regression of 15 human-kept articles, half the size of the one I had been chasing, and now localisable. Broken down by which step killed each human-kept article, old against candidate: topic filter 2 to 16, novelty check 12 to 12, content-type filter 5 to 5, relevance gate 4 to 4. Those four account for 23 of the 24 losses in the old configuration and 37 of the 39 in the candidate, so a small remainder died elsewhere. Every step flat except one, which is what ended the investigation and pointed at section 3.

Two limits on this ground truth, both of which I should have written down earlier. The labelled set only contains articles the old pipeline passed, so a new configuration can never be credited for recovering something the old one dropped; roughly 79 of the 200 rows carry no human label at all. And replaying the old configuration against its own production output recovered only 75 of the 121 labelled articles, about 62 percent agreement with itself. That bounds how much the 48 against 33 gap can carry, and it is the same variance problem as section 4.

The run also surfaced a problem that has nothing to do with the migration. The novelty check lost the same 12 of 72 human-kept articles under both the old and the candidate models, and it invents topic identifiers absent from the list it is given, a different fabricated identifier for each of its 24 rejections. That is roughly 17 percent of the good articles, it is model-independent, and it is still open. It is now the next thing I am investigating. A model swap is a good moment to find this kind of thing, because it is the only time anyone measures the steps individually.

If any part of your system ends in a human decision, that decision is your ground truth, and it is often already recorded somewhere: approval queues, moderation overrides, ticket reclassifications, edits to generated drafts.

## 3. The new model was enforcing rules the old one ignored

With the loss localised to the topic filter, the reflex was to retune that prompt for the new model. That would have been actively harmful, because a later and stronger stage reads the same prompt files. Loosening a rule to suit a first-stage model degrades the stage doing the real judging, and no first-stage test would show it.

So instead of editing, I captured the model's stated reason for every article the topic filter dropped. About twenty calls.

The reasons were coherent, cited rules by name, and quoted the prompt accurately. One reproduced almost word for word a rule excluding AI cost optimisation "even when implemented at an API gateway", while rejecting an article a human had kept.

gpt-5-mini had simply been ignoring that rule. Two of the rules it ignored were stale: one had no exception for messaging patterns implemented inside an application, and one predated AI gateways existing as infrastructure worth including. I fixed those two and left the rest alone, in a commit separate from the model change so the two could be measured apart.

Of everything in this migration, that is the part I would most want another team to check first. A newer model rejecting more at a judgement step is often better instruction-following running into a prompt that has been drifting for a year. The test is cheap: if the model's stated reasons cite your real rules accurately, your prompt is stale and fixing it helps everywhere. If it misapplies rules that plainly do not fit, the prompt is fine and the model is the problem. Those are opposite conclusions, and a rejection count cannot tell them apart.

## 4. Luna is stricter, and stricter is not the same as worse

When luna rejected twice as many articles as gpt-5.4-nano at the relevance gate, "this model is worse at this task" was the obvious read, and I nearly shipped a decision based on it.

That gate returns three separate judgements: is the article recent, is it in English, is its subject in scope. I had been reading only the combined result.

Split apart, at effort none, out of 198: gpt-5.4-nano passed 166 and failed 32 on subject scope, 4 of which also failed the language check. Luna passed 134 and failed 64 on subject scope, with the same 4 language failures. The language check is identical, down to the same four articles, so the entire difference is the subject judgement.

That split was a separate run from the ladder above, where the same two configurations rejected 30 and 66. A two-article drift on a 198-article set is the noise floor, which is the subject of the next section.

Reading luna's stated reasons, they were coherent and mostly defensible. It was rejecting a Go worker-pool library, a voice-agent platform, a cloud database-migration post. It was not broken; it reads "primary focus" more strictly than my prompt intends.

That is a different problem with a different fix. A broken check is worth repairing, but a stricter and defensible reading of an under-specified prompt, sitting at the gate that feeds everything else, is worth routing around, which is why that one call did not migrate. An aggregate score tells you something changed and never tells you what, so if a step returns several sub-judgements, log them separately before forming a theory about the model.

## 5. Find out what your test set can actually resolve

Twice I drew conclusions from differences my measurement could not support.

The same configuration, run three times on the same 200 rows, rejected 37, 48 and 55 percent at the topic filter. That is an 18-point spread at fixed configuration and fixed input, far too wide to be sampling noise, so it is model nondeterminism. End to end, two full runs of one configuration differed by four items gained and four lost.

So a single run resolves something like 20 points at one step, and about 8 items end to end. I had called one model "genuinely better" on a 10-point single-step difference and had to retract it when the third run landed on the other side. The 15-article loss in section 2 survives that floor, and it is concentrated in one step, which is why I trusted it.

What worked instead: attribute a specific change with a small deterministic probe of the exact items it targets, plus negative controls it must not move. Use the wide run only to catch large unintended damage elsewhere.

A related surprise. One prompt correction flipped four target items past the step it fixed and moved end-to-end recall by almost nothing, because three of the four then died at later steps for unrelated reasons. When several steps reject the same category on different grounds, fixing one buys very little you can measure downstream.

## 6. Traps in the API surface

Default reasoning effort is not stable across generations: gpt-5 defaulted to medium, the 5.4 family defaults to none, and 5.6 defaults to medium again. A bare model rename therefore changes how much every call reasons, in a direction that depends on which two generations you are moving between. I now set effort explicitly everywhere. Effort values are also per-family, so the max level, which exists only on 5.6 models, passes a naive startup whitelist on a 5.4 model and then fails on every request.

Function tools plus reasoning effort require the responses endpoint. Any effort other than none, on a call that binds function tools, returns a 400 on chat completions telling you to use the responses endpoint instead. I saw it on gpt-5.4-mini and gpt-5.6-luna alike, so it is not one generation's quirk. For luna the consequence is sharper: luna's own default effort is medium, so luna cannot use function tools on chat completions at all, even if you never pass the parameter.

None of the three is recoverable at runtime, which is what makes them expensive. Retry middleware correctly skips 4xx, so an invalid model and effort pair does not fail once and loudly; it fails on every row of the run. The fix is validation at process start, against the specific model, not a generic whitelist. The same gap caught me on a service tier value that the old endpoint accepts and the new one rejects, so a config value that had been fine for months became a per-request failure the moment I switched endpoints.

These specifics will age. Check them against the current documentation before you rely on any of them.

## What I did not measure

Beyond the shipped configuration never being measured end to end, three things.

There is one datapoint suggesting luna is worse than gpt-5.4-mini at the content-type filter, 9 human-kept articles lost against 5, at an effort level I failed to record. If the output thins out, that is the first place to look.

The topic filter ladder has no control run. Those rates come from a harness that feeds every article straight to the step, so they are inflated relative to real pipeline counts. I ran the equivalent control at the relevance gate and it validated cleanly, but did not repeat it for the topic ladder, so its shape is trustworthy and its absolute values are not.

One month, one corpus, one domain. The specific numbers will not transfer.

## The short version

gpt-5.6-luna is worth migrating to. It follows instructions more closely than gpt-5-mini did, and it bills against the same complimentary allowance as the nano and mini models, so on a high-volume pipeline it is effectively free where it matters. It is also markedly less talkative: reasoning output fell from 13 percent of my tokens to 2.6, and tokens per request from 6,874 to 5,953, about 13 percent. Some of that is the new per-call effort settings rather than the model itself, and my daily article volume fell over the same period, so treat the per-request figure as the honest one.

"More closely" is the part that turns a rename into a weekend. A newer model enforcing a rule your old model ignored looks exactly like a regression until you read what it said.

The rest is measurement discipline: do not assume an effort setting survives a model change, log a step's sub-judgements separately, score against the human decision your system already records rather than against your previous model's output, and find out what your test set can resolve before you trust a difference it reports. Check which billing allowance a model belongs to before you evaluate its quality; mine eliminated an entire tier of models regardless of how good they were.

If you have run a migration like this, I would be interested to hear whether your effort ladders behaved the same way. Mine were the part I was most confident about and most wrong about.
