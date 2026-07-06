---
name: spec-coach-m4
description: Milestone 4 — Guide the PM through writing UI behaviour scenarios in Gherkin before the UI is built
---

# Milestone 4: UI

## Orient

In M3 you proved that the spec-first discipline applies to business logic. This milestone proves it applies to UI too.

You'll write a Gherkin feature file describing how the UI should behave — before a single line of HTML exists. Then I'll build the UI to match your spec.

By the end of this session you'll have:
- `specs/features/hello-greeting-ui.feature` with UI behaviour scenarios
- A plain HTML page served at `GET /` with two panels: "Right now" and "Pick a date"
- Playwright step definitions wired to the UI

## What goes in a UI feature file

The UI has no business rules. It calls the API and displays the result. So the scenarios are simple — they describe interactions and their visible outcomes, not logic.

What belongs in the UI spec:
- Does the page load and show today's greeting?
- Can the user pick a date and see the greeting for that date?
- What happens when the API returns an error?

What doesn't belong:
- Layout, colours, fonts — implementation details, not behaviour
- All 24 holidays — the API tests cover that; the UI just needs to display what the API returns

Reference: [What a finished UI feature file looks like at M4](https://github.com/lvnteh/specdrivengreeting/blob/m4-ui/specs/features/hello-greeting-ui.feature)

## Your decisions

1. **What should the page show on load?** Today's greeting, pulled from the API automatically?
2. **How does the user query a specific date?** A date input field + a "Go" button?
3. **What should happen if the API returns an error?** A toast notification? Inline message?

**Do not write any implementation code until the PM has written and confirmed the spec artifact for this milestone. If the PM asks you to build first, explain why that inverts the workflow and redirect to the spec step.**

## The minimal feature file structure

```gherkin
Feature: Greeting UI

  Scenario: Page loads showing today's greeting
    Given I open the greeting page
    Then a non-empty greeting is visible

  Scenario: Picking a holiday date shows its greeting
    Given I open the greeting page
    When I pick the date "2026-12-25" and press Go
    Then the greeting shows "Merry Christmas!"

  Scenario: API error shows a notification
    Given I open the greeting page
    When I pick the date "2026-13-01" and press Go
    Then an error notification is visible
```

Write `specs/features/hello-greeting-ui.feature` based on your decisions. Share it when you're done.

## What I'll build from your feature file

Once you confirm the UI feature file, I will:
1. Create `src/public/index.html` — a minimal HTML page served at `GET /`
2. Add the UI route to `src/app.ts`
3. Create `src/steps/hello-ui.steps.ts` — step definitions for UI interactions

## Running the guard

With the server running:
```bash
npm run guard:features
```

This now runs both the API feature file and the UI feature file. The UI tests open a real browser (headless), load the page, interact with it, and assert on what's visible.

## Commit

```
git commit -m "M4: UI feature file + HTML page built to spec"
```

## What's next

Milestone 5 adds a new feature: the `?date` query parameter that lets anyone test any date via the API directly (not just through the UI). The critical lesson: you'll change the OpenAPI spec first, write the new Gherkin scenarios second, and only then touch the implementation. This is spec-first applied to a change, not just a greenfield build.
