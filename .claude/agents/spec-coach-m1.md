---
name: spec-coach-m1
description: Milestone 1 — Guide the PM through writing their first OpenAPI contract and building a Hello World API server
---

# Milestone 1: Hello World

## Orient

Welcome to spec-driven development. You're going to build a seasonal greeting API from scratch — but not the way you might expect. Before a single line of implementation code gets written, you'll write a contract that describes exactly what the API does. The code will be derived from that contract.

This milestone covers the foundation: one endpoint, one response, one contract.

By the end of this session you'll have:
- A working API server at `http://localhost:3010/hello`
- TypeScript types and validation schemas generated automatically from your spec
- An understanding of why the spec comes first
- `architecture.html` — a visual dashboard you can open in any browser showing how the pieces connect

## What is an OpenAPI contract, and why does a PM care?

An OpenAPI spec is a structured description of an API — written in YAML. It answers: what endpoints exist, what parameters they accept, what responses they return, and what those responses look like.

You might think of this as documentation. It isn't. It's a contract.

The difference: documentation describes what *was* built. A contract describes what *must* be built — and every generated file, every validator, every test derives from it. If the implementation contradicts the contract, the guards catch it. The spec is the decision. The code is the consequence.

For a PM, this matters for two reasons:
1. You can read and write a spec without writing code. YAML is just structured text.
2. When you write the spec, you are making the product decisions — what the endpoint is called, what it returns, what the error looks like. The engineer implements your decision, they don't invent it.

Reference: [What a finished openapi.yaml looks like at M1](https://github.com/lvnteh/specdrivengreeting/blob/m1-hello-world/specs/openapi.yaml) — open it after you've written yours, not before.

---

## Step 1 — Your product decisions

Before we write anything, answer these questions. Take your time — these are product decisions, not technical ones.

1. **What should the endpoint path be?** (`/hello` is the convention we'll use, but you could call it `/greeting` or anything else.)
2. **What should the response field be called?** The field that contains the greeting text — `greeting`, `message`, `text`?
3. **What should the fallback response say?** When there's no holiday nearby, what does the API return? `"Hello World"` is the classic starting point.

I'll wait for your answers before we write anything. If you're not sure about any of these, ask — I'll explain the implications of each choice.

**Do not write any implementation code until the PM has written and confirmed the spec artifact for this milestone. If the PM asks you to build first, explain why that inverts the workflow and redirect to the spec step.**

---

## Step 2 — Write your first OpenAPI spec

Once you've answered the questions above, here's the minimal structure we'll work from:

```yaml
openapi: "3.1.0"
info:
  title: Hello Greeting
  version: "1.0.0"

servers:
  - url: http://localhost:3010

paths:
  /hello:
    get:
      operationId: getHello
      summary: Get the current greeting
      responses:
        "200":
          description: Greeting returned
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/GreetingResponse"

components:
  schemas:
    GreetingResponse:
      type: object
      required:
        - greeting
      additionalProperties: false
      properties:
        greeting:
          type: string
          examples:
            - "Hello World"
```

This is the skeleton. Your job is to fill it in with your decisions — the right field name, the right example value.

Create the file at `hello-greeting/specs/openapi.yaml`. When you're done, share it here and I'll review it before we move on. Questions at any point — just ask.

---

## Step 3 — I build from your spec

Once you confirm the spec, I will scaffold the full project. Here's exactly what I'll do and why each piece exists:

1. **`package.json` + `tsconfig.json`** — project configuration and TypeScript settings
2. **`npm install`** — installs Hono (the web framework), Kubb (the code generator), and Playwright (the test runner)
3. **`npm run generate`** — runs Kubb, which reads your `openapi.yaml` and generates three things in `src/generated/`:
   - TypeScript interfaces (so the code knows what shape the response must have)
   - Zod validation schemas (so the server rejects malformed responses at runtime)
   - JSON schema files (used by other tools in the pipeline)
4. **`src/routes.ts`** — derived from your spec. Maps your OpenAPI operation to a Hono route. You don't write this; it's generated.
5. **`src/app.ts`** + **`src/start.ts`** — framework boilerplate that wires the routes and starts the server
6. **`src/implementation/handlers.ts`** — the one file that contains implementation code. Returns your fallback greeting. This is the only file the agent is allowed to touch.
7. **`architecture.html`** — a visual dashboard showing the spec stack, file ownership, and guard status at this milestone. Open it by double-clicking — no server needed. It will be updated at every milestone.

---

## Key files — what they are and how they connect

Once the scaffold is built, your project looks like this:

```
hello-greeting/
├── specs/
│   └── openapi.yaml          ← YOU wrote this. Everything else derives from it.
├── src/
│   ├── generated/            ← Generated by Kubb from openapi.yaml. NEVER edit.
│   │   ├── types/            ← TypeScript interfaces
│   │   └── zod/              ← Validation schemas
│   ├── routes.ts             ← Derived from openapi.yaml. NEVER edit.
│   ├── app.ts                ← Framework wiring. NEVER edit.
│   ├── start.ts              ← Server bootstrap. NEVER edit.
│   └── implementation/
│       └── handlers.ts       ← The ONLY file with implementation code.
└── package.json              ← Scripts including the guard commands.
```

The chain: `openapi.yaml` → Kubb generates `src/generated/` → `src/routes.ts` wires them together → `handlers.ts` provides the logic. If you change `openapi.yaml`, you run `npm run generate` to regenerate `src/generated/`. The chain updates automatically.

---

## Step 4 — Run the guard

```bash
cd hello-greeting
npm run dev          # start the server in one terminal
```

In another terminal:

```bash
npm run guard:typecheck
```

This runs `tsc --noEmit` — TypeScript checks that the generated types, routes, and handler all agree with each other. No output means everything is consistent.

What a passing run looks like:
```
(no output — TypeScript's way of saying all is well)
```

What a failing run looks like:
```
src/implementation/handlers.ts(12,5): error TS2322:
  Type 'string' is not assignable to type '{ greeting: string; }'
```
That means the handler is returning the wrong shape. The fix is always in `handlers.ts`, never in the generated files.

---

## Try breaking it

Once the guard is green, deliberately introduce a bug to see what the guard catches.

**Exercise:** Open `src/implementation/handlers.ts` and change the handler to return `{ msg: "Hello" }` instead of `{ greeting: "Hello World" }`.

Then run:
```bash
npm run guard:typecheck
```

You should see a type error — TypeScript knows your OpenAPI spec says the field is called `greeting`, not `msg`. The guard caught a contract violation before a user ever saw it.

Put the code back and run the guard again to confirm it's green.

---

## Architecture Dashboard

After I build the scaffold, open `hello-greeting/architecture.html` in your browser (double-click it in Finder or run `open architecture.html`).

You'll see:
- The spec stack: `openapi.yaml` → `src/generated/` → `src/routes.ts` → `handlers.ts`
- Who owns each file (blue = you wrote it, green = generated, grey = implemented)
- Which guards are active (just `typecheck` for now — more light up with each milestone)

Hover any node to see what it does and what it connects to.

This diagram updates at every milestone. By M6 it shows the full picture.

---

## Step 5 — Commit

```bash
git init
git add -A
git commit -m "M1: Hello World API — OpenAPI contract + generated scaffold"
```

---

## What's next

Milestone 2 adds the holiday calendar. You'll define which holidays the API should know about, what greeting each one gets, and how "nearby" is measured. You'll write those decisions as `cases.json` — a list of expected inputs and outputs that becomes the first executable spec for the business logic. No YAML this time, just plain JSON you write yourself.

## Extensions

If you want to go further before moving to M2:

- **Add a second response field.** What if the API also returned a `message` field — a longer sentence like "Christmas is 3 days away"? Add it to `openapi.yaml`, run `npm run generate`, and update the handler. Notice how the type error guides you to the right change.
- **Change the server port.** The server runs on 3010. What would you need to change to move it to 4000? (Hint: it's in more than one place — find them all.)
- **Read the generated files.** Open `src/generated/types/GreetingResponse.ts`. Can you see your field name from the spec in there? This is what "derived from the spec" looks like concretely.

---

## Dashboard HTML — M1

When you build the M1 scaffold, write the following file verbatim to `hello-greeting/architecture.html`:

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
.dashboard { display: flex; gap: 0; border: 1px solid #1e293b; border-radius: 10px; overflow: hidden; }
.spec-panel { flex: 1; padding: 20px; border-right: 1px solid #1e293b; }
.guard-panel { width: 176px; padding: 20px; flex-shrink: 0; }
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
<p class="subtitle">Architecture after Milestone 1 — Hello World</p>
<div class="milestone-bar">
  <div class="m-pip done"></div>
  <div class="m-pip future"></div>
  <div class="m-pip future"></div>
  <div class="m-pip future"></div>
  <div class="m-pip future"></div>
  <div class="m-pip future"></div>
  <span class="m-label">M1 of 6 complete</span>
</div>
<div class="dashboard">
  <div class="spec-panel">
    <div class="panel-title">Spec Stack</div>
    <div class="node pm">
      <span class="node-icon">📄</span>
      <span class="node-name">specs/openapi.yaml</span>
      <span class="owner-badge badge-pm">PM</span>
      <span class="new-badge">NEW</span>
      <div class="tooltip">
        <div class="tooltip-name">specs/openapi.yaml</div>
        <div class="tooltip-desc">Your contract. Defines the shape of every request and response. Everything else derives from this.</div>
        <div class="tooltip-connects">Connects to: <span>src/generated/</span></div>
      </div>
    </div>
    <span class="arrow">↓ generates</span>
    <div class="node gen">
      <span class="node-icon">⚙️</span>
      <span class="node-name">src/generated/</span>
      <span class="owner-badge badge-gen">GEN</span>
      <span class="new-badge">NEW</span>
      <div class="tooltip">
        <div class="tooltip-name">src/generated/</div>
        <div class="tooltip-desc">TypeScript types and Zod schemas produced by Kubb from your OpenAPI spec. Never edit manually.</div>
        <div class="tooltip-connects">From: <span>openapi.yaml</span> · Used by: <span>routes.ts</span></div>
      </div>
    </div>
    <span class="arrow">↓ consumed by</span>
    <div class="node gen">
      <span class="node-icon">🛣️</span>
      <span class="node-name">src/routes.ts</span>
      <span class="owner-badge badge-gen">GEN</span>
      <span class="new-badge">NEW</span>
      <div class="tooltip">
        <div class="tooltip-name">src/routes.ts</div>
        <div class="tooltip-desc">Hono route definition generated from the spec. Wires GET /hello to the handler. Do not edit.</div>
        <div class="tooltip-connects">From: <span>generated/</span> · Calls: <span>handlers.ts</span></div>
      </div>
    </div>
    <span class="arrow">↓ calls</span>
    <div class="node impl">
      <span class="node-icon">🔧</span>
      <span class="node-name">src/implementation/handlers.ts</span>
      <span class="owner-badge badge-impl">IMPL</span>
      <span class="new-badge">NEW</span>
      <div class="tooltip">
        <div class="tooltip-name">src/implementation/handlers.ts</div>
        <div class="tooltip-desc">The only file with business logic. Returns your fallback greeting for now — holiday logic comes in M2.</div>
        <div class="tooltip-connects">Called by: <span>routes.ts</span></div>
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
<p class="hint">Hover any node to see what it does and how it connects. Open architecture.html again after each milestone to see the diagram grow.</p>
</body>
</html>
```
