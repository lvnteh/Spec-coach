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
