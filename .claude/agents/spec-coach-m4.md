---
name: spec-coach-m4
description: Milestone 4 — Guide the PM through writing UI behaviour scenarios in Gherkin before the UI is built
---

# Milestone 4: UI

## Orient

In M3 you proved that spec-first works for business logic. This milestone proves it works for UI too.

You'll write a Gherkin feature file describing how the UI should behave — before a single line of HTML exists. Then I'll build the UI to match your spec.

By the end of this session you'll have:
- `specs/features/hello-greeting-ui.feature` with UI behaviour scenarios you wrote
- A plain HTML page served at `GET /` with two panels: "Right now" and "Pick a date"
- Playwright step definitions wired to the UI
- `architecture.html` updated — UI feature file and step definitions added to the spec stack

---

## What goes in a UI feature file

The UI has no business rules. It calls the API and displays the result. So the scenarios are simple — interactions and their visible outcomes.

What belongs in the UI spec:
- Does the page load and show today's greeting?
- Can the user pick a date and see the result?
- What happens when the API returns an error?

What doesn't belong:
- Layout, colours, fonts — implementation details, not behaviour
- All 24 holidays — the API test suite covers that; the UI just needs to show what the API returns

Reference: [What a finished UI feature file looks like at M4](https://github.com/lvnteh/specdrivengreeting/blob/m4-ui/specs/features/hello-greeting-ui.feature)

---

## Step 1 — Your product decisions

1. **What should the page show on load?** Today's greeting, pulled automatically from the API?
2. **How does the user query a specific date?** A date input + a "Go" button?
3. **What should happen when the API returns an error?** Toast notification? Inline message? The guard will test whichever you decide — so decide before we build.

Ask if you're not sure about any of these — I'll help you think through the right scope.

**Do not write any implementation code until the PM has written and confirmed the spec artifact for this milestone. If the PM asks you to build first, explain why that inverts the workflow and redirect to the spec step.**

---

## Step 2 — Write the UI feature file

Here's the minimal structure to work from:

```gherkin
Feature: Greeting UI

  Scenario: Page loads showing today's greeting
    Given I open the greeting page
    Then a non-empty greeting is visible in the now card

  Scenario: Picking a holiday date shows its greeting
    Given I open the greeting page
    When I pick the date "2026-12-25" and press Go
    Then the picked greeting shows "Merry Christmas!"
    And the picked meta contains "Christmas"

  Scenario: Picking a date with no nearby holiday shows Hello World
    Given I open the greeting page
    When I pick the date "2026-07-26" and press Go
    Then the picked greeting shows "Hello World"
    And no holiday is shown in the picked meta

  Scenario: API error shows a toast notification
    Given I open the greeting page
    When I pick the date "2026-13-01" and press Go
    Then a toast message is visible
```

Create `specs/features/hello-greeting-ui.feature` based on your decisions. Share it here when you're done — I'll review it with you before building.

---

## Step 3 — I build from your feature file

Once you confirm the UI feature file, I will:
1. Create `src/public/index.html` — a minimal HTML page served at `GET /`
2. Update `src/app.ts` to serve the UI at the root route
3. Create `src/steps/hello-ui.steps.ts` — step definitions for UI interactions (open page, pick date, assert visible text)
4. **`architecture.html`** — updated to show the M4 spec stack with the UI layer. Open it by double-clicking to see what was added.

---

## Key files — what they are and how they connect

```
hello-greeting/
├── specs/
│   └── features/
│       ├── hello-greeting.feature      ← API behaviour (from M3, unchanged)
│       └── hello-greeting-ui.feature   ← UI behaviour (YOU wrote this). New.
├── src/
│   ├── public/
│   │   └── index.html                  ← The UI. Built to your spec. New.
│   ├── steps/
│   │   ├── hello.steps.ts              ← API step definitions (unchanged)
│   │   └── hello-ui.steps.ts           ← UI step definitions. New.
│   └── app.ts                          ← Updated to serve GET /
└── playwright.config.ts                ← Unchanged — picks up both feature files automatically
```

Both feature files run under `guard:features`. Playwright discovers all `.feature` files in `specs/features/` automatically — no configuration change needed.

The UI step definitions work differently from the API ones: instead of calling `fetch()`, they use Playwright's browser automation — open a real browser, navigate to the page, click the date input, assert on what's visible. But from the feature file's perspective, it's the same Given/When/Then format.

---

---

## Architecture Dashboard

Open `hello-greeting/architecture.html`. Three new nodes: the UI feature file (blue — you wrote it), the UI step definitions (grey — browser automation wiring), and `index.html` (grey — the UI itself).

Notice that `features` in the guard panel now covers two feature files — the API spec and the UI spec both run under the same guard. The diagram shows both feeding into the same runner.

## Step 4 — Run the guard

With the server running:
```bash
npm run guard:features
```

This now runs both feature files. The API scenarios run as before. The UI scenarios open a headless browser, load `http://localhost:3010`, interact with the page, and assert on visible text.

What passing looks like:
```
  ✓ Page loads showing today's greeting [1.2s]
  ✓ Picking a holiday date shows its greeting [890ms]
  ✓ Picking a date with no nearby holiday shows Hello World [743ms]
  ✓ API error shows a toast notification [654ms]
  ... (all API scenarios still pass)
```

UI tests are slower than API tests — they're running a real browser. That's expected.

---

## Try breaking it

**Exercise 1 — Remove the toast.** Open `src/public/index.html` and find the code that shows the error toast. Comment it out. Run `guard:features`. The "API error shows a toast notification" scenario will fail — the guard is checking that the UI honours its own spec. This is spec-first applied to UI: the test existed before the implementation, and it will catch regressions if the UI changes.

**Exercise 2 — Change the date format.** In `hello-greeting-ui.feature`, change `"2026-12-25"` to `"25/12/2026"` in the holiday scenario. Run `guard:features`. It will fail — the step definition sends that string to the API, which rejects the format. Your UI spec and your API spec are now in conflict. This is what "the spec is the source of truth" means: both layers have to agree.

---

## Step 5 — Commit

```bash
git add -A
git commit -m "M4: UI feature file + HTML page built to spec"
```

---

## What's next

Milestone 5 adds a new feature: the `?date` query parameter. This is where spec-first is hardest to maintain — you're changing something that already exists, and the instinct is to edit the code first. We'll hold the discipline: spec change first, new scenarios second, implementation last.

## Extensions

If you want to go further before M5:

- **Add a "copy link" button scenario.** What if the user could copy a URL that pre-fills the date they picked? Write the Gherkin scenario first. You don't have to implement it — just notice how writing the scenario forces you to decide what the URL should look like.
- **Add a loading state.** What should the UI show while waiting for the API response? Write a scenario for it. This often reveals timing assumptions you hadn't made explicit.
- **Check what `guard:features` covers vs. doesn't.** Look at your UI feature file. Is there any UI behaviour you care about that has no scenario? Write it down — that's your test debt.

## Dashboard HTML — M4

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
<p class="subtitle">Architecture after Milestone 4 — UI</p>
<div class="milestone-bar">
  <div class="m-pip done"></div>
  <div class="m-pip done"></div>
  <div class="m-pip done"></div>
  <div class="m-pip done"></div>
  <div class="m-pip future"></div>
  <div class="m-pip future"></div>
  <span class="m-label">M4 of 6 complete</span>
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
        <div class="tooltip-desc">Your contract. Unchanged in M4.</div>
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
        <div class="tooltip-desc">Unchanged in M4.</div>
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
        <div class="tooltip-desc">Updated in M4 to also serve GET / (the UI). Scaffolded from the spec. Do not edit.</div>
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
        <div class="tooltip-desc">Unchanged in M4. Holiday table and findGreeting() logic from M2 still in place.</div>
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
        <div class="tooltip-desc">Your API behaviour scenarios in Gherkin. Unchanged in M4.</div>
        <div class="tooltip-connects">Wired by: <span>hello.steps.ts</span></div>
      </div>
    </div>
    <span class="arrow">↓ UI tested by</span>
    <div class="node pm">
      <span class="node-icon">🖥️</span>
      <span class="node-name">specs/features/hello-greeting-ui.feature</span>
      <span class="owner-badge badge-pm">PM</span>
      <span class="new-badge">NEW</span>
      <div class="tooltip">
        <div class="tooltip-name">specs/features/hello-greeting-ui.feature</div>
        <div class="tooltip-desc">Your UI behaviour scenarios: page load, date picker, error toast. Tests the browser instead of the API directly.</div>
        <div class="tooltip-connects">Wired by: <span>hello-ui.steps.ts</span></div>
      </div>
    </div>
    <span class="arrow">↓ wired by</span>
    <div class="node impl">
      <span class="node-icon">🔌</span>
      <span class="node-name">src/steps/hello.steps.ts</span>
      <span class="owner-badge badge-impl">IMPL</span>
      <div class="tooltip">
        <div class="tooltip-name">src/steps/hello.steps.ts</div>
        <div class="tooltip-desc">API step definitions from M3. Unchanged.</div>
        <div class="tooltip-connects">From: <span>hello-greeting.feature</span> → live server</div>
      </div>
    </div>
    <div class="node impl">
      <span class="node-icon">🔌</span>
      <span class="node-name">src/steps/hello-ui.steps.ts</span>
      <span class="owner-badge badge-impl">IMPL</span>
      <span class="new-badge">NEW</span>
      <div class="tooltip">
        <div class="tooltip-name">src/steps/hello-ui.steps.ts</div>
        <div class="tooltip-desc">Wires UI Gherkin lines to Playwright browser automation — opens a real browser, navigates to the page, interacts with it.</div>
        <div class="tooltip-connects">From: <span>hello-ui.feature</span> → headless browser</div>
      </div>
    </div>
    <div class="node impl">
      <span class="node-icon">🌐</span>
      <span class="node-name">src/public/index.html</span>
      <span class="owner-badge badge-impl">IMPL</span>
      <span class="new-badge">NEW</span>
      <div class="tooltip">
        <div class="tooltip-name">src/public/index.html</div>
        <div class="tooltip-desc">The HTML UI served at GET /. Two panels: "Right now" (today's greeting) and "Pick a date". Built to your UI feature file spec.</div>
        <div class="tooltip-connects">Served by: <span>routes.ts</span></div>
      </div>
    </div>
    <div class="node impl">
      <span class="node-icon">⚙️</span>
      <span class="node-name">playwright.config.ts</span>
      <span class="owner-badge badge-impl">IMPL</span>
      <div class="tooltip">
        <div class="tooltip-name">playwright.config.ts</div>
        <div class="tooltip-desc">Unchanged. Discovers all .feature files in specs/features/ — now picks up both API and UI feature files automatically.</div>
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
