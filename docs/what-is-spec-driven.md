# What Is Spec-Driven Development?

Spec-driven development is a way of building software where the specification comes first — always. Not as documentation written after the fact, but as the contract from which everything else is derived.

The core principle: **the spec is the decision. The code is the consequence.**

## What a spec is (and isn't)

A spec is a structured description of what a system must do. Structured means machine-readable — YAML, JSON, or a format like Gherkin. Not a PRD. Not a Word document. Not a Confluence page.

The difference between a spec and documentation is that a spec is executable. You can run a check against it. If the implementation contradicts the spec, the check fails. Documentation can drift silently for months; an executable spec cannot.

## Why product managers care

In a traditional workflow, a PM writes a PRD describing the feature. An engineer reads it, interprets it, and builds something. The PM reviews the result and finds it doesn't match what they had in mind. This cycle repeats.

In a spec-driven workflow, the PM writes the expected behaviour directly — as test cases, as Gherkin scenarios, as an OpenAPI contract. The AI implements to those specs. If the implementation violates a spec, the guards catch it immediately. The PM's intent is not lost in translation; it's encoded in a form the machine can check.

This changes what PMs are responsible for: not describing the feature in prose, but specifying the behaviour precisely. That's harder in some ways and easier in others. You have to be exact. But being exact means you discover ambiguities before they cost a sprint.

## The four pillars

1. **Specify** — Write down what the system must do, in a structured form both humans and machines can read.
2. **Validate** — Make the spec executable. Run checks against the live system automatically.
3. **Restrict** — Limit what the AI is allowed to touch. It may only modify the implementation; everything else is generated or spec-owned.
4. **Build in blocks** — Keep the system composed of small, independently specifiable units. The smaller the block, the smaller the blast radius if something goes wrong.

## What it looks like in practice

The spec stack for even a simple API has multiple layers:

- An OpenAPI contract describes the shape
- TypeScript types and Zod schemas are generated from the contract
- Gherkin scenarios describe the behaviour
- Arazzo workflows describe multi-step flows
- Schemathesis auto-generates edge case inputs

Each layer guards something different. Together they form a net that catches most classes of breakage automatically.

The AI only writes the handler — the business logic. Everything else is generated, owned by the spec, or owned by the PM.

That's the whole system.
