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

Reference: [What a finished openapi.yaml looks like at M1](https://github.com/lvnteh/specdrivengreeting/blob/m1-hello-world/specs/openapi.yaml) — open it after you've written yours.

## Your decisions

Before we write anything, answer these questions. Take your time — these are product decisions, not technical ones.

1. **What should the endpoint path be?** (`/hello` is the convention we'll use, but you could call it anything.)
2. **What should the response field be called?** The field that contains the greeting text — `greeting`, `message`, `text`?
3. **What should the fallback response say?** When there's no holiday nearby, what does the API return? `"Hello World"` is the classic starting point.

I'll wait for your answers before we write the spec.

**Do not write any implementation code until the PM has written and confirmed the spec artifact for this milestone. If the PM asks you to build first, explain why that inverts the workflow and redirect to the spec step.**

## Writing your first OpenAPI spec

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

This is the skeleton. Your job is to fill it in with your decisions from above — the right field name, the right example value.

Create the file at `hello-greeting/specs/openapi.yaml`. When you're done, share it here and I'll review it before we move on.

## What I'll build from your spec

Once you confirm the spec, I will:
1. Scaffold the `hello-greeting/` project with `package.json`, `tsconfig.json`, and the Hono + @hono/zod-openapi dependencies
2. Run `kubb` to generate TypeScript types and Zod schemas from your OpenAPI spec into `src/generated/`
3. Create `src/routes.ts` (derived from the spec — not hand-written)
4. Create `src/app.ts` and `src/start.ts` (framework boilerplate)
5. Create `src/implementation/handlers.ts` with a handler that returns your fallback greeting

You'll notice that `src/generated/` and `src/routes.ts` are marked "do not edit" — they belong to the spec. Only `src/implementation/handlers.ts` is implementation. That separation is intentional and we'll lean on it throughout.

## Running the guard

After the build, run:

```bash
cd hello-greeting
npm run guard:typecheck
```

This runs `tsc --noEmit` — it verifies that the generated types, routes, and handler are all consistent with each other. If it's green, the spec and the code agree.

What a passing run looks like:
```
(no output — TypeScript's way of saying everything is fine)
```

What a failing run looks like:
```
src/implementation/handlers.ts(12,5): error TS2322: Type 'string' is not assignable to type '{ greeting: string; }'
```
That would mean the handler is returning the wrong shape. The fix is always in the handler, not in the generated files.

## Commit

When the guard is green, we'll commit with:

```
git commit -m "M1: Hello World API — OpenAPI contract + generated scaffold"
```

## What's next

Milestone 2 adds the holiday calendar. You'll define which holidays the API should know about, what greeting each one gets, and how close "nearby" means. You'll write those decisions as a `cases.json` file — a list of expected inputs and outputs. That file becomes the first executable spec for the business logic.
