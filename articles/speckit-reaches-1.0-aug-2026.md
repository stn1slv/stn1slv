---
title: "Spec Kit Reaches 1.0: From One Pipeline to Five Primitives"
description: In February 2026 Spec Kit was a four-command pipeline that had just gained an extension system. In August 2026 it shipped 1.0.0 with five primitives, bundles that compose them and catalogs that distribute them.
published-at: https://www.linkedin.com/pulse/spec-kit-reaches-10-from-one-pipeline-five-primitives-deviatov-bdmcf
author: Stanislav Deviatov
date: Aug-2026
language: en
---

# Spec Kit Reaches 1.0: From One Pipeline to Five Primitives

On 21 August 2026, one year after its first commit, GitHub's [Spec-Kit](https://github.com/github/spec-kit) released 1.0.0, followed the same day by 1.0.1. I wrote about the project in March in [From Drift to Parity](https://www.linkedin.com/pulse/from-drift-parity-building-feedback-loop-spec-driven-deviatov-o7zcf), when I published two extensions for it. The tool I wrote about then and the tool that shipped 1.0 are recognisably related, but the distance between them is larger than six months of version numbers suggests.

## 1. Where things stood in February 2026

In February, Spec Kit was a linear pipeline: `/speckit.specify`, `/speckit.plan`, `/speckit.tasks`, `/speckit.implement`. You picked your coding agent with an `--ai` flag, and that was most of the configuration surface.

Two things happened around then. Manfred Riem took over as lead maintainer on 22 January 2026, and on 10 February version 0.0.93 merged a modular extension system contributed by Michal Bachorik. That was the first point where you could change the tool without forking it, and the project's own [history page](https://github.com/github/spec-kit/blob/main/docs/history.md) treats it as the start of the current phase. The maintainers then stopped adding features to the pipeline and started adding places where other people could add them. That model is now described as five primitives: integrations, extensions, presets, workflows and workflow steps.

## 2. Extensions: adding capabilities

Extensions add commands, templates, scripts and hooks. They are declared in an `extension.yml` manifest and installed with `specify extension add`. This is the layer my own two extensions sit in, and the one the community adopted fastest: the documentation counted 157 community extensions at 1.0.

An extension declares a category, an effect (`read-only` or `read-write`), a minimum Spec Kit version, hooks with priority ordering, and its own templates and scripts. The effect field earns its place. It sorts a hundred strangers' extensions into "safe to try" and "read the source first".

## 3. Presets: changing how the core behaves

Presets arrived in 0.3.0 on 13 March 2026 and solve the opposite problem. The cleanest formulation is the maintainer's: an extension adds new slots, a preset reshapes existing ones, whichever layer those slots came from.

Two additions made presets practical rather than merely possible. Priority ordering lets several presets stack, each file resolved independently from the highest-priority source that provides it. Composition strategies, added in 0.8.0 on 23 April, let a preset prepend to, append to, or wrap core content instead of replacing it. Wrapping is the useful one: the preset writes `{CORE_TEMPLATE}` where the lower-priority content should land, so a compliance preset can add a security section to the plan template without owning that template every time the core changes.

The stack resolves at runtime, per file: project-local overrides, presets by priority, extensions by priority, then the core. Presets sit above extensions, so a preset can reshape an installed extension as readily as the core. That reach is what lets one layer bring a whole setup in line with organisational standards.

## 4. Integrations: agents became plugins

Support for coding agents used to be conditional logic inside the CLI. Over 1 and 2 April, releases 0.4.4 and 0.4.5 migrated every agent to a registry-backed plugin architecture and removed the legacy scaffold path. On 16 April, 0.7.2 added an integration catalog with discovery, versioning and independent distribution.

Agent support became data rather than code. The documentation lists 38 integrations at 1.0, and adding another no longer requires touching the core. It also settled the split between slash commands and agent-native skills: agents that support skills install `speckit-*` skills instead of prompt files, and both layouts go through the same manifest system.

The detail worth an architect's attention is file provenance. An integration writes real files into your repository and records a SHA-256 hash of each one. Uninstall removes only files that still match their hash, so anything you edited by hand survives the integration that created it, and upgrade refuses to touch modified files unless you pass `--force`. That is the difference between a scaffolder and something you can keep in a project for two years.

## 5. Workflows and steps: the pipeline became a definition

The workflow engine landed in 0.7.0 on 14 April. A workflow is a YAML file describing a sequence of steps: `command`, `prompt`, `shell`, `init`, `gate`, `if`, `switch`, `while`, `do-while`, `fan-out` and `fan-in`. Runs persist their state, so a run that pauses at a human approval gate resumes from the exact step where it stopped, hours later and on another machine if needed.

This is the change with the largest consequences. The standard specify-to-implement sequence is now a workflow definition rather than hard-coded control flow, so it can be replaced and composed with steps the core never shipped. Version 0.11.0 on 16 June added a step catalog for community step types. By late July there were also overlays, small YAML files that insert, replace or remove steps in an installed workflow without editing it, and which live outside the installed directory so your customisation survives the next update.

There is no privileged core step type. Every step implements one small interface, and the control-flow steps are ordinary implementations of it: `if` evaluates a condition and returns a branch as its next steps, and the engine knows nothing about branching beyond following what a step hands back. A custom step is two source files fetched over HTTPS, with no archive and no install script, and it cannot shadow a built-in name.

One caution. A `shell` step runs with your privileges in no sandbox, and a custom step type is arbitrary code in your process. The project rejects a `requires.permissions` field precisely because it would imply enforcement that does not exist.

## 6. Catalogs: the governance layer nobody advertises

By June there were five kinds of installable thing, and provisioning a team meant a list of install commands. Version 0.11.4 on 22 June added `specify bundle`. A bundle pins a curated set of extensions, presets, workflows and steps to specific versions, so a whole role installs in one command. A bundle composes but does not execute: every component still installs through its own machinery, which is why removing a bundle removes only what it contributed.

Underneath sits a catalog system that works the same way for every primitive: a priority-ordered stack of environment variable, project config, user config, then built-in sources, each carrying an install policy. Read as a trust story, that explains why the built-in `community` catalog is discovery-only. You can search and inspect it but not install from it, because it is an open and unvetted list. The maintainers verify that an entry is well formed, not that the code behind it is safe.

Read as a platform story it is more interesting, and this is the part I have not seen discussed much. An organisation publishes its own catalog of vetted components at priority 1. From that moment `search` and `add` resolve the approved list first, and engineers keep typing the same commands while "what is installable" quietly becomes "what we approved". The discovery-only and install-allowed split turns that into a review pipeline: candidates are visible but not installable, and promotion means moving an entry between two files. Catalogs live in project config, so the approved set is versioned and diffable and the vetting record sits in pull request history rather than a wiki. One priority-and-policy model covers all five primitives.

## 7. The core loop closed, and how it got closed

The most instructive change at 1.0 is not a feature but a mechanism. Until June the reference sequence ended at `implement`, and the gap right after it, whether the code actually matches the spec, was according to the lead maintainer's own [field guide](https://www.manorrock.com/blog/2026/07/08/spec_kit_field_guide.html) the most-cited criticism of the tool. Independent authors filled that gap with extensions before the core did. Then the core adopted the pattern. `/speckit.converge` shipped on 18 June in 0.11.2, assessing the codebase against the spec, plan and tasks and appending the missing work under a Convergence section in `tasks.md`.

The ordering matters more than the command. Reconcile and Archive entered the community catalog on 17 March in 0.3.1, three months before converge shipped. The community proposes, the catalog measures what is growing, and the core promotes the winners. If you are deciding whether to build on this project, that is a more useful signal than any roadmap, because the catalog is public and you can read the direction off it yourself.

The full core sequence at 1.0 is constitution, specify, clarify, plan, checklist, tasks, analyze, implement, converge, with the middle three as optional quality gates. Two bundled opt-in extensions widened the scope beyond feature delivery: `bug` in 0.9.5 runs assess, fix, test, and `assess` in 0.13.0 takes a raw idea through intake, research, define, shape and decide.

What the core still refuses to decide is what happens to `spec.md`, `plan.md` and `tasks.md` after requirements change. The documentation names three persistence models and leaves the choice to the team as a convention rather than a setting. Flow-back allows edits to start in any artifact and reconciles the set afterwards, which is fast and risks silent drift. Flow-forward treats completed feature directories as immutable history, so a new requirement means a new directory, which is auditable and duplicative. Living spec makes `spec.md` the contract and regenerates plan and tasks from it, which stays consistent and can lose implementation rationale. Choosing one is a real architectural decision, and it is the decision my two extensions exist to support.

## 8. What broke on the way

If you are still running a project initialised in the spring, two changes matter.

Version 0.10.0 on 9 June removed the `--ai`, `--ai-commands-dir` and `--ai-skills` flags in favour of `--integration` and `--integration-options`. The same release made the git extension opt-in and removed `--no-git`.

Agent context file updates moved out of the core into a bundled `agent-context` extension in 0.9.0, auto-enabled at first for compatibility, then fully opt-in from 0.12.0. If your `CLAUDE.md` or `AGENTS.md` quietly stopped updating itself, that is why.

## 9. What the version number claims, and what it does not

The lead maintainer's [anniversary post](https://www.manorrock.com/blog/2026/08/21/spec_kit_turns_one.html) is unusually direct. 1.0.0 is not a feature release and not a stability promise. His argument is that a major version used to be insurance against the cost of a breaking change, and that agents have made that cost collapse: you point an agent at the diff and it updates the call sites. The number marks coherence between the five primitives, not permanence. The README still files the project's ambitions under "Experimental Goals", and he deliberately did not rename that heading.

I agree with the direction and would qualify the scope. Migration cost is more than mechanical call-site editing. It is also regression risk, review effort and the time a team spends deciding whether to trust the change. An agent removes the typing. It does not remove the review. On a platform where many teams consume the same catalogs, a stable major version still does coordination work that cheap refactoring does not replace.

What 1.0 does not include is worth stating too. The trust boundary is guidance rather than enforcement, and the maintainer names making it legible as the most valuable unfinished work. Cost visibility is absent from the core; the community built the meters. `taskstoissues` still speaks GitHub, and every other tracker is a community adapter.

Brownfield is the case most often misread. The stock process already handles existing code: a community walkthrough runs the unmodified sequence against a 420,000-line Jakarta EE codebase across 180 Maven modules with no extension at all, and the only special move is a constitution prompt telling the agent to analyse the code first. The brownfield extension cluster is not filling a capability gap. It is paying down the cost of writing that prompt correctly every time. That distinction, between what a tool can do and what it costs you to do repeatedly, is the one I would keep in view here.

For a tool at this stage the trade is a good one. Twelve months produced 38 integrations, 157 community extensions, 33 presets and more than 270 contributors. Freezing the surface to protect that would have cost more than it saved.

## 10. The two extensions, updated

Both extensions from [the March article](https://www.linkedin.com/pulse/from-drift-parity-building-feedback-loop-spec-driven-deviatov-o7zcf) have been updated for the current release. The pattern is unchanged: an inner loop that reconciles artifacts with shipped code, and an outer loop that consolidates finished features into project memory.

[Reconcile](https://github.com/stn1slv/spec-kit-reconcile) is the inner loop, and in the vocabulary of section 7 it is a flow-back tool. The code shipped, the artifacts fell behind, and you describe the drift in plain language so the command amends the feature's own `spec.md` and `plan.md` and appends remediation tasks. It requires Spec Kit 0.16.2 or newer, because it resolves templates through the override stack from section 3 rather than reading the base template. It registers optional hooks on `after_implement` and `after_converge`.

That second hook is where the relationship with `/speckit.converge` becomes clear. Converge treats the artifacts as correct and finds work the code is missing. Reconcile treats the shipped code as correct and amends the artifacts to match. They are mirror cases, a feature often needs both, and reconcile never writes into a Convergence section, so the two outputs stay separable. The core closing one direction did not close the other.

[Archive](https://github.com/stn1slv/spec-kit-archive) is the outer loop, consolidating a merged feature into `.specify/memory/` with item-level source traceability, supersession handling and a changelog entry. It serves the flow-forward and living-spec models, where feature directories are history and the project still needs one current account of itself. It requires Spec Kit 0.14.0 or newer.

One practical note follows from section 6. Because the community catalog is discovery-only, `specify extension update` skips everything in it. Run `specify extension info <name>` for the candidate archive URL, review that archive, then install with `specify extension add <name> --from <url>` and `--force` to overwrite an existing version. That is the trust boundary working as designed, not a gap in the tooling.

The pipeline I wrote about in March had one shape and one extension point. The tool that shipped 1.0 has five primitives, two composition layers above them and a distribution system underneath. The feedback loops still have to be built by whoever needs them, and there are now considerably more places to attach them.
