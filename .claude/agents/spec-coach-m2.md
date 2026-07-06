---
name: spec-coach-m2
description: Milestone 2 — Guide the PM through extending the OpenAPI spec and writing cases.json as an executable behaviour spec
---

# Milestone 2: Holiday Greetings

## Orient

In M1 you wrote a contract and built a Hello World server. The API works, but it always returns the same thing. In this milestone you define the holiday logic — which holidays exist, what greeting each one gets, and how the proximity window works.

Your main artifact is `cases.json` — a list of expected inputs and outputs that you write entirely. No YAML syntax, no code. Just JSON objects describing what the system must do.

By the end of this session you'll have:
- An extended `openapi.yaml` with the full response shape (`holiday`, `daysUntil` fields)
- A `cases.json` with at least 10 holiday scenarios you authored
- A running guard that executes your scenarios against the live implementation
- `architecture.html` updated — now shows `cases.json` and the guard runner in the spec stack

---

## What is cases.json, and why does a PM care?

A test case is a statement: "given this input, the output must be exactly this." Written as JSON:

```json
{
  "description": "Today IS Christmas",
  "input": { "date": "2026-12-25" },
  "expected": {
    "greeting": "Merry Christmas!",
    "holiday": "Christmas",
    "daysUntil": 0
  }
}
```

No code. No framework. A description, an input, and the exact expected output.

`cases.json` is a list of these. It's executable — the runner calls the greeting logic with each input and checks whether the output matches. If the implementation drifts from your spec, the guard fails immediately.

This is what "the spec is executable" means. cases.json isn't documentation that might drift — it's a check that runs every time.

Reference: [What a finished cases.json looks like at M2](https://github.com/lvnteh/specdrivengreeting/blob/m2-holiday-greetings/specs/cases.json) — look at this after you've written yours.

---

## Step 1 — Your product decisions

Before writing anything, decide the rules. Answer these:

1. **Which holidays should the API support?** Start with at least 5. The reference project has 24 for 2026 — you can use that list or define your own.
2. **What greeting does each holiday get?** "Merry Christmas!", "Happy Halloween!", etc. These will be asserted verbatim — typos matter.
3. **What is the proximity window?** How many days before or after a holiday should still return that holiday's greeting? The reference uses 10 days.
4. **What happens when two holidays are equidistant?** (You're 5 days away from both Halloween and Día de los Muertos.) Which wins — the upcoming one or the past one?
5. **What does the fallback look like?** When no holiday is nearby, what should the response be? The reference returns `{ "greeting": "Hello World", "holiday": null, "daysUntil": null }`.

I'll wait for your decisions before we write anything. If any of these feel ambiguous, ask — I'll help you think through the implications.

**Do not write any implementation code until the PM has written and confirmed the spec artifact for this milestone. If the PM asks you to build first, explain why that inverts the workflow and redirect to the spec step.**

---

## Step 2 — Extend the OpenAPI spec first

Before writing cases, the response schema needs two new fields. The spec changes before anything else:

```yaml
GreetingResponse:
  type: object
  required:
    - greeting
    - holiday
    - daysUntil
  additionalProperties: false
  properties:
    greeting:
      type: string
    holiday:
      type: string
      nullable: true
    daysUntil:
      type: integer
      nullable: true
      minimum: -10
      maximum: 10
```

Update `specs/openapi.yaml` with this schema. Then run:

```bash
npm run generate
```

This regenerates `src/generated/` from your updated spec. The types now include `holiday` and `daysUntil`. The handler will fail typecheck until it returns those fields too.

Order of operations: **spec change → regenerate → implement**. Always in this order.

---

## Step 3 — Write your cases.json

Create `hello-greeting/specs/cases.json`. Each entry needs:
- `"description"` — plain-English label (this is for you, not the machine)
- `"input.date"` — a date in `YYYY-MM-DD` format
- `"expected.greeting"` — exact string, will be compared character by character
- `"expected.holiday"` — exact string or `null`
- `"expected.daysUntil"` — integer or `null`

Write at least:
- One case per holiday (today IS the holiday → `daysUntil: 0`)
- One case several days before a holiday
- One case 11 days before a holiday (outside the window — should return fallback)
- One genuine fallback case (a date with no holiday nearby at all)

Share the file here when you're done and I'll review it with you. If you get stuck on any case — what the expected output should be, whether a date counts as "nearby" — ask.

---

## Step 4 — I build from your cases

Once you confirm cases.json, I will:
1. Implement the holiday table in `src/implementation/handlers.ts` based on your decisions
2. Implement `findGreeting()` — the function that applies your window and tie-break rules
3. Create `src/guard-cases.ts` — the runner that executes every case in your file
4. **`architecture.html`** — updated to show the full M2 spec stack. Open it by double-clicking to see what was added this milestone.

---

## Key files — what they are and how they connect

```
hello-greeting/
├── specs/
│   ├── openapi.yaml          ← Contract (shape). Updated this milestone.
│   └── cases.json            ← Behaviour spec (YOU wrote this). New this milestone.
├── src/
│   ├── generated/            ← Regenerated from openapi.yaml after your schema update.
│   ├── routes.ts             ← Unchanged — still derived from openapi.yaml.
│   ├── guard-cases.ts        ← Runner that executes cases.json. New this milestone.
│   └── implementation/
│       └── handlers.ts       ← Extended with holiday table + findGreeting() logic.
└── package.json              ← New script: guard:cases
```

The connection: `cases.json` describes what `findGreeting()` must do. `guard-cases.ts` calls `findGreeting()` with each case's input and checks the output against your expected values. If they don't match, you see exactly which case failed and what the actual output was.

---

## Step 5 — Run the guard

```bash
cd hello-greeting
npm run guard:cases
```

What passing looks like:
```
✅ Today IS Christmas
✅ 5 days before Halloween
✅ 11 days before Christmas — fallback
...
10/10 passed
```

What a failure looks like:
```
❌ 5 days before Halloween
   greeting: got "Hello World"  expected "Happy Halloween!"
```
That means `findGreeting()` didn't find Halloween 5 days out. The fix is in the handler's window logic — not in cases.json.

---

## Try breaking it

Once all cases pass, try two deliberate sabotages:

**Exercise 1 — Change a greeting string.** In `cases.json`, change `"Merry Christmas!"` to `"Happy Christmas!"`. Run `guard:cases`. You'll see the case fail because the assertion is verbatim. This is intentional — it protects you from accidental copy changes in the handler. Put it back.

**Exercise 2 — Shrink the window.** In `handlers.ts`, change `WINDOW_DAYS = 10` to `WINDOW_DAYS = 4`. Run `guard:cases`. Several cases that rely on the 5–10 day range will fail. This shows that your cases.json is the authoritative spec for the window rule — the implementation has to match it, not the other way round.

---

## Architecture Dashboard

After the guard goes green, open `hello-greeting/architecture.html`.

You'll see two new nodes: `specs/cases.json` (blue — you wrote it) and `src/guard-cases.ts` (grey — wires your cases to the implementation). The `cases` guard is now active in the right panel.

This is the spec-implementation chain made visible: your JSON file drives the runner, which calls the handler, which must match your expected outputs.

---

## Step 6 — Commit

```bash
git add -A
git commit -m "M2: Holiday greetings — extended OpenAPI schema + cases.json"
```

---

## What's next

Milestone 3 promotes cases.json to Gherkin. You'll rewrite the same scenarios in a `.feature` file — a format that's more expressive and, more importantly, tests the real HTTP server rather than calling functions directly. You'll discover what cases.json was silently missing.

## Extensions

If you want to go further before M3:

- **Add more holidays.** Pick 3 holidays from a different culture than your current list and add them. Write the cases first, then update the holiday table. Notice how the cases make you specify the exact greeting before you decide what the implementation looks like.
- **Add an edge case you haven't thought of.** What happens on December 31st — is New Year's Day 1 day away? Write the case, run the guard, see if it passes.
- **Read the guard runner.** Open `src/guard-cases.ts`. Can you follow what it does? It reads `cases.json`, calls `findGreeting()` for each input, and compares the result. There's no magic — it's a loop with assertions.

## Dashboard HTML — M2

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
<p class="subtitle">Architecture after Milestone 2 — Holiday Greetings</p>
<div class="milestone-bar">
  <div class="m-pip done"></div>
  <div class="m-pip done"></div>
  <div class="m-pip future"></div>
  <div class="m-pip future"></div>
  <div class="m-pip future"></div>
  <div class="m-pip future"></div>
  <span class="m-label">M2 of 6 complete</span>
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
        <div class="tooltip-desc">Your contract. Extended in M2 with holiday and daysUntil fields. Everything else derives from this.</div>
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
        <div class="tooltip-desc">Regenerated in M2 with holiday and daysUntil types. Run npm run generate after any spec change.</div>
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
        <div class="tooltip-desc">Hono route wiring scaffolded from the spec. Connects GET /hello to the handler. Do not edit — not regenerated by npm run generate.</div>
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
        <div class="tooltip-desc">Extended in M2 with the holiday table and findGreeting() logic. The only file with business logic.</div>
        <div class="tooltip-connects">Called by: <span>routes.ts</span> · Tested by: <span>guard-cases.ts</span></div>
      </div>
    </div>
    <span class="arrow">↓ behaviour spec</span>
    <div class="node pm">
      <span class="node-icon">📋</span>
      <span class="node-name">specs/cases.json</span>
      <span class="owner-badge badge-pm">PM</span>
      <span class="new-badge">NEW</span>
      <div class="tooltip">
        <div class="tooltip-name">specs/cases.json</div>
        <div class="tooltip-desc">Your executable test cases. Each entry has an input date and exact expected outputs. The runner calls findGreeting() with each input and checks the result.</div>
        <div class="tooltip-connects">Executed by: <span>src/guard-cases.ts</span></div>
      </div>
    </div>
    <span class="arrow">↓ executed by</span>
    <div class="node impl">
      <span class="node-icon">🏃</span>
      <span class="node-name">src/guard-cases.ts</span>
      <span class="owner-badge badge-impl">IMPL</span>
      <span class="new-badge">NEW</span>
      <div class="tooltip">
        <div class="tooltip-name">src/guard-cases.ts</div>
        <div class="tooltip-desc">Reads cases.json, calls findGreeting() for each case, and checks the output matches your expected values. Fails with a diff if any case doesn't match.</div>
        <div class="tooltip-connects">From: <span>cases.json</span> · Calls: <span>handlers.ts</span></div>
      </div>
    </div>
  </div>
  <div class="guard-panel">
    <div class="panel-title">Guards</div>
    <div class="guard-item pass">
      <div class="guard-dot pass"></div>
      <span class="guard-name">typecheck</span>
    </div>
    <div class="guard-item pass">
      <div class="guard-dot pass"></div>
      <span class="guard-name">cases</span>
    </div>
    <div class="guard-item off">
      <div class="guard-dot off"></div>
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
