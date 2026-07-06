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
