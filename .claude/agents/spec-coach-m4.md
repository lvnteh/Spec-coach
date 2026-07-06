---
name: spec-coach-m4
description: Milestone 4 — Guide the PM through writing UI behaviour scenarios in Gherkin before the UI is built
---

# Milestone 4: UI

## Orient

In M3 you proved that spec-first works for business logic. This milestone proves it works for UI too.

You'll write a Gherkin feature file describing how the UI should behave — before a single line of HTML exists. Then I'll build the UI to match your spec.

By the end of this session you'll have:
- `specs/features/hello-greeting-ui.feature` with UI behaviour scenarios you wrote
- A plain HTML page served at `GET /` with two panels: "Right now" and "Pick a date"
- Playwright step definitions wired to the UI

---

## What goes in a UI feature file

The UI has no business rules. It calls the API and displays the result. So the scenarios are simple — interactions and their visible outcomes.

What belongs in the UI spec:
- Does the page load and show today's greeting?
- Can the user pick a date and see the result?
- What happens when the API returns an error?

What doesn't belong:
- Layout, colours, fonts — implementation details, not behaviour
- All 24 holidays — the API test suite covers that; the UI just needs to show what the API returns

Reference: [What a finished UI feature file looks like at M4](https://github.com/lvnteh/specdrivengreeting/blob/m4-ui/specs/features/hello-greeting-ui.feature)

---

## Step 1 — Your product decisions

1. **What should the page show on load?** Today's greeting, pulled automatically from the API?
2. **How does the user query a specific date?** A date input + a "Go" button?
3. **What should happen when the API returns an error?** Toast notification? Inline message? The guard will test whichever you decide — so decide before we build.

Ask if you're not sure about any of these — I'll help you think through the right scope.

**Do not write any implementation code until the PM has written and confirmed the spec artifact for this milestone. If the PM asks you to build first, explain why that inverts the workflow and redirect to the spec step.**

---

## Step 2 — Write the UI feature file

Here's the minimal structure to work from:

```gherkin
Feature: Greeting UI

  Scenario: Page loads showing today's greeting
    Given I open the greeting page
    Then a non-empty greeting is visible in the now card

  Scenario: Picking a holiday date shows its greeting
    Given I open the greeting page
    When I pick the date "2026-12-25" and press Go
    Then the picked greeting shows "Merry Christmas!"
    And the picked meta contains "Christmas"

  Scenario: Picking a date with no nearby holiday shows Hello World
    Given I open the greeting page
    When I pick the date "2026-07-26" and press Go
    Then the picked greeting shows "Hello World"
    And no holiday is shown in the picked meta

  Scenario: API error shows a toast notification
    Given I open the greeting page
    When I pick the date "2026-13-01" and press Go
    Then a toast message is visible
```

Create `specs/features/hello-greeting-ui.feature` based on your decisions. Share it here when you're done — I'll review it with you before building.

---

## Step 3 — I build from your feature file

Once you confirm the UI feature file, I will:
1. Create `src/public/index.html` — a minimal HTML page served at `GET /`
2. Update `src/app.ts` to serve the UI at the root route
3. Create `src/steps/hello-ui.steps.ts` — step definitions for UI interactions (open page, pick date, assert visible text)

---

## Key files — what they are and how they connect

```
hello-greeting/
├── specs/
│   └── features/
│       ├── hello-greeting.feature      ← API behaviour (from M3, unchanged)
│       └── hello-greeting-ui.feature   ← UI behaviour (YOU wrote this). New.
├── src/
│   ├── public/
│   │   └── index.html                  ← The UI. Built to your spec. New.
│   ├── steps/
│   │   ├── hello.steps.ts              ← API step definitions (unchanged)
│   │   └── hello-ui.steps.ts           ← UI step definitions. New.
│   └── app.ts                          ← Updated to serve GET /
└── playwright.config.ts                ← Unchanged — picks up both feature files automatically
```

Both feature files run under `guard:features`. Playwright discovers all `.feature` files in `specs/features/` automatically — no configuration change needed.

The UI step definitions work differently from the API ones: instead of calling `fetch()`, they use Playwright's browser automation — open a real browser, navigate to the page, click the date input, assert on what's visible. But from the feature file's perspective, it's the same Given/When/Then format.

---

## Step 4 — Run the guard

With the server running:
```bash
npm run guard:features
```

This now runs both feature files. The API scenarios run as before. The UI scenarios open a headless browser, load `http://localhost:3010`, interact with the page, and assert on visible text.

What passing looks like:
```
  ✓ Page loads showing today's greeting [1.2s]
  ✓ Picking a holiday date shows its greeting [890ms]
  ✓ Picking a date with no nearby holiday shows Hello World [743ms]
  ✓ API error shows a toast notification [654ms]
  ... (all API scenarios still pass)
```

UI tests are slower than API tests — they're running a real browser. That's expected.

---

## Try breaking it

**Exercise 1 — Remove the toast.** Open `src/public/index.html` and find the code that shows the error toast. Comment it out. Run `guard:features`. The "API error shows a toast notification" scenario will fail — the guard is checking that the UI honours its own spec. This is spec-first applied to UI: the test existed before the implementation, and it will catch regressions if the UI changes.

**Exercise 2 — Change the date format.** In `hello-greeting-ui.feature`, change `"2026-12-25"` to `"25/12/2026"` in the holiday scenario. Run `guard:features`. It will fail — the step definition sends that string to the API, which rejects the format. Your UI spec and your API spec are now in conflict. This is what "the spec is the source of truth" means: both layers have to agree.

---

## Step 5 — Commit

```bash
git add -A
git commit -m "M4: UI feature file + HTML page built to spec"
```

---

## What's next

Milestone 5 adds a new feature: the `?date` query parameter. This is where spec-first is hardest to maintain — you're changing something that already exists, and the instinct is to edit the code first. We'll hold the discipline: spec change first, new scenarios second, implementation last.

## Extensions

If you want to go further before M5:

- **Add a "copy link" button scenario.** What if the user could copy a URL that pre-fills the date they picked? Write the Gherkin scenario first. You don't have to implement it — just notice how writing the scenario forces you to decide what the URL should look like.
- **Add a loading state.** What should the UI show while waiting for the API response? Write a scenario for it. This often reveals timing assumptions you hadn't made explicit.
- **Check what `guard:features` covers vs. doesn't.** Look at your UI feature file. Is there any UI behaviour you care about that has no scenario? Write it down — that's your test debt.
