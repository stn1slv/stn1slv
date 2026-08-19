---
title: Camel DSL choice is now a harness decision
description: YAML DSL has become the recommended answer for AI-assisted Camel development. In a workflow where the agent authors and a named human approves, I still choose Java DSL, because the DSL decides how much verification you get before a person has to supply it.
published-at: https://www.linkedin.com/pulse/camel-dsl-choice-now-harness-decision-stanislav-deviatov-rajif/
author: Stanislav Deviatov
date: Aug-2026
language: en
---

# Camel DSL choice is now a harness decision

YAML DSL has become the recommended answer for AI-assisted Camel development, and the ecosystem is built to make that recommendation work. The CLI runs a YAML file with no build step, the visual designers read and write it, Kamelets are YAML, and a published JSON schema makes generated YAML checkable. An agent wired to the project's guidance and catalog tooling will hand you YAML.

A model without that context will more often hand you Java, because nineteen years of Camel examples are Java and that is what it learned from. The steer toward YAML is a deliberate choice by the project, not the natural output of a model.

For the workflow it targets, the choice is right. No compilation, no IDE, no Java required, and the file runs as it is. It solves the problem of getting a model to emit something runnable in one shot, and of letting a person try it a second later.

My problem is different. On code-first integration work the agent and its harness write the route and the tests, and a named person approves the merge and owns the incident afterwards. For that loop I choose **Java DSL**. Not because it reads better in the abstract, but because it decides how much of the review can be delegated to deterministic tooling before a human spends attention on it.

## 1. The artifact set changed, not the framework

Camel is the same. What an integration engineer produces is not. Three artifacts now matter, and the route is the cheapest of them.

The **specification** is the contract the route must satisfy, in a form a machine can check. The **harness** is the compiler, type checker, schema validation, contract tests, mock endpoints and CI gates the agent must pass before anyone looks at the work. The **route** is generated and regenerated at near zero cost, although owning it in production is as expensive as it ever was.

So the design question is no longer which syntax I enjoy writing. It is which syntax lets the harness reject the most defects without me. What the harness catches is free at review time, once you have reviewed the harness itself. What reaches me costs review attention, and in an agent workflow review attention is the binding constraint.

That budget is smaller than most teams assume. Defect detection degrades as a diff grows, and it collapses well before the volume a single agent can produce in an afternoon. A model can also be wrong with complete confidence and no signal that anything is off, which removes the instinct a reviewer relies on when reading a colleague's code. A weak harness does not slow any of this down. It moves the defects to production faster.

One rule follows, and it is easy to skip while the agent is doing well. The harness cannot sit inside the agent's write scope. If the agent can relax a threshold, adjust a CI gate or edit a test resource, a green build only tells you it found a way to make the build green. Changes to the harness belong in their own review, held to a higher standard than the routes they guard.

## 2. Compactness is review bandwidth and context budget

The Camel manual publishes the same content-based router in all three DSLs. Measured as published:

- Java DSL: 7 lines, about 230 characters
- XML DSL: 16 lines, about 410 characters
- YAML DSL: 18 lines, about 490 characters

Roughly 1 to 1.8 to 2.1 by characters. The cause is structural. Java allows fluent chaining, so a predicate and its destination sit on one line:

```java
.when(header("type").isEqualTo("widget")).to("bean:widgetOrder")
```

The same rule in YAML is four nested keys, and in the full route the deepest line carries 22 leading spaces:

```yaml
- simple: "${header.type} == 'widget'"
  steps:
    - to:
        uri: bean:widgetOrder
```

Nesting is the syntax in both markup DSLs, so neither compresses the way a fluent chain does. XML can be crammed onto one line, but nobody formats it that way and no generator emits it. The gap widens with EIP complexity, which is where a reviewer needs to concentrate most.

**Diff noise.** Renaming a queue in Java DSL touches one line. In YAML it touches a nested key inside indentation that makes the hunk larger than the change. At one route this is irrelevant. At a pull request with a dozen generated routes it decides whether I read the diff or approve it on reputation.

**Context budget.** In an agent loop the route is not written once. It stays in the context window and is re-sent while the agent iterates. Fewer characters means fewer tokens held in the window, and on a multi-route task that decides how much of the earlier reasoning is still there before the agent starts contradicting decisions it made twenty turns ago. Prompt caching lowers what this costs in money. It does not give the window back.

I measured characters rather than tokens, and tokenizers compress runs of whitespace, so YAML's indentation penalty is smaller than the character count suggests. The narrow claim that survives is that for Camel route definitions Java is the most compact of the three, and that compactness is now an operational property rather than a matter of style.

## 3. Failure timing decides who pays

Hallucinated component options and invented URI parameters are normal in every DSL. General-purpose models do not carry a reliable copy of the Camel catalog. What differs is when the error becomes visible, and therefore who absorbs it.

The naive version of the argument is that Java catches typos and YAML does not. That is wrong, and both sides have more build-time checking available than most projects switch on.

A URI string is opaque to the Java compiler. `kafka:orders?brkers=localhost:9092` compiles cleanly and fails when the endpoint is resolved at startup, exactly as it would in YAML. What the compiler rejects is the code around the URI: a method that does not exist on that EIP, an argument of the wrong type, a processor whose signature drifted since the route was written, a bean reference that no longer resolves to a class. Route structure, not endpoint configuration.

Endpoint configuration has its own build step, and it exists. The `camel-report:validate` goal parses the source, checks endpoint URIs and Simple expressions against the catalog, and reports an unknown option with a suggested correction, which is the mistake an agent makes most often. It covers Java and XML. The YAML equivalent, `camel-yaml-dsl-validator:validate`, checks routes against the generated JSON schema, so it catches a misspelled key or a missing `steps` block. Schema conformance rather than component options, and the plugin does not run Camel's route loader, so it will not see custom step names contributed by a module.

Both plugins default to `failOnError=false`, so out of the box they log a warning and the build stays green. A warning is worth nothing in an agent harness, because the agent reads the exit code and moves on. Turn `failOnError` on or stop claiming to have the gate. The Endpoint DSL removes the URI question at the language level, at the cost of more verbose route code, which makes it worth enabling for the components an agent tends to improvise on.

What survives is still an advantage, just a narrower one. The compiler checks route structure on every agent iteration, before any gate is configured and at no human cost. The URI checks have to be wired up, and for YAML the wired-up check answers a different question.

One failure mode runs the other way. Block nesting in Java DSL is expressed through method calls, so a missing `end()` often compiles and quietly attaches the following steps to the wrong branch. In YAML the nesting is the structure and that mistake is much harder to make. It is exactly the kind of defect a compact diff hides well.

The Camel MCP server and Language Server narrow the remaining gap by validating routes and completing URIs against the live catalog. They work, and they are also components to install, version and keep running. On total cost of ownership, a check the build already performs beats a service you have to operate.

Java DSL carries capability the others do not, and some of it is directly a review affordance. Boolean logic is available everywhere inside a `simple` expression. What only the Java DSLs allow is treating a predicate as an object: naming it, and combining named predicates with `and`, `or` and `not`, including across different expression languages. A route can then read as business rules instead of inline expressions:

```java
Predicate isWidget = header("type").isEqualTo("widget");
```

```java
from("jms:queue:order")
   .choice()
      .when(isWidget).to("bean:widgetOrder")
```

The reviewer verifies the rule once, then reads intent. Inline processors and `LambdaRouteBuilder` extend the same property. There is also a registry detail worth knowing: beans declared inside XML or YAML are registered in the Camel registry, which is not your Spring or Quarkus bean container, so in a code-first application that is a second place to reason about during review.

## 4. The agent should not be trusted with all of its own tests

When the agent writes the route, the tests are what I review, and that is where the sharpest failure mode lives. Agents optimise for the tests they can see. Given a suite they also wrote, they will satisfy it, including by weakening or deleting the assertions that stand in the way. The larger the change, the wider the gap between passing the visible tests and being correct.

This inverts the review order. The route is verified by the tests. The tests are verified by nobody, so they are read first and hardest. In practice I read the contract change, then the test diff, then the route, and the route is usually where I spend the least time.

Coverage is close to meaningless when the same process wrote both sides. What works is a mutation threshold, property-based tests for edge cases, and at least one acceptance test the agent never saw. That last one costs me real time to write, and it is the part of this workflow that does not get cheaper.

Integration makes it harder. The defects that matter here are rarely logical. They are ordering, redelivery, poison messages, partial failure, idempotency under retry and schema evolution. None of them appear in a unit test generated from a happy-path prompt.

Camel already has the right toolchain: `camel-test-junit5` with `CamelTestSupport`, mock endpoints, `AdviceWith` for replacing inputs and intercepting endpoints without touching the production route, and Test Infra for Testcontainers-backed brokers and databases. All of it is Java.

A YAML route gets expensive here. The CLI can run tests with `camel test`, but the serious Camel test is Java, so a YAML route means the change lands in two languages and I review both. The link between them is also a string. The test names a route id, an endpoint URI or a mock name, and when the agent renames one of those in the YAML the Java test still compiles. It fails at test runtime instead of at build time, or it keeps passing while asserting against a mock that nothing feeds any more. With a Java route that link is a symbol, so a rename either succeeds or breaks the build. When the agent writes both sides, string coupling between them is the seam most likely to hide a defect from me.

Integration has an advantage on the specification side that most domains lack. Our specs already exist in machine-checkable form: OpenAPI and AsyncAPI documents, JSON Schema and Avro in a registry, consumer-driven contracts. Those are better agent input than prose and much better review objects, because CI can decide whether the route satisfies them. Review the contract change first and the route second, since a correct route against a wrong contract is the more expensive mistake.

## 5. The case for YAML, stated properly

YAML DSL has a canonical JSON schema. That allows schema validation and, in principle, grammar-constrained decoding, where the model cannot emit structurally invalid output at all. Java DSL has no equivalent constraint at generation time. For an autonomous pipeline producing routes that no person reads line by line, that is a safety property Java cannot offer. Its limit is that structural validity is not correctness. A schema-valid route can still reference a component that does not exist, or route the wrong message to the wrong queue.

YAML is also the only sensible choice when the route round-trips through Kaoto or Karavan, when it is a Kamelet, when it is a Camel K resource under GitOps, or when the people building integrations do not write Java. And `camel export` removes the old objection that CLI-developed routes cannot reach production.

Hot-reload belongs on its own, because it gets filed under AI tooling when it is not. `camel dev` optimises the loop of a person editing a file and looking at what happens. An agent gains nothing from a reload it cannot observe; to use that loop it has to send a test message and read the trace back, which is machinery someone has to build. A compile step hands the agent a deterministic pass or fail with no wiring at all. The reload loop serves the writer and the compile loop serves the verifier. Both still matter, but in an AI workflow the one typing is the agent and the one looking is me, so they serve different people.

The cost on my side is real. A build step in every agent iteration is slower than a file save, and it is dead time for the human waiting on the result. Java also carries structure the markup DSLs do not have: a `RouteBuilder` class, imports, and the scaffolding around the route, which is extra surface for the agent to get wrong before it reaches the part that matters.

The two options optimise different loops, and that is the whole disagreement.

## 6. The rule, and what would change it

Java DSL for code-first Spring Boot and Quarkus projects, for routes with non-trivial EIP logic, custom processors or real error-handling requirements, and wherever a named human is accountable for the merge. YAML DSL for visual tooling, Kamelets, Camel K, mixed-skill teams, and for routes running under a different accountability model, where schema validation and a fast rollback stand in for a reviewer instead of supporting one. If XML is required, use `xml-io-dsl` rather than legacy Spring XML. Mixing DSLs inside one service stays a bad idea either way.

Whichever DSL, the agent needs style constraints or compactness turns into density. I ask for one concern per route, explicit route ids, named predicates rather than inline expressions, and business logic in a processor with its own test rather than buried in a chain. Compact code is only reviewable if it was written to be read, and an agent will happily produce a correct route that no one can check.

The threshold that would move me is measurable, and I would run it on my own pipeline: validated-first-attempt rate for schema-constrained YAML against compile-success rate for generated Java, plus median review time per route and defect-escape rate on merged agent pull requests. If YAML wins on all three, the compiler stops paying for itself and I switch.

The standard advice has been that DSL choice is a technicality and mostly a team preference. That was accurate when a person typed the route. Once the agent types it and a person signs it, the DSL is a component of the harness, and it is judged on how much verification it gives you before a human has to supply it.
