---
name: spec-coach-m3
description: Milestone 3 — Guide the PM through promoting cases.json to a Gherkin feature file and understanding why it matters
---

# Milestone 3: Gherkin

## Orient

In M2 you wrote cases.json — a list of expected inputs and outputs. It worked. The guard ran, it called the functions, everything was green.

This milestone replaces it with something better: a Gherkin `.feature` file. Same scenarios, different format. But the difference in format unlocks something important: Gherkin tests the real HTTP server, not just the functions inside it.

By the end of this session you'll have:
- `specs/features/hello-greeting.feature` with your scenarios in Gherkin
- A Playwright BDD runner that hits `http://localhost:3010`
- cases.json retired (kept as reference, no longer the guard)

## Why Gherkin instead of cases.json?

Your cases.json called `findGreeting()` directly — a pure function. It never made an HTTP request. This matters because there are two layers between "user sends a request" and "findGreeting() runs": the HTTP framework (Hono) and the route validator (Zod).

Here's what cases.json missed: if Zod silently intercepted a bad `?date` parameter and returned a Zod error instead of your custom error, cases.json wouldn't know. It was testing the right logic in the wrong place.

Gherkin + Playwright BDD runs against the live server. It sends real HTTP requests, reads real responses, and asserts on what the user actually receives. That's the only layer that matters.

The other reason: Gherkin is readable by anyone. A stakeholder, a support engineer, a new team member can open `hello-greeting.feature` and understand exactly what the system does — without reading code.

```
Scenario: Today is Christmas
  When I call GET hello with date "2026-12-25"
  Then the status is 200
  And the greeting is "Merry Christmas!"
  And the holiday is "Christmas"
  And daysUntil is 0
```

That's your cases.json entry, translated to Gherkin. The content is identical. The format changed. The runner changed. The test got stronger.

Reference: [What a finished feature file looks like at M3](https://github.com/lvnteh/specdrivengreeting/blob/m3-gherkin/specs/features/hello-greeting.feature)

## The mapping from cases.json to Gherkin

| cases.json field | Gherkin equivalent |
|---|---|
| `"description"` | Scenario name |
| `"input"` | `When I call GET hello with date "..."` |
| `"expected"` | `Then` assertions |

For many similar scenarios (all 24 holidays), use `Scenario Outline` with an `Examples` table:

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

For structurally distinct cases (boundary, fallback, tie-break), use individual `Scenario` blocks.

## Your task

Translate your cases.json into `specs/features/hello-greeting.feature`.

As you write it, you may notice cases that were hard to express in JSON but natural in Gherkin:
- **Boundary cases**: exactly 10 days before a holiday (inside window) vs. 11 days (outside)
- **Tie-break cases**: equidistant from two holidays — which one wins?

Add these if you see them. They'll make your spec more complete.

**Do not write any implementation code until the PM has written and confirmed the spec artifact for this milestone. If the PM asks you to build first, explain why that inverts the workflow and redirect to the spec step.**

Share the feature file when you're done and I'll review it.

## What I'll build from your feature file

Once you confirm the feature file, I will:
1. Install `playwright-bdd` and `@playwright/test`
2. Create `playwright.config.ts`
3. Create `src/steps/hello.steps.ts` — step definitions that wire each Gherkin line to an HTTP call or assertion
4. Update `package.json`: replace `guard:cases` with `guard:features`

## Running the guard

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
That means the boundary logic is off by one day. The fix is in `findGreeting()`.

## Commit

```
git commit -m "M3: Gherkin feature file — promotes cases.json to HTTP-layer BDD"
```

## What's next

Milestone 4 adds a UI. But before we build it, you'll write a feature file describing how the UI should behave — page loads, date picker interaction, error handling. The UI will be built to your spec, not the other way around.
