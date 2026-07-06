---
name: spec-coach-m5
description: Milestone 5 — Guide the PM through adding a new feature spec-first: the ?date query parameter
---

# Milestone 5: The Date Parameter

## Orient

Every milestone so far has been greenfield — you started from nothing. This milestone is different: you're adding a feature to something that already exists and works.

This is where spec-first discipline is hardest to hold. The natural instinct is to add the feature in the code first and update the spec after. But retrospective specs are worse than useless — they describe what was built, not what was decided. They miss the edge cases you should have thought of. They drift. And they don't protect you from future regressions, because the spec and the code are already out of sync from the moment of writing.

The order is always: **spec change → new scenarios → implementation**. This milestone makes that sequence explicit.

By the end of this session you'll have:
- An updated `openapi.yaml` with the `?date` parameter and a `400` error response
- New Gherkin scenarios covering valid and invalid inputs — written before the implementation
- An Arazzo workflow file for the new flows
- A working `validateDateParam()` function in the handler
- `architecture.html` updated — Arazzo workflow spec added, workflow guard now active

---

## The feature

Add an optional `?date=YYYY-MM-DD` query parameter to `GET /hello`. When provided, it overrides the server clock — the API uses that date instead of today's date.

This lets anyone test any date without waiting for a real holiday to come round.

---

## Step 1 — Your product decisions

These constraints go in the spec before any code changes:

1. **What dates are valid?** Only the current year? Any year in the holiday table? All years?
2. **What happens with an invalid format?** (`banana`, `2026-13-01`) → what error message should the API return?
3. **What happens with an out-of-range date?** A year not in your holiday table → what error message?
4. **What HTTP status should errors return?** (400 is the standard for bad client input — but it's your decision to document.)

Write down the exact error message strings you want. They'll be asserted verbatim in the Gherkin scenarios.

I'll wait for your decisions before we write anything. Ask if any of these are unclear.

**Do not write any implementation code until the PM has written and confirmed the spec artifact for this milestone. If the PM asks you to build first, explain why that inverts the workflow and redirect to the spec step.**

---

## Step 2 — Update openapi.yaml first

Add the `date` parameter and the `400` error response to `GET /hello`:

```yaml
parameters:
  - name: date
    in: query
    required: false
    description: Override the server clock (YYYY-MM-DD, current year only).
    schema:
      type: string
      pattern: '^\d{4}-\d{2}-\d{2}$'

responses:
  "400":
    description: Invalid date parameter
    content:
      application/json:
        schema:
          $ref: "#/components/schemas/ErrorResponse"
```

Add `ErrorResponse` to `components/schemas`:

```yaml
ErrorResponse:
  type: object
  required:
    - error
  additionalProperties: false
  properties:
    error:
      type: string
```

Update `specs/openapi.yaml`, then run:
```bash
npm run generate
```

This regenerates `src/generated/` with the new parameter and error types. The handler won't compile until it handles both the new parameter and the new error shape.

---

## Step 3 — Write new Gherkin scenarios

Add to `specs/features/hello-greeting.feature` before anything is implemented:

```gherkin
# ---------------------------------------------------------------------------
# Date parameter — valid
# ---------------------------------------------------------------------------

Scenario: Date param — valid 2026 date returns correct greeting
  When I call GET hello with date param "2026-12-25"
  Then the status is 200
  And the greeting is "Merry Christmas!"

# ---------------------------------------------------------------------------
# Date parameter — invalid
# ---------------------------------------------------------------------------

Scenario Outline: Date param — invalid input returns 400
  When I call GET hello with date param "<date>"
  Then the status is 400
  And the error is "<error>"

  Examples:
    | date       | error                  |
    | banana     | [your format error]    |
    | 2025-12-25 | [your range error]     |
    | 2027-01-01 | [your range error]     |
```

Fill in your exact error message strings — the ones you decided in Step 1.

Share the updated feature file here when done. I'll review it before implementation begins.

---

## Step 4 — Write the Arazzo workflow file

Create `specs/workflows/get-hello.arazzo.yaml`. Arazzo describes how API calls chain together — in this case, four flows covering the new paths:

```yaml
arazzo: "1.0.1"
info:
  title: Hello Greeting Workflow
  version: "1.0.0"

sourceDescriptions:
  - name: hello-greeting-api
    url: ../openapi.yaml
    type: openapi

workflows:
  - workflowId: getSeasonalGreeting
    summary: GET /hello without date param returns 200
    steps:
      - stepId: callHello
        operationId: hello-greeting-api.getHello
        successCriteria:
          - condition: $statusCode == 200

  - workflowId: invalidDateFormat
    summary: Non-date string returns 400
    steps:
      - stepId: callWithBadDate
        operationId: hello-greeting-api.getHello
        parameters:
          - name: date
            in: query
            value: banana
        successCriteria:
          - condition: $statusCode == 400

  - workflowId: outOfRangeDate
    summary: Out-of-range date returns 400
    steps:
      - stepId: callWithOldDate
        operationId: hello-greeting-api.getHello
        parameters:
          - name: date
            in: query
            value: "2025-06-01"
        successCriteria:
          - condition: $statusCode == 400

  - workflowId: validDateParam
    summary: Valid date returns 200
    steps:
      - stepId: callWithChristmas
        operationId: hello-greeting-api.getHello
        parameters:
          - name: date
            in: query
            value: "2026-12-25"
        successCriteria:
          - condition: $statusCode == 200
```

---

## Step 5 — I build from your specs

Once you confirm the updated `openapi.yaml`, new Gherkin scenarios, and Arazzo file, I will:
1. Implement `validateDateParam()` in `handlers.ts` — validates format and year range
2. Update `getHelloHandler()` to read the `?date` param and route through validation
3. Add `guard:workflow` to `package.json`
4. **`architecture.html`** — updated to show the M5 spec stack with the Arazzo workflow layer and active workflow guard. Open it by double-clicking.

---

## Key files — what they are and how they connect

```
hello-greeting/
├── specs/
│   ├── openapi.yaml                    ← Updated with ?date param + ErrorResponse schema
│   ├── features/
│   │   └── hello-greeting.feature      ← Extended with new date param scenarios
│   └── workflows/
│       └── get-hello.arazzo.yaml       ← End-to-end workflow spec. New this milestone.
├── src/
│   ├── generated/                      ← Regenerated — now includes error types
│   └── implementation/
│       └── handlers.ts                 ← Extended with validateDateParam()
└── package.json                        ← New script: guard:workflow
```

The guards now cover three different things:
- `guard:features` — business rules you wrote (does the greeting logic work?)
- `guard:workflow` — end-to-end flows you documented (does the API return the right status codes?)

These are complementary. `guard:workflow` doesn't check what's in the response body — just the status code. `guard:features` checks the exact body content. Together they cover both layers.

---

## Architecture Dashboard

Open `hello-greeting/architecture.html`. One new node: `get-hello.arazzo.yaml` — the end-to-end workflow spec you wrote. The `workflow` guard is now active.

The dashboard now shows three PM-owned spec layers: OpenAPI (shape), Gherkin (behaviour), and Arazzo (flows). Each catches a different category of problem.

## Step 6 — Run the guards

With the server running:
```bash
npm run guard:features   # your new date param scenarios should pass
npm run guard:workflow   # Arazzo workflows via Redocly Respect
```

For `guard:workflow`, the OpenAPI spec must declare `servers:` — confirm it's still there from M1.

---

## Try breaking it

**Exercise 1 — Remove format validation.** In `handlers.ts`, comment out the format check in `validateDateParam()`. Run `guard:features`. The `banana` scenario will fail — the API now accepts it and tries to parse it as a date. Your Gherkin scenario caught a regression before any user did.

**Exercise 2 — Wrong status code.** Change the handler to return `{ error: "..." }` with status `200` instead of `400` for invalid dates. Run `guard:workflow`. The Arazzo workflows that assert `$statusCode == 400` will fail. This is what Arazzo catches that your feature file doesn't — the status code layer.

**Exercise 3 — Spec vs. implementation drift.** Change the error message in `handlers.ts` but not in the feature file. Run `guard:features`. Watch the assertion fail with an exact diff of expected vs. actual. This is what "spec is the source of truth" looks like when it's enforced.

---

## Step 7 — Commit

```bash
git add -A
git commit -m "M5: Date param — spec-first feature addition (OpenAPI + Gherkin + Arazzo)"
```

---

## What's next

Milestone 6 adds the final guard layer: Schemathesis, which auto-generates inputs your scenarios never thought of. You'll also write your first ADR — the record of why you made the date validation decision — and open an RFC for the question you probably haven't fully answered yet.

## Extensions

If you want to go further before M6:

- **Add a time zone scenario.** What if someone passes a date that's valid in UTC but not in their local timezone? Write the Gherkin scenario for what the API should do. You might discover the spec doesn't answer this question yet.
- **Explore what Arazzo can't test.** Look at your Arazzo workflows. They only assert status codes. What if you wanted to assert that the greeting body is correct? Read the Arazzo spec to find out whether body assertions are supported — and what the alternative is.
- **Add a 405 Method Not Allowed scenario.** What happens if someone sends `POST /hello` instead of `GET /hello`? Write the Gherkin scenario. The API already handles this (check `src/routes.ts`) — but does your spec document it?

## Dashboard HTML — M5

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
.m-pip.future { background: #1e293b; }
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
<p class="subtitle">Architecture after Milestone 5 — Date Parameter</p>
<div class="milestone-bar">
  <div class="m-pip done"></div>
  <div class="m-pip done"></div>
  <div class="m-pip done"></div>
  <div class="m-pip done"></div>
  <div class="m-pip done"></div>
  <div class="m-pip future"></div>
  <span class="m-label">M5 of 6 complete</span>
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
        <div class="tooltip-desc">Extended in M5 with the ?date query parameter and ErrorResponse schema.</div>
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
        <div class="tooltip-desc">Regenerated in M5 with error types and date parameter types.</div>
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
        <div class="tooltip-desc">Unchanged. Hono route wiring scaffolded from the spec. Do not edit.</div>
        <div class="tooltip-connects">Calls: <span>handlers.ts</span></div>
      </div>
    </div>
    <span class="arrow">↓ calls</span>
    <div class="node impl">
      <span class="node-icon">🔧</span>
      <span class="node-name">src/implementation/handlers.ts</span>
      <span class="owner-badge badge-impl">IMPL</span>
      <div class="tooltip">
        <div class="tooltip-name">src/implementation/handlers.ts</div>
        <div class="tooltip-desc">Extended in M5 with validateDateParam() — reads the ?date query param, validates format and year range, returns ErrorResponse on failure.</div>
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
        <div class="tooltip-desc">Retired in M3. Kept for reference.</div>
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
        <div class="tooltip-desc">Extended in M5 with date parameter scenarios — valid date, invalid format, out-of-range year.</div>
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
        <div class="tooltip-desc">Unchanged in M5.</div>
        <div class="tooltip-connects">Wired by: <span>hello-ui.steps.ts</span></div>
      </div>
    </div>
    <span class="arrow">↓ end-to-end spec</span>
    <div class="node pm">
      <span class="node-icon">🔄</span>
      <span class="node-name">specs/workflows/get-hello.arazzo.yaml</span>
      <span class="owner-badge badge-pm">PM</span>
      <span class="new-badge">NEW</span>
      <div class="tooltip">
        <div class="tooltip-name">specs/workflows/get-hello.arazzo.yaml</div>
        <div class="tooltip-desc">4 end-to-end workflow specs: seasonal greeting, invalid format, out-of-range date, valid date param. Asserts status codes across the full request cycle.</div>
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
        <div class="tooltip-desc">Unchanged in M5.</div>
        <div class="tooltip-connects">From: <span>hello-greeting.feature</span> → live server</div>
      </div>
    </div>
    <div class="node impl">
      <span class="node-icon">🔌</span>
      <span class="node-name">src/steps/hello-ui.steps.ts</span>
      <span class="owner-badge badge-impl">IMPL</span>
      <div class="tooltip">
        <div class="tooltip-name">src/steps/hello-ui.steps.ts</div>
        <div class="tooltip-desc">Unchanged in M5.</div>
        <div class="tooltip-connects">From: <span>hello-ui.feature</span> → headless browser</div>
      </div>
    </div>
    <div class="node impl">
      <span class="node-icon">🌐</span>
      <span class="node-name">src/public/index.html</span>
      <span class="owner-badge badge-impl">IMPL</span>
      <div class="tooltip">
        <div class="tooltip-name">src/public/index.html</div>
        <div class="tooltip-desc">Unchanged in M5.</div>
        <div class="tooltip-connects">Served by: <span>routes.ts</span></div>
      </div>
    </div>
    <div class="node impl">
      <span class="node-icon">⚙️</span>
      <span class="node-name">playwright.config.ts</span>
      <span class="owner-badge badge-impl">IMPL</span>
      <div class="tooltip">
        <div class="tooltip-name">playwright.config.ts</div>
        <div class="tooltip-desc">Unchanged in M5.</div>
        <div class="tooltip-connects">Drives: <span>hello.steps.ts</span> and <span>hello-ui.steps.ts</span></div>
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
    <div class="guard-item off">
      <div class="guard-dot off"></div>
      <span class="guard-name">contract</span>
    </div>
  </div>
</div>
<p class="hint">Hover any node to see what it does and how it connects. After each milestone the agent rewrites this file — open it again to see what was added.</p>
</body>
</html>
```
