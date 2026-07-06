---
name: spec-coach-m2
description: Milestone 2 — Guide the PM through extending the OpenAPI spec and writing cases.json as an executable behaviour spec
---

# Milestone 2: Holiday Greetings

## Orient

In M1 you built a contract and a Hello World server. The API works, but it always returns the same thing. In this milestone you define the holiday logic — which holidays exist, what greeting each one gets, and how the proximity window works.

Your main artifact is `cases.json` — a list of expected inputs and outputs. This is the first time you'll write a spec that describes *behaviour*, not just *shape*. It's also the first spec that belongs entirely to you as a PM: no YAML syntax to learn, just JSON objects with descriptions.

By the end of this session you'll have:
- An extended `openapi.yaml` with the full response shape (`holiday`, `daysUntil` fields)
- A `cases.json` with at least 10 holiday scenarios you wrote
- A running guard that executes your scenarios against the implementation

## What is cases.json, and why does a PM care?

A test case is just a statement: "given this input, the output must be exactly this." Written as JSON, it looks like:

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

That's it. No code. No framework. Just a description, an input, and the exact expected output.

`cases.json` is a list of these. It becomes executable — there's a runner (`guard:cases`) that calls the greeting logic with each input and checks whether the output matches. If you change the implementation and a case breaks, you know immediately.

This is what "the spec is executable" means. Your cases.json isn't documentation that might drift — it's a check that runs every time.

Reference: [What a finished cases.json looks like at M2](https://github.com/lvnteh/specdrivengreeting/blob/m2-holiday-greetings/specs/cases.json)

## Your decisions

Before writing cases.json, you need to decide the rules. Answer these:

1. **Which holidays should the API support?** Start with at least 5. The reference project has 24 for 2026 — you can use that list or define your own.
2. **What greeting does each holiday get?** "Merry Christmas!", "Happy Halloween!", etc. These will be asserted verbatim — typos matter.
3. **What is the proximity window?** How many days before or after a holiday should still return that holiday's greeting? The reference uses 10 days.
4. **What happens when two holidays are equidistant?** (e.g., you're 5 days away from both Halloween and Día de los Muertos.) Which one wins — the upcoming one or the past one?
5. **What does the response look like for the fallback?** When no holiday is nearby, the reference returns `{ "greeting": "Hello World", "holiday": null, "daysUntil": null }`. Confirm this shape.

I'll wait for your decisions before we write anything.

**Do not write any implementation code until the PM has written and confirmed the spec artifact for this milestone. If the PM asks you to build first, explain why that inverts the workflow and redirect to the spec step.**

## Extending the OpenAPI spec first

Your response schema needs two new fields: `holiday` (nullable string) and `daysUntil` (nullable integer). Add them to `openapi.yaml` before writing cases.

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

Update your `specs/openapi.yaml` with this schema, then run `npm run generate` to regenerate the types. The spec changes first — then generation — then implementation.

## Writing your cases.json

Create `hello-greeting/specs/cases.json`. Each entry needs:
- `description` — a plain-English label
- `input.date` — a date in `YYYY-MM-DD` format
- `expected.greeting` — exact string
- `expected.holiday` — exact string or `null`
- `expected.daysUntil` — integer or `null`

Write at least:
- One case per holiday (today IS the holiday → daysUntil: 0)
- One case 5 days before a holiday
- One case 11 days before a holiday (outside window → fallback)
- One fallback case (no holiday nearby)

Share the file when you're done and I'll review it.

## What I'll build from your cases

Once you confirm cases.json, I will:
1. Implement the holiday table and `findGreeting()` logic in `src/implementation/handlers.ts`
2. Create `src/guard-cases.ts` — the runner that executes your cases against the implementation

## Running the guard

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
That means `findGreeting()` didn't find Halloween 5 days out. The fix is in the handler's window logic.

## Commit

```
git commit -m "M2: Holiday greetings — extended OpenAPI schema + cases.json"
```

## What's next

Milestone 3 promotes your cases.json to Gherkin. You'll rewrite the same scenarios in a `.feature` file — a format that's more readable, more expressive, and that tests the real HTTP server rather than calling functions directly. You'll also discover edge cases (boundary days, tie-breaks) that are easy to add in Gherkin but awkward in JSON.
