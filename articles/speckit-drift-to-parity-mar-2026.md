---
title: "From Drift to Parity: Building a Feedback Loop for Spec-Driven Development"
description: Spec-driven development pipelines generate code from specifications, but have no mechanism to reconcile specs when reality diverges. This article examines the artifact drift problem, introduces the Double-Loop Parity pattern, and presents two community extensions for GitHub Spec-Kit that close the feedback loop.
published-at:
author: Stanislav Deviatov
date: Mar-2026
language: en
---

# From Drift to Parity: Building a Feedback Loop for Spec-Driven Development

## 1. The Rise of Spec-Driven Development

As AI-assisted code generation matures, the industry is moving toward a pattern that reverses the traditional development workflow: instead of writing code first and documenting later, you write the **specification** first and let an AI agent generate the implementation. This approach is known as **Spec-Driven Development (SDD)**.

Tools like GitHub's [Spec-Kit](https://github.com/github/spec-kit) make this workflow practical. The developer creates a `spec.md` (what to build: user stories, acceptance criteria), a `plan.md` (how to build it: architecture, dependencies, routing), and a `tasks.md` (the step-by-step breakdown). Each feature lives in its own folder (e.g., `specs/006-invoice-settings/`), and the standard pipeline is a linear chain of commands:

```
/speckit.specify → /speckit.plan → /speckit.tasks → /speckit.implement
```

For greenfield features, this pipeline works remarkably well. The AI agent treats the specification documents as a contract and produces code that aligns with them. But this forward-only pipeline relies on an assumption that is rarely stated: that the specification remains correct throughout the entire development lifecycle. In practice, this assumption falls apart almost immediately.

## 2. The Two Blind Spots of a Forward-Only Pipeline

SDD tools are designed to move in one direction: from specification to code. They excel at generating new plans for new features. But software development is not a straight line. It is a messy, iterative process where requirements shift, edge cases appear during implementation, and architectural decisions get revised under real-world constraints. The standard pipeline has no answer for what happens when reality diverges from the plan.

### 2.1 Artifact Drift (The PR-Phase Problem)

The first blind spot appears during active development. The specification pipeline produces artifacts, the developer (or AI agent) starts coding, and problems appear: a route was missed, the API contract needs an additional header, the frontend component exists but is unreachable because no one wired the navigation.

Here is a situation I encountered more than once. The backend is complete. Tests pass. The UI component is built. But a user cannot actually reach the new screen because no one added a route to the router and a link in the sidebar. Or the API returns a field called `total_amount`, but the spec says `totalAmount`. Or an endpoint now requires an `org_id` header that was not in the original plan.

Each individual gap is small. But they accumulate. After a few of these, the specification no longer reflects what was actually built. And since the AI agent trusts the spec as its ground truth, any further work based on it produces incorrect results. By the time the Pull Request is merged, the documentation is already a lie. For AI agents relying on this context, this **artifact drift** is toxic: it causes the model to hallucinate corrections for problems that do not exist, or miss problems that do.

The obvious fix is to re-run the full specify-plan-implement cycle. But that is far too expensive for a few missing lines. What is needed is a mechanism to surgically patch the spec, the plan, and the task list based on what actually changed, without rebuilding the entire artifact chain.

### 2.2 Project Memory Loss (The Post-Merge Problem)

The second blind spot appears after the feature is merged. The Spec-Kit pipeline ends at `/speckit.implement`. Once a feature branch lands in main, the framework considers the job done. The feature's spec folder becomes a historical artifact: a snapshot of what the team built during one sprint.

We are trained to think in terms of *tasks* rather than *systems*. We write a spec for a feature, merge it, and move on. After several sprints, the project accumulates a dozen of these disconnected folders. If a new developer (or an AI agent) asks, "How does the invoicing module work right now?", there is no single document to answer that question. They have to reconstruct the picture from scattered feature specs and the codebase itself. There is no consolidated **source of truth** for the current state of the product.

This is analogous to the lifecycle management problem in **Architecture Decision Records (ADRs)**, where every decision carries a status: proposed, accepted, superseded. Without an equivalent mechanism for specifications, feature folders accumulate indefinitely with no clear indication of which artifacts represent current reality and which have been partially or fully replaced by later work.

### 2.3 A Known Pain, No Existing Solution

When I started looking for solutions, I discovered that these were well-known pain points in the Spec-Kit community. Issue [#1063](https://github.com/github/spec-kit/issues/1063) (22 upvotes) described the need for lightweight reconciliation during development. Issue [#1100](https://github.com/github/spec-kit/issues/1100) (11 upvotes) asked for a post-merge consolidation mechanism. In [discussion #1818](https://github.com/github/spec-kit/discussions/1818), the community was actively searching for a way to manage the specification lifecycle, drawing direct analogies to ADR workflows.

But no ready-made solution existed. The core framework provided the forward pipeline and the extension mechanism. The feedback loops were left for the community to build.

## 3. The Solution: Double-Loop Parity

The realization was straightforward: the ecosystem did not need better generation tools. It needed **feedback loops**. Specifically, it needed two of them, operating at different stages of the development lifecycle.

I designed an architectural pattern called **Double-Loop Parity** and implemented it as two Spec-Kit extensions. The term "parity" is deliberate: the goal is not just synchronization (updating artifacts to match code), but maintaining ongoing equivalence between what the specification says and what the system actually does.

- **Inner Loop (Reconcile):** Operates during active development. Closes the gap between the evolving implementation and the feature-level specification artifacts.
- **Outer Loop (Archive):** Operates after merge. Consolidates finalized feature knowledge into a single, canonical project-level memory.

Spec-Kit's extension system allows adding new commands without modifying the core framework. Each extension is a standalone repository with a YAML manifest (`extension.yml`) declaring its commands and dependencies, plus one or more command files that define the execution logic. The two extensions described below follow this architecture.

### 3.1 The Inner Loop: Reconcile

The [Reconcile extension](https://github.com/stn1slv/spec-kit-reconcile) is a **post-implementation gap closer**. It runs during active feature development, after the initial implementation has revealed discrepancies between the plan and reality. Instead of manually rewriting markdown files or re-running the entire pipeline, the developer feeds the extension a natural-language description of what drifted:

```
/speckit.reconcile.run "Backend exists, but React screen is unreachable;
need sidebar link and route"
```

The command executes a five-step workflow:

1. **Discovery and Setup:** Resolves the active feature directory and validates that the required artifacts (`spec.md`, `plan.md`, `tasks.md`) exist. Optionally loads the project's `constitution.md` for architectural constraint checking.

2. **Gap Normalization:** Parses the natural-language input and categorizes the discrepancies into five structured types. **Wiring and Navigation** covers missing routes, menu items, and sidebar links. **Contracts** covers API field mismatches, missing headers, and changed response shapes. **Acceptance Criteria** covers behavior differences from the original plan. **Test Coverage** flags new components without corresponding verification. **Logic/UX** covers missing error handling, toast notifications, and edge cases.

3. **Constitution Compliance:** Cross-checks each proposed change against the project's MUST-level architectural constraints. If a gap fix would violate a constitutional principle, it is flagged as CRITICAL before any edits are made.

4. **Surgical Reconciliation:** Updates only the parts that actually drifted. In `spec.md`, it amends acceptance criteria and user scenarios. In `plan.md`, it updates routing, integration contracts, and testing strategy. In `tasks.md`, it appends remediation tasks with auto-incremented IDs (continuing from the highest existing `T###`) and exact file paths. Each modified file gets a revision note.

5. **Sync Impact Report:** Outputs a structured summary of all changes and recommends the next Spec-Kit command to run (e.g., `/speckit.implement` to execute the new tasks).

**A key design constraint:** if a Wiring or Navigation gap is detected, the extension automatically requires an integration test task. This prevents the most common class of drift (a feature that exists but is unreachable) from being "fixed" without verification.

The scope can be narrowed with modifiers (`--spec-only`, `--plan-only`, `--tasks-only`) when only specific artifacts need updating.

### 3.2 The Outer Loop: Archive

The [Archive extension](https://github.com/stn1slv/spec-kit-archive) is a **post-merge consolidation tool**. Once a feature branch is merged, the extension takes the finalized specification and merges its content into the project's canonical memory:

```
/speckit.archive.run specs/006-invoice-settings
```

The command executes a seven-step workflow:

1. **Setup and Validation:** Resolves the feature directory and the project's memory directory (`.specify/memory/`). If this is the first archival and the memory directory does not exist, bootstraps it from templates or creates it from the feature's content.

2. **Feature Analysis:** Extracts everything of lasting value: user stories, functional requirements (detecting the project's ID convention, whether it is FR-XXX, REQ-XXX, or unnumbered), entities, dependencies, architecture changes, known issues from research notes, and task completion counts.

3. **Conflict Detection:** Systematically checks for issues before merging. **Constitution Compliance** flags any feature content that conflicts with architectural MUST principles. **Requirement ID Collisions** are detected when a feature ID already exists in the main spec. **Entity Redefinitions** highlight differences when an entity is modified, not just added. **Dependency Conflicts** note version mismatches with existing dependencies.

4. **Clarification:** If conflicts require human judgment, asks targeted questions (maximum five) with structured options. Constitution conflicts are always escalated.

5. **Archival Merge:** Applies edits to the canonical project memory. In `.specify/memory/spec.md`, it merges requirements with continued ID numbering, adds user stories, and updates entities. In `.specify/memory/plan.md`, it adds new dependencies, modules, configuration, and routing, and removes implemented items from "Future Work." In `.specify/memory/changelog.md`, it appends a structured entry with branch name, summary, new components, and task completion ratio. In the **agent knowledge file** (GEMINI.md / CLAUDE.md / AGENTS.md), it updates active technologies, project structure, recent changes, and merges known issues from research notes.

6. **Traceability:** Every merged element is tagged with `[Source: specs/006-invoice-settings]`, creating an audit trail from any requirement in the project memory back to the feature that introduced it.

7. **Archival Report:** Outputs a structured summary of all changes, bootstrapped files, constitution compliance status, conflicts resolved, and recommended follow-up actions.

The feature folders are not deleted. They remain as read-only historical "deltas" of work done, while `.specify/memory/` becomes the living, module-level specification. Instead of a growing collection of disconnected feature folders, the project builds a unified memory that serves as the source of truth for onboarding developers and providing context to AI agents.

## 4. Prompt-as-Code: An Architectural Note

One detail worth highlighting: neither extension contains any traditional source code. The entire logic lives in structured Markdown files that serve as instructions for an AI agent. Each extension consists of:

- A **YAML manifest** (`extension.yml`) that declares the extension ID, version, Spec-Kit version dependency, required scripts, and the command it provides.
- A **Markdown command file** (`commands/reconcile.md` or `commands/archive.md`) that defines the agent's role, step-by-step workflow, validation rules, output format, and done criteria.

This is **"prompt-as-code"**: version-controlled, reviewable, diffable, and testable, but executed by an LLM rather than a traditional runtime. The approach fits naturally into the Spec-Kit extension architecture, where every command is essentially a structured prompt that the AI agent follows within the project's context.

The Spec-Kit ecosystem hosts a [growing catalog of community extensions](https://github.com/github/spec-kit/blob/main/extensions/README.md#available-community-extensions), all following this pattern. The prompt-as-code model makes it easier to contribute (no build toolchain, no language dependency) while keeping the discipline of a versioned, schema-validated plugin system.

## 5. Why This Matters Beyond Housekeeping

As we offload more code generation to AI, the quality of the context we provide becomes our most important output. LLMs and agents are only as good as the specifications they read. If we let our specs drift, our AI tools will hallucinate corrections for problems that do not exist, or miss problems that do. The specification is no longer just a planning artifact. In a spec-driven workflow, it is a **runtime dependency**.

Building feedback loops that continuously reconcile code with documentation is not just good engineering practice. It is the infrastructure required for reliable, autonomous AI agents. The Double-Loop Parity pattern addresses this by ensuring that the specification stays truthful at every stage: during active development (Reconcile) and after features are consolidated into the project's memory (Archive).

## Conclusion

Both extensions were included in the [Spec-Kit v0.3.1 release](https://github.com/github/spec-kit/releases/tag/v0.3.1) and are now the 14th and 15th community extensions in the official catalog.

If you are exploring Spec-Driven Development, Doc-as-Code, or trying to make your AI agents more reliable, you can install either extension directly:

```bash
# Install Reconcile (Inner Loop)
specify extension add reconcile \
  --from https://github.com/stn1slv/spec-kit-reconcile/archive/refs/tags/v1.0.0.zip

# Install Archive (Outer Loop)
specify extension add archive \
  --from https://github.com/stn1slv/spec-kit-archive/archive/refs/tags/v1.0.0.zip
```

**Links:**
- [GitHub Spec-Kit](https://github.com/github/spec-kit)
- [spec-kit-reconcile](https://github.com/stn1slv/spec-kit-reconcile)
- [spec-kit-archive](https://github.com/stn1slv/spec-kit-archive)

How is your team handling documentation drift in AI-assisted workflows? Are your specifications staying truthful after merge, or are you rebuilding context from code?
