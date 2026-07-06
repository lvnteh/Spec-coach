---
name: spec-coach-m6
description: Milestone 6 — Add Schemathesis contract guard and write ADR + RFC to document decisions
---

# Milestone 6: Contract Guard + Decisions

## Orient

You've built the full application. Every layer of the spec stack is in place except two: the contract guard and the decision trail.

This milestone adds both. It's also where you step back and ask: what decisions did we make that future-you — or a new team member six months from now — needs to understand?

By the end of this session you'll have:
- Schemathesis running as `guard:contract` — testing inputs your scenarios never imagined
- `docs/decisions/ADR-001.md` — recording the date validation decision with context and consequences
- `docs/rfcs/RFC-001.md` — opening the question you probably haven't fully answered yet
- `docs/status.md` — the project health snapshot

---

## What is Schemathesis, and why does it matter?

Your Gherkin scenarios test what you thought of. Schemathesis tests what you didn't.

It reads your `openapi.yaml` and auto-generates inputs:
- Arrays where the spec says string (`?date=a&date=b`)
- Empty strings
- Very long strings
- Boundary integers
- Every combination it can derive from your schema

It fires these at the live server and checks that every response matches the schema you declared. If your endpoint returns a Zod error object when it should return your `ErrorResponse` shape, Schemathesis catches it. Your Gherkin scenarios wouldn't — you only wrote inputs you expected.

The division of labour across all four guards:

| Guard | Tool | Catches |
|---|---|---|
| `guard:typecheck` | TypeScript | Type mismatches between spec and handler |
| `guard:features` | Playwright BDD | Business rules and HTTP-layer behaviour (you write these) |
| `guard:workflow` | Arazzo/Redocly | End-to-end status codes |
| `guard:contract` | Schemathesis | Protocol conformance, edge case inputs (auto-generated) |

These are complementary. No single guard catches everything.

Reference: [guard:contract setup at M6](https://github.com/lvnteh/specdrivengreeting/blob/m6-contract/package.json)

---

## Step 1 — Add guard:contract

Schemathesis runs via Docker — no local Python install needed.

Add to `package.json`:

```json
"guard:contract": "docker run --rm -v \"$(pwd)/specs:/specs\" schemathesis/schemathesis run --url http://host.docker.internal:3010 --phases examples,coverage /specs/openapi.yaml",
"guard": "npm run guard:typecheck && npm run guard:features && npm run guard:contract && npm run guard:workflow"
```

With the server running:
```bash
npm run guard:contract
```

What passing looks like:
```
Schemathesis test session starts
OpenAPI: http://host.docker.internal:3010
Loaded: 1 endpoint, 2 status codes
1. GET /hello
   [200, 400] 45 test cases passed
```

What a failure looks like:
```
1. GET /hello
   FAILED: Response violates schema
   status: 400
   body: {"success":false,"error":{"issues":[...]}}  ← Zod error object, not ErrorResponse
```

If Schemathesis finds a failure, it tells you exactly what input triggered it and what the actual response was. Fix the handler until it passes.

**Do not write any implementation code until the PM has written and confirmed the spec artifact for this milestone. If the PM asks you to build first, explain why that inverts the workflow and redirect to the spec step.**

---

## Try breaking it first

Before writing your documentation, run a deliberate experiment to understand what Schemathesis finds that you didn't.

**Exercise:** In `handlers.ts`, temporarily comment out the route-level Zod transform that normalises the `?date` array parameter (if present). Then run `guard:contract`. Schemathesis will send `?date=a&date=b` — an array, not a string — and the response will be a Zod validation error object instead of your `ErrorResponse` shape. Schemathesis will catch it; your Gherkin scenarios won't, because you never wrote a scenario for array parameters.

This is the lesson: your scenarios cover what you thought of. Schemathesis covers what the protocol allows that you didn't think of. Both are needed.

Put the code back and confirm the guard passes before continuing.

---

## Step 2 — Your task: write ADR-001

The system has no memory. Files capture current state. They don't capture *why*.

You made a significant decision in M5: which years does `?date` accept, and why? That decision:
- Changed what inputs the API accepts
- Changed the error messages users see
- Determined whether 2025 holiday data is reachable via HTTP at all

Without a record, the next person to touch `validateDateParam()` — or a future you in six months — has no idea why the rule is what it is. They might change it thinking it was arbitrary.

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
[What is now true as a result of this decision?]
[What do you need to watch for in the future?]
[What becomes harder or easier because of this choice?]
```

The **Consequences** section is the most valuable part over time. It's how future-you answers "why is it like this?" without archaeology.

Reference: [ADR-001 from specdrivengreeting](https://github.com/lvnteh/specdrivengreeting/blob/m6-contract/docs/decisions/ADR-001-feature-scenarios-2026-only.md)

Share your draft and I'll give feedback before you finalise it.

---

## Step 3 — Your task: open RFC-001

An ADR captures a closed decision. An RFC captures an open question — something you haven't resolved yet.

The year validation decision probably raised a follow-on question: should the rule ever be expanded? What happens in January 2027 when the year rolls over? Who maintains the holiday table?

Create `docs/rfcs/RFC-001-[topic].md`:

```markdown
# RFC-001: [title of the open question]

**Status:** Open
**Date:** [today]
**Author:** [your name]
**Review deadline:** [two weeks from today]

## Problem
[What asymmetry or gap does this question address?
What happens if we do nothing?]

## Proposal
[What would the preferred option look like if you acted on it?]

## Open questions
[What do you not yet know?]

## Options
### Option A — [proposed change]
Pros: ...
Cons: ...

### Option B — [keep current behaviour]
Pros: ...
Cons: ...

### Option C — [alternative]
Pros: ...
Cons: ...

## Resolution
*(To be filled when this RFC closes)*
Decision taken: —
Recorded in: ADR-00X
```

An RFC without a review deadline is just a note. Give it a date and own it.

---

## Step 4 — Your task: write status.md

Create `docs/status.md` — a snapshot of where the project stands:

```markdown
# Project Status: Hello Greeting

**Last updated:** [today]
**Phase:** Stabilisation
**Health:** Green

## What's done
- M1: OpenAPI contract + Hello World server
- M2: Holiday table + cases.json
- M3: Gherkin feature file (promotes cases.json to HTTP-layer BDD)
- M4: UI feature file + HTML page
- M5: Date parameter spec-first (OpenAPI + Gherkin + Arazzo)
- M6: Contract guard (Schemathesis) + ADR + RFC

## What's in flight
- RFC-001: [your open question]

## Known risks
[What could break this if left unaddressed?]

## Spec integrity
| Spec | Status | Notes |
|------|--------|-------|
| openapi.yaml | In sync | ... |
| hello-greeting.feature | In sync | ... |
| hello-greeting-ui.feature | In sync | ... |
| get-hello.arazzo.yaml | In sync | ... |
| src/generated/ | In sync | Regenerate after openapi.yaml changes |
```

---

## Key files — the complete picture

```
hello-greeting/
├── specs/
│   ├── openapi.yaml                        ← Single source of truth. All changes start here.
│   ├── features/
│   │   ├── hello-greeting.feature          ← API behaviour (PM-owned)
│   │   └── hello-greeting-ui.feature       ← UI behaviour (PM-owned)
│   └── workflows/
│       └── get-hello.arazzo.yaml           ← End-to-end flow spec
├── src/
│   ├── generated/                          ← Derived from openapi.yaml. Never edit.
│   ├── routes.ts                           ← Derived from openapi.yaml. Never edit.
│   ├── app.ts                              ← Framework wiring.
│   ├── public/index.html                   ← UI.
│   ├── steps/
│   │   ├── hello.steps.ts                  ← Wires API Gherkin → HTTP
│   │   └── hello-ui.steps.ts               ← Wires UI Gherkin → browser
│   └── implementation/
│       └── handlers.ts                     ← The ONLY file with business logic.
├── docs/
│   ├── decisions/ADR-001-*.md              ← Why decisions were made
│   ├── rfcs/RFC-001-*.md                   ← Open questions
│   └── status.md                           ← Project health snapshot
└── package.json                            ← All four guards wired into guard:*
```

Every spec points at something it protects. Every implementation file is either generated (never touch) or implementation (only touch `handlers.ts`). The decision trail explains why the specs look the way they do.

---

## Step 5 — Run the full guard suite

```bash
npm run guard
```

This runs all four guards in sequence: typecheck → features → contract → workflow. If all four pass, every layer of the spec is consistent with the implementation.

What full green looks like:
```
✓ guard:typecheck — no errors
✓ guard:features — 40 passed
✓ guard:contract — 45 cases passed
✓ guard:workflow — 4 workflows passed
```

---

## Step 6 — Commit

```bash
git add -A
git commit -m "M6: Contract guard (Schemathesis) + ADR-001 + RFC-001 + status.md"
```

---

## You're done

You've built a spec-driven API from scratch:
- An OpenAPI contract as the single source of truth
- TypeScript types and Zod schemas generated from the spec — never hand-written
- A PM-owned Gherkin suite testing the real HTTP server
- A UI with its own behaviour spec
- A new feature added spec-first — contract before code
- Arazzo workflows for end-to-end status checks
- Schemathesis for protocol conformance and edge cases you didn't think of
- Decision records that explain the *why*, not just the *what*

The mental model: **spec is the decision, code is the consequence, guards enforce the contract**. This applies to every product you'll work on — not just APIs.


## What could come next

You've completed the core journey. Here are directions worth exploring:

- **Add 2027 holiday data.** Update `handlers.ts` with next year's dates. Your Gherkin scenarios will tell you exactly which dates to add — the feature file is your acceptance checklist.
- **Resolve RFC-001.** Take the open question you raised and close it with a decision. Write ADR-002 recording what you decided and why.
- **Apply this to your own product.** Pick a feature you're working on at work. Write the OpenAPI contract for it before the sprint starts. Hand it to a developer. See what questions come up that your PRD didn't answer.
- **Add the `fuzzing` phase to Schemathesis.** Change `--phases examples,coverage` to `--phases examples,coverage,fuzzing`. Run it. This enables random input generation. Note how many more test cases it runs and what it finds.
- **Contribute a milestone.** Is there a layer of the spec stack not covered here — database schemas, event contracts, GraphQL? The same four-pillar structure (specify, validate, restrict, build in blocks) applies. Design M7.
