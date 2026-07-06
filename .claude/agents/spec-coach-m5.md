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
