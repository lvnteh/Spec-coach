# Spec-Coach

A guided journey through spec-driven development for product managers.

You will build a real, working API — a seasonal greeting service that returns the right holiday greeting for whatever date you ask about. You'll define what it does, write the tests, and watch it get built to your spec.

Claude Code is your guide. It explains each concept, asks you to make the product decisions, and only writes code after you've written the spec.

## What you'll build

A `GET /hello` API that:
- Returns the nearest holiday greeting within a 10-day window
- Falls back to "Hello World" when no holiday is nearby
- Accepts an optional `?date` parameter to test any date you choose
- Has a UI with a date picker

You build it milestone by milestone, adding one layer of the spec stack at each step.

## Prerequisites

[Claude Code](https://claude.ai/code) installed. Nothing else.

## How to start

```bash
git clone https://github.com/lvnteh/Spec-coach
cd Spec-coach
claude --agent spec-coach-m1 "start"
```

## The six milestones

| # | What you'll do | What you'll have at the end |
|---|---------------|----------------------------|
| M1 | Write an OpenAPI contract for a Hello World endpoint | A working API server, TypeScript types generated from your spec |
| M2 | Write test cases for the holiday calendar logic | A PM-owned `cases.json` that acts as an executable spec |
| M3 | Promote your test cases to Gherkin feature files | A living, readable BDD spec that tests the real HTTP server |
| M4 | Write UI behaviour scenarios before the UI exists | A working HTML UI with Playwright tests |
| M5 | Add a date picker feature — spec first | An updated contract, new scenarios, new implementation |
| M6 | Add a contract guard and document your decisions | Schemathesis running, ADR and RFC written |

## Reference project

[specdrivengreeting](https://github.com/lvnteh/specdrivengreeting) is the answer key. Each milestone is tagged (`m1-hello-world` through `m6-contract`) so you can see what the project looks like at any stage. Look at it after you've written your own version — not before.

