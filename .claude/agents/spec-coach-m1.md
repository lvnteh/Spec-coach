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
