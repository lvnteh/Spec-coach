---
name: spec-coach-m3
description: Milestone 3 — Guide the PM through promoting cases.json to a Gherkin feature file and understanding why it matters
---

# Milestone 3: Gherkin

## Orient

In M2 you wrote cases.json and the guard went green. Everything looked right.

This milestone shows you what cases.json was silently missing — and replaces it with something stronger. A Gherkin `.feature` file. Same scenarios, different format. The difference: Gherkin tests the real HTTP server, not just the functions inside it.

By the end of this session you'll have:
- `specs/features/hello-greeting.feature` with your scenarios in Gherkin
- A Playwright BDD runner that hits `http://localhost:3010` with real HTTP requests
- cases.json retired (kept as a reference, no longer the guard)
- `architecture.html` updated — cases.json dimmed as retired, three new nodes for the Gherkin layer

---

## Why Gherkin instead of cases.json?

Your cases.json called `findGreeting()` directly — a pure function. It never made an HTTP request. This matters because there are layers between "user sends a request" and "findGreeting() runs": the HTTP framework (Hono) and the route validator (Zod).

Here's a real example of what cases.json missed: if Zod silently intercepted a bad `?date` parameter and returned a Zod error object instead of your custom error message, cases.json would show 40/40 green. But the user in the browser would see a broken response. The guard was testing the right logic in the wrong place.

Gherkin + Playwright BDD sends real HTTP requests to the live server and reads real HTTP responses. That's the only layer that matters.

The other benefit: Gherkin is readable by anyone. A stakeholder, a support engineer, a new team member — anyone can open `hello-greeting.feature` and understand exactly what the system does, without reading code.

```gherkin
Scenario: Today is Christmas
  When I call GET hello with date "2026-12-25"
  Then the status is 200
  And the greeting is "Merry Christmas!"
  And the holiday is "Christmas"
  And daysUntil is 0
```

That's your cases.json entry, translated to Gherkin. The content is identical. The format changed. The runner changed. The coverage got stronger.

Reference: [What a finished feature file looks like at M3](https://github.com/lvnteh/specdrivengreeting/blob/m3-gherkin/specs/features/hello-greeting.feature) — look at it after you've written yours.

---

## The mapping from cases.json to Gherkin

| cases.json field | Gherkin equivalent |
|---|---|
| `"description"` | Scenario name |
| `"input"` | `When I call GET hello with date "..."` |
| `"expected"` | `Then` assertions |

For many similar scenarios (all your holidays), use `Scenario Outline` with an `Examples` table — one row per holiday, same assertions:

```gherkin
Scenario Outline: Today is a holiday
  When I call GET hello with date "<date>"
  Then the status is 200
  And the greeting is "<greeting>"
  And the holiday is "<holiday>"
  And daysUntil is 0

  Examples:
    | date       | greeting         | holiday   |
    | 2026-12-25 | Merry Christmas! | Christmas |
    | 2026-10-31 | Happy Halloween! | Halloween |
```

For structurally different cases (boundary, fallback, tie-break), use individual `Scenario` blocks — they tell a different story and deserve their own name.

---

## Step 1 — Your task: write the feature file

Translate your cases.json into `specs/features/hello-greeting.feature`.

As you write it, you'll notice cases that were awkward in JSON but natural in Gherkin:
- **Boundary cases**: exactly 10 days before a holiday (inside window) vs. 11 days (outside). These are worth adding as explicit scenarios — they pin the exact edge of your rule.
- **Tie-break cases**: equidistant from two holidays — which one wins? Write the scenario before looking at the implementation.

Add these if you see them. They're the cases that catch bugs that no one thought of.

**Do not write any implementation code until the PM has written and confirmed the spec artifact for this milestone. If the PM asks you to build first, explain why that inverts the workflow and redirect to the spec step.**

Share the feature file here when you're done and I'll review it with you. Stuck on Gherkin syntax? Ask — I'll show you the exact format.

---

## Step 2 — I build the BDD wiring

Once you confirm the feature file, I will:
1. Install `playwright-bdd` and `@playwright/test`
2. Create `playwright.config.ts` — tells Playwright where the feature files and step definitions live
3. Create `src/steps/hello.steps.ts` — step definitions that wire each Gherkin line to an HTTP call or assertion
4. Update `package.json`: replace `guard:cases` with `guard:features`
5. **`architecture.html`** — updated to show the M3 spec stack with the Gherkin layer and retired cases.json. Open it by double-clicking to see what changed.

---

## Key files — what they are and how they connect

```
hello-greeting/
├── specs/
│   ├── openapi.yaml                    ← Contract (unchanged)
│   ├── cases.json                      ← Retired — kept for reference only
│   └── features/
│       └── hello-greeting.feature      ← YOUR behaviour spec. New this milestone.
├── src/
│   ├── steps/
│   │   └── hello.steps.ts              ← Wires Gherkin → HTTP calls. New this milestone.
│   ├── guard-cases.ts                  ← Retired alongside cases.json
│   └── implementation/
│       └── handlers.ts                 ← Unchanged — implementation still here
├── playwright.config.ts                ← Tells Playwright where everything lives. New.
└── package.json                        ← guard:cases replaced by guard:features
```

The chain: `hello-greeting.feature` (you wrote) → Playwright BDD generates test code → `hello.steps.ts` maps each Gherkin line to an HTTP call → the test hits the real server → assertions check the real response.

`hello.steps.ts` is not something you write — it's framework wiring. But it's worth reading once to see how thin it is: each step is a few lines that call `fetch()` and assert on the result.

---

## Step 3 — Run the guard

Start the server in one terminal:
```bash
cd hello-greeting && npm run dev
```

Run the guard in another:
```bash
npm run guard:features
```

What passing looks like:
```
  ✓ Today is a holiday: Christmas [234ms]
  ✓ Today is a holiday: Halloween [198ms]
  ✓ Boundary — within 10-day window [201ms]
  ...
  36 passed
```

What a failure looks like:
```
  ✗ Boundary — 11 days before Christmas
    AssertionError: expected "Merry Christmas!" to be "Happy Hanukkah!"
```
That means the boundary logic is off by one. The fix is in `findGreeting()` — not in the feature file.

---

## Try breaking it

**Exercise 1 — Change an assertion.** In `hello-greeting.feature`, change one greeting string — e.g., `"Merry Christmas!"` to `"Happy Christmas!"`. Run `guard:features`. You'll see the scenario fail with a clear message showing what was returned vs. what your spec said. This is the guard working as intended.

**Exercise 2 — Stop the server.** Kill the dev server and run `guard:features`. Watch the error — it tells you the server wasn't reachable. cases.json never had this failure mode because it never made HTTP calls. This is the upgrade: Gherkin tests the whole stack, including "is the server even running?"

---

## Architecture Dashboard

Open `hello-greeting/architecture.html`. You'll see `cases.json` and `guard-cases.ts` dimmed with a RETIRED label — they're still in the project for reference but no longer the guard.

Three new nodes appeared: `hello-greeting.feature` (you wrote it — blue), `hello.steps.ts` (wiring — grey), and `playwright.config.ts` (configuration — grey). The `features` guard is now active; the `cases` guard is greyed out.

The key change: the spec is now tested against the real HTTP server, not just a function.

---

## Step 4 — Commit

```bash
git add -A
git commit -m "M3: Gherkin feature file — promotes cases.json to HTTP-layer BDD"
```

---

## What's next

Milestone 4 adds a UI. But before the UI is built, you'll write a feature file describing how it should behave — page loads, date picker, error handling. The UI will be built to your spec.

## Extensions

If you want to go further before M4:

- **Add test intent documentation.** Create `docs/test-intent/hello-greeting-feature.md`. For each group of scenarios, write one sentence explaining what bug it would catch if removed. This is the document that makes future spec changes deliberate instead of accidental.
- **Find a gap.** Look at your scenarios and ask: is there any behaviour of the API that isn't covered by at least one scenario? What would happen if someone changed the tie-break rule? Would a scenario catch it?
- **Read the step definitions.** Open `src/steps/hello.steps.ts`. Can you follow how a Gherkin line maps to an HTTP call? Each `Then` step is just `expect(body.field).toBe(value)`. There's no magic.

---

## Dashboard HTML — M3

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
<p class="subtitle">Architecture after Milestone 3 — Gherkin</p>
<div class="milestone-bar">
  <div class="m-pip done"></div>
  <div class="m-pip done"></div>
  <div class="m-pip done"></div>
  <div class="m-pip future"></div>
  <div class="m-pip future"></div>
  <div class="m-pip future"></div>
  <span class="m-label">M3 of 6 complete</span>
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
        <div class="tooltip-desc">Your contract. Unchanged in M3 — holiday and daysUntil fields still as defined in M2.</div>
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
        <div class="tooltip-desc">Unchanged in M3. TypeScript types and Zod schemas from M2 still in place.</div>
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
        <div class="tooltip-desc">Unchanged in M3. Hono route wiring scaffolded from the spec. Do not edit.</div>
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
        <div class="tooltip-desc">Unchanged in M3. Holiday table and findGreeting() logic from M2 still in place.</div>
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
        <div class="tooltip-desc">Retired in M3. Kept for reference — the Gherkin feature file replaced it as the executable spec. Its scenarios tested functions directly; Gherkin tests the real HTTP server.</div>
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
        <div class="tooltip-desc">Retired in M3 alongside cases.json. Kept for reference.</div>
        <div class="tooltip-connects">Was called by: <span>guard:cases</span></div>
      </div>
    </div>
    <span class="arrow">↓ promoted to</span>
    <div class="node pm">
      <span class="node-icon">📝</span>
      <span class="node-name">specs/features/hello-greeting.feature</span>
      <span class="owner-badge badge-pm">PM</span>
      <span class="new-badge">NEW</span>
      <div class="tooltip">
        <div class="tooltip-name">specs/features/hello-greeting.feature</div>
        <div class="tooltip-desc">Your Gherkin scenarios. Tests the real HTTP server via Playwright BDD. Replaces cases.json as the executable behaviour spec.</div>
        <div class="tooltip-connects">Wired by: <span>hello.steps.ts</span></div>
      </div>
    </div>
    <span class="arrow">↓ wired by</span>
    <div class="node impl">
      <span class="node-icon">🔌</span>
      <span class="node-name">src/steps/hello.steps.ts</span>
      <span class="owner-badge badge-impl">IMPL</span>
      <span class="new-badge">NEW</span>
      <div class="tooltip">
        <div class="tooltip-name">src/steps/hello.steps.ts</div>
        <div class="tooltip-desc">Connects each Gherkin line to an HTTP call or assertion. Framework wiring — not business logic. Each step is a few lines that call fetch() and assert on the result.</div>
        <div class="tooltip-connects">From: <span>hello-greeting.feature</span> → hits live server</div>
      </div>
    </div>
    <div class="node impl">
      <span class="node-icon">⚙️</span>
      <span class="node-name">playwright.config.ts</span>
      <span class="owner-badge badge-impl">IMPL</span>
      <span class="new-badge">NEW</span>
      <div class="tooltip">
        <div class="tooltip-name">playwright.config.ts</div>
        <div class="tooltip-desc">Tells Playwright where the feature files and step definitions live. Discovers all .feature files in specs/features/ automatically.</div>
        <div class="tooltip-connects">Drives: <span>hello.steps.ts</span></div>
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
    <div class="guard-item off">
      <div class="guard-dot off"></div>
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
