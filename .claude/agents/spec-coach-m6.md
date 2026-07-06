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
- `architecture.html` completed — the full spec stack visible with all guards active and the decision trail documented

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

I will also write `architecture.html` with the complete M6 view — all nodes, all guards, the decision trail.

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

After writing `docs/status.md`, ask me to update `architecture.html` and I will write the final M6 version to `hello-greeting/architecture.html`.

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

## Architecture Dashboard

Open `hello-greeting/architecture.html`. The complete picture is now visible:

- Four PM-owned spec layers: OpenAPI (shape), Gherkin (behaviour), Arazzo (flows), and the decision trail (ADR/RFC/status)
- Four active guards covering every layer
- The full derivation chain from your spec to generated code to implementation

The decision trail at the bottom — ADR, RFC, status.md — is what separates a project you can hand to someone else from a project only you understand.

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

---

## Dashboard HTML — M6

Write this file to `hello-greeting/architecture.html`:

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Spec-Driven Greeting — Architecture</title>
<style>
* { box-sizing: border-box; margin: 0; padding: 0; }
body { background: #0f1117; color: #e2e8f0; font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif; padding: 32px; max-width: 860px; }
h1 { font-size: 16px; color: #f1f5f9; margin-bottom: 4px; }
.subtitle { font-size: 12px; color: #64748b; margin-bottom: 24px; }
.milestone-bar { display: flex; gap: 6px; align-items: center; margin-bottom: 20px; }
.m-pip { width: 32px; height: 5px; border-radius: 3px; }
.m-pip.done { background: #38bdf8; }
.m-label { font-size: 11px; color: #475569; margin-left: 6px; }
.dashboard { display: flex; gap: 0; border: 1px solid #1e293b; border-radius: 10px; }
.spec-panel { flex: 1; padding: 20px; border-right: 1px solid #1e293b; border-radius: 10px 0 0 10px; overflow: visible; }
.guard-panel { width: 176px; padding: 20px; flex-shrink: 0; border-radius: 0 10px 10px 0; }
.panel-title { font-size: 10px; letter-spacing: 1px; color: #64748b; text-transform: uppercase; margin-bottom: 14px; }
.node { position: relative; display: flex; align-items: center; gap: 8px; padding: 8px 10px; border-radius: 6px; margin-bottom: 6px; cursor: default; border: 1px solid transparent; }
.node:hover .tooltip { display: block; }
.node.pm { background: #172554; border-color: #1e3a8a; }
.node.gen { background: #14532d; border-color: #166534; }
.node.impl { background: #1c1917; border: 1px solid #292524; }
.node.dim { opacity: 0.3; }
.node-icon { font-size: 13px; flex-shrink: 0; }
.node-name { font-size: 12px; font-weight: 500; flex: 1; }
.node.pm .node-name { color: #93c5fd; }
.node.gen .node-name { color: #86efac; }
.node.impl .node-name { color: #d6d3d1; }
.owner-badge { font-size: 9px; padding: 2px 6px; border-radius: 3px; font-weight: 600; letter-spacing: 0.5px; flex-shrink: 0; }
.badge-pm { background: #1e40af; color: #bfdbfe; }
.badge-gen { background: #166534; color: #bbf7d0; }
.badge-impl { background: #292524; color: #a8a29e; }
.new-badge { font-size: 9px; padding: 1px 5px; border-radius: 3px; background: #7c3aed; color: #ede9fe; font-weight: 600; flex-shrink: 0; }
.retired-badge { font-size: 9px; padding: 1px 5px; border-radius: 3px; background: #374151; color: #9ca3af; font-weight: 600; flex-shrink: 0; }
.section-divider { margin: 10px 0 12px; border-top: 1px solid #1e293b; padding-top: 12px; }
.arrow { color: #334155; font-size: 11px; margin: 0 4px 6px 20px; display: block; }
.tooltip { display: none; position: absolute; left: 100%; top: 50%; transform: translateY(-50%); margin-left: 10px; background: #1e293b; border: 1px solid #334155; border-radius: 8px; padding: 10px 12px; width: 210px; z-index: 10; box-shadow: 0 8px 24px rgba(0,0,0,0.5); pointer-events: none; }
.tooltip-name { font-size: 11px; font-weight: 600; color: #f1f5f9; margin-bottom: 4px; }
.tooltip-desc { font-size: 10px; color: #94a3b8; line-height: 1.5; margin-bottom: 6px; }
.tooltip-connects { font-size: 10px; color: #64748b; }
.tooltip-connects span { color: #818cf8; }
.guard-item { display: flex; align-items: center; gap: 8px; padding: 7px 8px; border-radius: 6px; margin-bottom: 5px; background: #0f1117; border: 1px solid #1e293b; }
.guard-item.pass { border-color: #166534; background: #052e16; }
.guard-item.off { opacity: 0.35; }
.guard-dot { width: 7px; height: 7px; border-radius: 50%; flex-shrink: 0; }
.guard-dot.pass { background: #4ade80; }
.guard-dot.off { background: #334155; }
.guard-name { font-size: 11px; }
.guard-item.pass .guard-name { color: #86efac; }
.guard-item.off .guard-name { color: #475569; }
.hint { margin-top: 16px; font-size: 11px; color: #334155; }
</style>
</head>
<body>
<h1>Spec-Driven Greeting</h1>
<p class="subtitle">Architecture after Milestone 6 — Complete</p>
<div class="milestone-bar">
  <div class="m-pip done"></div>
  <div class="m-pip done"></div>
  <div class="m-pip done"></div>
  <div class="m-pip done"></div>
  <div class="m-pip done"></div>
  <div class="m-pip done"></div>
  <span class="m-label">All 6 milestones complete</span>
</div>
<div class="dashboard">
  <div class="spec-panel">
    <div class="panel-title">Spec Stack</div>
    <div class="node pm">
      <span class="node-icon">📄</span>
      <span class="node-name">specs/openapi.yaml</span>
      <span class="owner-badge badge-pm">PM</span>
      <div class="tooltip">
        <div class="tooltip-name">specs/openapi.yaml</div>
        <div class="tooltip-desc">Your contract. Defines the shape of every request and response, including the ?date parameter and ErrorResponse schema.</div>
        <div class="tooltip-connects">Connects to: <span>src/generated/</span></div>
      </div>
    </div>
    <span class="arrow">↓ generates</span>
    <div class="node gen">
      <span class="node-icon">⚙️</span>
      <span class="node-name">src/generated/</span>
      <span class="owner-badge badge-gen">GEN</span>
      <div class="tooltip">
        <div class="tooltip-name">src/generated/</div>
        <div class="tooltip-desc">TypeScript types and Zod schemas generated by Kubb. Regenerate after any spec change with npm run generate.</div>
        <div class="tooltip-connects">From: <span>openapi.yaml</span> · Used by: <span>routes.ts</span></div>
      </div>
    </div>
    <span class="arrow">↓ consumed by</span>
    <div class="node impl">
      <span class="node-icon">🛣️</span>
      <span class="node-name">src/routes.ts</span>
      <span class="owner-badge badge-impl">IMPL</span>
      <div class="tooltip">
        <div class="tooltip-name">src/routes.ts</div>
        <div class="tooltip-desc">Hono route wiring scaffolded from the spec. Connects GET /hello and GET / to handlers. Do not edit.</div>
        <div class="tooltip-connects">Calls: <span>handlers.ts</span> · Serves: <span>index.html</span></div>
      </div>
    </div>
    <span class="arrow">↓ calls</span>
    <div class="node impl">
      <span class="node-icon">🔧</span>
      <span class="node-name">src/implementation/handlers.ts</span>
      <span class="owner-badge badge-impl">IMPL</span>
      <div class="tooltip">
        <div class="tooltip-name">src/implementation/handlers.ts</div>
        <div class="tooltip-desc">All business logic: holiday table, findGreeting(), validateDateParam(). The only file you write yourself in this project.</div>
        <div class="tooltip-connects">Called by: <span>routes.ts</span></div>
      </div>
    </div>
    <span class="arrow">↓ retired</span>
    <div class="node pm dim">
      <span class="node-icon">📋</span>
      <span class="node-name">specs/cases.json</span>
      <span class="owner-badge badge-pm">PM</span>
      <span class="retired-badge">RETIRED</span>
      <div class="tooltip">
        <div class="tooltip-name">specs/cases.json</div>
        <div class="tooltip-desc">Retired in M3. Kept for reference — the Gherkin feature file replaced it as the executable spec.</div>
        <div class="tooltip-connects">Was executed by: <span>guard-cases.ts</span></div>
      </div>
    </div>
    <div class="node impl dim">
      <span class="node-icon">🏃</span>
      <span class="node-name">src/guard-cases.ts</span>
      <span class="owner-badge badge-impl">IMPL</span>
      <span class="retired-badge">RETIRED</span>
      <div class="tooltip">
        <div class="tooltip-name">src/guard-cases.ts</div>
        <div class="tooltip-desc">Retired in M3. Kept for reference.</div>
        <div class="tooltip-connects">Was called by: <span>guard:cases</span></div>
      </div>
    </div>
    <span class="arrow">↓ API tested by</span>
    <div class="node pm">
      <span class="node-icon">📝</span>
      <span class="node-name">specs/features/hello-greeting.feature</span>
      <span class="owner-badge badge-pm">PM</span>
      <div class="tooltip">
        <div class="tooltip-name">specs/features/hello-greeting.feature</div>
        <div class="tooltip-desc">Your API behaviour scenarios — 24 holidays, boundary, tie-break, fallback, and date parameter cases.</div>
        <div class="tooltip-connects">Wired by: <span>hello.steps.ts</span></div>
      </div>
    </div>
    <span class="arrow">↓ UI tested by</span>
    <div class="node pm">
      <span class="node-icon">🖥️</span>
      <span class="node-name">specs/features/hello-greeting-ui.feature</span>
      <span class="owner-badge badge-pm">PM</span>
      <div class="tooltip">
        <div class="tooltip-name">specs/features/hello-greeting-ui.feature</div>
        <div class="tooltip-desc">Your UI behaviour scenarios: page load, date picker, fallback, error toast.</div>
        <div class="tooltip-connects">Wired by: <span>hello-ui.steps.ts</span></div>
      </div>
    </div>
    <span class="arrow">↓ end-to-end spec</span>
    <div class="node pm">
      <span class="node-icon">🔄</span>
      <span class="node-name">specs/workflows/get-hello.arazzo.yaml</span>
      <span class="owner-badge badge-pm">PM</span>
      <div class="tooltip">
        <div class="tooltip-name">specs/workflows/get-hello.arazzo.yaml</div>
        <div class="tooltip-desc">4 end-to-end workflow specs covering all status code paths. Run by guard:workflow.</div>
        <div class="tooltip-connects">Run by: <span>guard:workflow</span></div>
      </div>
    </div>
    <span class="arrow">↓ wired by</span>
    <div class="node impl">
      <span class="node-icon">🔌</span>
      <span class="node-name">src/steps/hello.steps.ts</span>
      <span class="owner-badge badge-impl">IMPL</span>
      <div class="tooltip">
        <div class="tooltip-name">src/steps/hello.steps.ts</div>
        <div class="tooltip-desc">API Gherkin step definitions. Wires each line to an HTTP fetch call.</div>
        <div class="tooltip-connects">From: <span>hello-greeting.feature</span> → live server</div>
      </div>
    </div>
    <div class="node impl">
      <span class="node-icon">🔌</span>
      <span class="node-name">src/steps/hello-ui.steps.ts</span>
      <span class="owner-badge badge-impl">IMPL</span>
      <div class="tooltip">
        <div class="tooltip-name">src/steps/hello-ui.steps.ts</div>
        <div class="tooltip-desc">UI Gherkin step definitions. Wires each line to Playwright browser automation.</div>
        <div class="tooltip-connects">From: <span>hello-ui.feature</span> → headless browser</div>
      </div>
    </div>
    <div class="node impl">
      <span class="node-icon">🌐</span>
      <span class="node-name">src/public/index.html</span>
      <span class="owner-badge badge-impl">IMPL</span>
      <div class="tooltip">
        <div class="tooltip-name">src/public/index.html</div>
        <div class="tooltip-desc">The HTML UI. Two panels: "Right now" and "Pick a date".</div>
        <div class="tooltip-connects">Served by: <span>routes.ts</span></div>
      </div>
    </div>
    <div class="node impl">
      <span class="node-icon">⚙️</span>
      <span class="node-name">playwright.config.ts</span>
      <span class="owner-badge badge-impl">IMPL</span>
      <div class="tooltip">
        <div class="tooltip-name">playwright.config.ts</div>
        <div class="tooltip-desc">Tells Playwright where feature files and step definitions live.</div>
        <div class="tooltip-connects">Drives: <span>hello.steps.ts</span> and <span>hello-ui.steps.ts</span></div>
      </div>
    </div>
    <div class="section-divider">
      <div class="panel-title">Decision Trail</div>
    </div>
    <div class="node pm">
      <span class="node-icon">📋</span>
      <span class="node-name">docs/decisions/ADR-001.md</span>
      <span class="owner-badge badge-pm">PM</span>
      <span class="new-badge">NEW</span>
      <div class="tooltip">
        <div class="tooltip-name">docs/decisions/ADR-001.md</div>
        <div class="tooltip-desc">Records the date validation decision — what was decided, what options were considered, and what the consequences are.</div>
        <div class="tooltip-connects">Documents: <span>validateDateParam()</span> in handlers.ts</div>
      </div>
    </div>
    <div class="node pm">
      <span class="node-icon">❓</span>
      <span class="node-name">docs/rfcs/RFC-001.md</span>
      <span class="owner-badge badge-pm">PM</span>
      <span class="new-badge">NEW</span>
      <div class="tooltip">
        <div class="tooltip-name">docs/rfcs/RFC-001.md</div>
        <div class="tooltip-desc">Opens an unresolved question about future year support. Has a review deadline — it's not just a note.</div>
        <div class="tooltip-connects">Related to: <span>ADR-001.md</span></div>
      </div>
    </div>
    <div class="node pm">
      <span class="node-icon">📊</span>
      <span class="node-name">docs/status.md</span>
      <span class="owner-badge badge-pm">PM</span>
      <span class="new-badge">NEW</span>
      <div class="tooltip">
        <div class="tooltip-name">docs/status.md</div>
        <div class="tooltip-desc">Project health snapshot: what's done, what's in flight, spec integrity table. The document that tells a new team member where things stand.</div>
        <div class="tooltip-connects">References: <span>RFC-001.md</span></div>
      </div>
    </div>
  </div>
  <div class="guard-panel">
    <div class="panel-title">Guards</div>
    <div class="guard-item pass">
      <div class="guard-dot pass"></div>
      <span class="guard-name">typecheck</span>
    </div>
    <div class="guard-item off">
      <div class="guard-dot off"></div>
      <span class="guard-name">cases (retired)</span>
    </div>
    <div class="guard-item pass">
      <div class="guard-dot pass"></div>
      <span class="guard-name">features</span>
    </div>
    <div class="guard-item pass">
      <div class="guard-dot pass"></div>
      <span class="guard-name">workflow</span>
    </div>
    <div class="guard-item pass">
      <div class="guard-dot pass"></div>
      <span class="guard-name">contract</span>
    </div>
  </div>
</div>
<p class="hint">You built this. Every node you see was either written by you (blue) or derived from something you wrote. The spec is the decision — the code is the consequence.</p>
</body>
</html>
```
