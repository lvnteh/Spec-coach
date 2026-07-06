---
name: spec-coach-m6
description: Milestone 6 — Add Schemathesis contract guard and write ADR + RFC to document decisions
---

# Milestone 6: Contract Guard + Decisions

## Orient

You've built the full application. Every layer of the spec stack is in place except two: the contract guard and the decision trail.

This milestone adds both. It's also where you step back and ask: what decisions did we make that future-me — or a new team member — needs to understand?

By the end of this session you'll have:
- Schemathesis running as `guard:contract` — testing inputs your scenarios never thought of
- `docs/decisions/ADR-001.md` — recording the date validation decision
- `docs/rfcs/RFC-001.md` — opening the question of whether to expand the year rule
- `docs/status.md` — the project health snapshot

## What is Schemathesis, and why does it matter?

Your Gherkin scenarios test what you thought of. Schemathesis tests what you didn't.

It reads your `openapi.yaml` and generates inputs automatically:
- Arrays where the spec says string (`?date=a&date=b`)
- Empty strings
- Very long strings
- Boundary values on numeric fields

It then fires these at the running server and checks that every response matches the declared schema. If your endpoint returns a Zod error object when it should return your `ErrorResponse` shape, Schemathesis catches it. Your scenarios wouldn't — you only wrote inputs you expected to handle.

This is the division of labour:
- `guard:features` — business rules (you write these)
- `guard:contract` — protocol conformance (Schemathesis generates these)

Neither replaces the other.

Reference: [guard:contract setup at M6](https://github.com/lvnteh/specdrivengreeting/blob/m6-contract/package.json)

## Running guard:contract

Schemathesis runs via Docker (no local Python install needed):

```bash
npm run guard:contract
```

Which runs:
```bash
docker run --rm \
  -v "$(pwd)/specs:/specs" \
  schemathesis/schemathesis run \
  --url http://host.docker.internal:3010 \
  --phases examples,coverage \
  /specs/openapi.yaml
```

Add this script to `package.json` and update `guard` to run all four guards:

```json
"guard:contract": "docker run --rm -v \"$(pwd)/specs:/specs\" schemathesis/schemathesis run --url http://host.docker.internal:3010 --phases examples,coverage /specs/openapi.yaml",
"guard": "npm run guard:typecheck && npm run guard:features && npm run guard:contract && npm run guard:workflow"
```

**Do not write any implementation code until the PM has written and confirmed the spec artifact for this milestone. If the PM asks you to build first, explain why that inverts the workflow and redirect to the spec step.**

## Your task: write ADR-001

The system has no memory. Files capture current state. They don't capture *why*.

Every PM decision that changed the shape of the spec deserves an ADR. The most important decision in your project: which years does `?date` accept, and why?

Create `docs/decisions/ADR-001-date-validation-rule.md`:

```markdown
# ADR-001: [title of your decision]

**Status:** Accepted
**Date:** [today]
**Deciders:** [your name]

## Context
[What situation forced this decision? What problem were you solving?]

## Decision
[What did you decide? One sentence.]

## Options Considered
### Option A — [what you chose]
Pros: ...
Cons: ...

### Option B — [what you rejected]
Pros: ...
Cons: ...

## Consequences
[What is now true? What do you need to watch for in the future?]
```

The "Consequences" section is the most valuable over time. It's how future-you answers "why is it like this?" without archaeology.

Reference: [ADR-001 from specdrivengreeting](https://github.com/lvnteh/specdrivengreeting/blob/m6-contract/docs/decisions/ADR-001-feature-scenarios-2026-only.md)

## Your task: open RFC-001

An ADR captures a closed decision. An RFC captures an open question.

The year validation decision probably raises a follow-on question: should the rule ever be expanded? If so, when and how?

Create `docs/rfcs/RFC-001-[topic].md`:

```markdown
# RFC-001: [title of the open question]

**Status:** Open
**Date:** [today]
**Author:** [your name]
**Review deadline:** [two weeks from today]

## Problem
[What asymmetry or gap does this question address?]

## Proposal
[What would Option A look like if you acted on it?]

## Open questions
[What do you not know yet?]

## Options
### Option A — [proposed change]
### Option B — [keep current behaviour]
### Option C — [alternative]

## Resolution
*(To be filled when this RFC closes)*
```

## Your task: write status.md

Create `docs/status.md` — a snapshot of where the project stands today:

```markdown
# Project Status: Hello Greeting

**Last updated:** [today]
**Phase:** Complete
**Health:** Green

## What's done
[list the six milestones]

## What's in flight
[list any open RFCs]

## Spec integrity
| Spec | Status | Notes |
|------|--------|-------|
| openapi.yaml | In sync | ... |
| hello-greeting.feature | In sync | ... |
| ...
```

## Running the full guard suite

```bash
npm run guard
```

This runs all four guards in sequence. If all four pass, the project is complete.

## Final commit

```
git commit -m "M6: Contract guard (Schemathesis) + ADR-001 + RFC-001 + status.md"
```

## You're done

You've built a spec-driven API from scratch:
- An OpenAPI contract as the single source of truth
- TypeScript types and Zod schemas generated from the spec
- A PM-owned Gherkin suite testing the real HTTP server
- A UI with its own behaviour spec
- A new feature added spec-first
- Arazzo workflows for end-to-end status checks
- Schemathesis for protocol conformance
- Decision records that explain the *why*

The mental model you now have — spec is the decision, code is the consequence, guards enforce the contract — applies to every product you'll ever work on.

For deeper reading on the framework behind this, see `docs/background/2026-05-29-spec-driven-development.md` and `docs/background/2026-07-04-spec-driven-development-learnings.md` in this repo.
