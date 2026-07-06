# Spec-Driven Development — What I Learned Building It

> *Personal notes from building the Hello Greeting project, July 2026.*
> *Not a reference — a narrative. The "why" behind the decisions.*

---

## The core idea

Instead of building an application directly, you build a framework in which an AI (or anyone) can build the application safely. The framework has three jobs:

1. **Say what the system must do** — in a structured, machine-readable form
2. **Check that it actually does it** — automatically, every time
3. **Limit what the AI is allowed to touch** — so it can only break small things

Everything that follows is just the concrete expression of those three jobs.

---

## The four layers, and what each one does

### 1. openapi.yaml — the single source of truth

This is not documentation. It is the contract. Every other artifact in the project derives from it.

- What endpoints exist
- What parameters they accept
- What responses they return
- What the error shapes look like

If it's not in `openapi.yaml`, it doesn't exist as far as the framework is concerned.

**What derives from it:**
- TypeScript types (via Kubb)
- Zod validation schemas (via Kubb)
- Route definitions (via @hono/zod-openapi)
- Contract tests (via Schemathesis)
- Workflow tests (via Arazzo)

**The lesson learned:** When the Zod schema on the route was validating the `?date` format, it silently intercepted `"banana"` before the handler ran — and returned a Zod error object instead of the custom error message. The spec said what the error shape should be. The implementation violated it. The contract guard (Schemathesis) caught the shape violation. But it was found in the browser first because the cases guard didn't cover the HTTP layer.

The spec being the source of truth only works if every layer actually derives from it. A Zod regex added directly to the route — not derived from the spec — is a leak. It creates a second source of truth.

---

### 2. specs/cases.json → specs/features/*.feature — the behaviour spec

This is the PM's document. It answers: **for a given input, what is the exact output?**

Originally this was `cases.json` — a list of `{ description, input, expected }` objects. That's the BDD idea expressed as JSON. It was replaced with Gherkin (`.feature` files) because Gherkin:

- Has a standard runner (Playwright BDD / Cucumber)
- Hits the real HTTP server, not just pure functions
- Is more readable to non-engineers

**The substitution is exact:**

| cases.json | Gherkin |
|---|---|
| `"description"` | Scenario name |
| `"input"` | `Given` / `When` |
| `"expected"` | `Then` |

They express the same thing. The format changed. The content didn't.

**The lesson learned:** `cases.json` + a custom runner (`guard-cases.ts`) called pure functions directly. This meant it missed the HTTP layer entirely. When Zod silently intercepted `"banana"` on the route, the cases guard showed 40/40 green while the browser showed a broken response. Gherkin + Playwright BDD closes that gap — it hits the actual running server.

**What goes in the feature file vs. what doesn't:**
- Business rules → feature file (greeting logic, error messages, validation)
- Protocol/shape conformance → Schemathesis (handles this automatically)
- CSS, layout, copy → neither (not behaviour)

**Scenario Outline vs. individual Scenario:**
- Use `Scenario Outline` + `Examples` table when the same behaviour repeats across many data points (24 holidays, each follows the same pattern)
- Use individual `Scenario` when the case is structurally distinct (the boundary case, the null-handling case)

---

### 3. guard:contract — what humans don't think of

Schemathesis reads `openapi.yaml` and auto-generates inputs. It tries:
- Arrays where strings are expected (`?date=a&date=b`)
- Empty strings
- Very long strings
- Negative numbers
- Every combination it can derive from the schema

You don't write these. You don't think of them. Schemathesis finds them.

**The lesson learned:** Schemathesis sent `?date=a&date=b` — a query parameter as an HTTP array. The route's Zod schema expected a `string`, received an array, and Hono returned a Zod error object that didn't match the `ErrorResponse` schema. Schemathesis caught it. Fix: make the route accept `string | array` and transform to the first element before the handler sees it.

**Division of responsibility:**
- `guard:features` — business rules (PM writes these)
- `guard:contract` — protocol/shape conformance (Schemathesis generates these)

These are complementary. Neither replaces the other.

---

### 4. guard:workflow — end-to-end status checks

Arazzo is a YAML format for describing how API calls chain together. You declare steps, each step calls an `operationId` from the OpenAPI spec, and you assert success criteria.

In this project it's simple — four workflows, each asserting a status code. The status code check is the thing Schemathesis doesn't do well (it checks schema shape, not business-level outcomes).

**The limitation discovered:** Redocly Respect (the Arazzo runner) only reliably supports `$statusCode` comparisons in `successCriteria`. Body value assertions (`$response.body.greeting == "Merry Christmas!"`) are not supported in v2.37.0. So workflows cover: did the right HTTP status come back? The feature file covers: was the content correct?

---

## The testing pyramid

```
        /\
       /  \        ← UI behaviour (4 scenarios)
      /----\
     /      \      ← Workflow / status (4 workflows)
    /--------\
   /          \    ← Contract / protocol (Schemathesis, auto)
  /------------\
 /              \  ← Business logic (33 scenarios)
/________________\
```

Many tests at the bottom where rules live. Few at the top where display lives.

**Why the UI needs so few scenarios:**
The UI has no rules. It calls the API and displays the result. The only questions worth asserting: does it call the API, does it show the result, does it handle errors. Everything else (layout, styling, copy) is implementation detail — not behaviour.

---

## BDD and TDD — how they relate

**TDD** — Test-Driven Development. Write a failing test, write minimum code to pass it, refactor. Tests are unit-level, written in code, by engineers.

**BDD** — Behaviour-Driven Development. TDD applied at the feature/behaviour layer, with a syntax non-engineers can read and contribute to. Given/When/Then. Gherkin is just structured Given/When/Then.

Both are useful. They operate at different levels and don't conflict:
- BDD covers: does the feature behave correctly?
- TDD covers: does this specific function work correctly?

What was built here is BDD-style at the behaviour layer. The PM wrote the scenarios (what the system should do). The engineer wired the step definitions (how to check it). That division is the point.

---

## ADRs — why decisions disappear without them

The system has no memory. Files capture the current state. They don't capture why.

When the Gherkin migration happened, it triggered a PM decision: the `?date` param only accepts 2026, so the feature file can only use 2026 dates. That decision:
- Changed 24 holiday rows (2025 → 2026 dates)
- Changed tie-break rows (recalculated for 2026 holiday positions)
- Changed fallback dates (Feb 26 was no longer a gap in 2026)
- Split the null-handling scenario out of the Outline

None of that is visible in the files. Someone reading the feature file tomorrow has no idea why 2025 dates aren't tested via HTTP.

**ADRs (Architecture Decision Records)** capture:
- **Context** — what situation forced the decision
- **Decision** — what was chosen
- **Options considered** — what was weighed and rejected
- **Consequences** — what is now true, including things to watch for

The "Consequences" section is the most valuable over time. It's how future-you answers "why is it like this?" without archaeology.

Every PM decision that changes the shape of the spec deserves an ADR.

---

## The mistake that taught the most

**The silent Zod intercept.**

The route had this validation:
```typescript
date: z.string().regex(/^\d{4}-\d{2}-\d{2}$/).optional()
```

When `?date=banana` was sent, Zod caught it before the handler ran and returned:
```json
{ "success": false, "error": { "issues": [{ "validation": "regex" ... }] } }
```

Instead of the custom error:
```json
{ "error": "Nem valid datum, BLEGH" }
```

The cases guard showed 40/40 green. The contract guard showed 14/14 green. The browser showed a broken response.

**Why the guards missed it:** `guard-cases.ts` called `validateDateParam()` directly — a pure function. It never made an HTTP request. It never hit the Zod validation on the route. The guard was testing the right logic in the wrong place.

**The fix:** Remove all format validation from the route. The handler is the single validation point. The route's Zod schema stays permissive — just enough to not break on unexpected input shapes (like arrays).

**The principle:** Every validation that produces a user-facing error must live in the handler. The route schema is framework wiring, not business logic.

---

## What spec-driven development is not

It is not "the AI builds the app and humans don't work anymore."

The effort moves. Instead of writing implementation code, you write specs, set up generators, and configure guards. The first time you do this for a new domain, it costs more than writing the app directly. The payoff is that the next app in the same framework is dramatically cheaper — and the AI's blast radius is bounded.

The AI only touches one file: `src/implementation/handlers.ts`. Everything else is generated or framework-owned. If the AI writes something that breaks the spec, the guards catch it. If it tries to edit the wrong file, the framework prevents it.

That's the whole system.

---

## Quick reference — what each file is for

| File | Owned by | Purpose |
|---|---|---|
| `specs/openapi.yaml` | PM + Engineer | Single source of truth for API shape |
| `specs/features/*.feature` | PM | Behaviour scenarios (Gherkin) |
| `specs/workflows/*.arazzo.yaml` | Engineer | End-to-end workflow status checks |
| `src/generated/` | Kubb (auto) | Types + Zod schemas — never edit |
| `src/routes.ts` | Framework | Route wiring — never edit |
| `src/implementation/handlers.ts` | AI / Engineer | Business logic — only file AI touches |
| `src/steps/*.ts` | Engineer | Step definitions wiring Gherkin to code |
| `docs/decisions/ADR-*.md` | PM + Engineer | Why decisions were made |

## Quick reference — what each guard checks

| Guard | Tool | What it catches |
|---|---|---|
| `guard:typecheck` | TypeScript | Type errors, missing response types |
| `guard:features` | Playwright BDD | Business rules, HTTP-layer behaviour, UI behaviour |
| `guard:contract` | Schemathesis | Protocol conformance, edge case inputs |
| `guard:workflow` | Arazzo / Redocly Respect | End-to-end status codes |
