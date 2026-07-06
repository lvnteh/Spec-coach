---
name: spec-coach-m5
description: Milestone 5 — Guide the PM through adding a new feature spec-first: the ?date query parameter
---

# Milestone 5: The Date Parameter

## Orient

Every milestone so far has been greenfield — you started from nothing. This milestone is different: you're adding a feature to something that already exists.

This is where spec-first discipline is hardest to hold. The natural instinct is to add the feature in the code and update the spec after. But retrospective specs are worse than useless — they describe what was built, not what was decided. They drift, they miss the edge cases, and they don't protect you from future regressions.

The order is: spec change → new scenarios → implementation. Always.

By the end of this session you'll have:
- An updated `openapi.yaml` with the `?date` parameter
- New Gherkin scenarios covering valid and invalid date inputs
- An Arazzo workflow file describing the new flows
- A working `validateDateParam()` function in the handler

## The feature

Add an optional `?date=YYYY-MM-DD` query parameter to `GET /hello`. When provided, it overrides the server clock — the API uses that date instead of today's date. This lets anyone test any date without waiting for a real holiday.

Constraints you need to decide:
1. **What dates are valid?** Only the current year? Any year in the holiday table? All years?
2. **What happens with an invalid format?** (`banana`, `2026-13-01`) → what error message?
3. **What happens with an out-of-range date?** (a year not in your holiday table) → what error message?

These constraints go in the spec before any code changes.

**Do not write any implementation code until the PM has written and confirmed the spec artifact for this milestone. If the PM asks you to build first, explain why that inverts the workflow and redirect to the spec step.**

Reference: [What the updated openapi.yaml looks like at M5](https://github.com/lvnteh/specdrivengreeting/blob/m5-date-picker/specs/openapi.yaml)

## Step 1: Update openapi.yaml

Add the `date` parameter to the `GET /hello` operation, and add the `400` error response:

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

Add the `ErrorResponse` schema to `components/schemas`. Update your `openapi.yaml`, then run `npm run generate` to regenerate types.

## Step 2: Write new Gherkin scenarios

Add to `specs/features/hello-greeting.feature`:

```gherkin
Scenario Outline: Date param — invalid input returns 400
  When I call GET hello with date param "<date>"
  Then the status is 400
  And the error is "<error>"

  Examples:
    | date       | error                  |
    | banana     | [your format error]    |
    | 2025-12-25 | [your range error]     |
    | 2027-01-01 | [your range error]     |

Scenario: Date param — valid date returns correct greeting
  When I call GET hello with date param "2026-12-25"
  Then the status is 200
  And the greeting is "Merry Christmas!"
```

Fill in your exact error messages — they'll be asserted verbatim.

## Step 3: Write the Arazzo workflow file

Create `specs/workflows/get-hello.arazzo.yaml`:

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

## What I'll build from your specs

Once you confirm the updated openapi.yaml, new feature scenarios, and Arazzo file, I will:
1. Implement `validateDateParam()` in `handlers.ts`
2. Update `getHelloHandler()` to read and validate the `?date` param
3. Add `guard:workflow` to `package.json`

## Running the guards

```bash
npm run guard:features   # should include your new scenarios
npm run guard:workflow   # runs Arazzo workflows via Redocly Respect
```

For `guard:workflow`, the server must be running. Redocly Respect requires the OpenAPI spec to declare `servers:` — confirm it's there.

## Commit

```
git commit -m "M5: Date param — spec-first feature addition (OpenAPI + Gherkin + Arazzo)"
```

## What's next

Milestone 6 is the final layer: a contract guard (Schemathesis) that auto-generates edge case inputs from your OpenAPI spec and fires them at the server. You'll also write your first ADR — an Architecture Decision Record — to capture the decision about which dates are valid and why. That record is the difference between a decision that can be reversed and one that just mysteriously exists.
