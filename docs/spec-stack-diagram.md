# The Spec Stack

How the layers of a spec-driven project connect at the end of Milestone 6.

```
┌─────────────────────────────────────────────────────────────────┐
│                         PM-OWNED SPECS                          │
│                                                                 │
│  specs/openapi.yaml           ← API contract                   │
│  specs/features/*.feature     ← Behaviour scenarios (Gherkin)  │
│  specs/workflows/*.arazzo.yaml ← End-to-end flow specs         │
└─────────────────────────┬───────────────────────────────────────┘
                          │ derives
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│                       GENERATED (never edit)                    │
│                                                                 │
│  src/generated/types/         ← TypeScript interfaces (Kubb)   │
│  src/generated/zod/           ← Zod schemas (Kubb)             │
│  src/routes.ts                ← Hono routes (from OpenAPI)     │
└─────────────────────────┬───────────────────────────────────────┘
                          │ wires into
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│                   AI-IMPLEMENTED (only file AI touches)         │
│                                                                 │
│  src/implementation/handlers.ts   ← Business logic             │
└─────────────────────────────────────────────────────────────────┘

GUARDS — run against the live server

  guard:typecheck   tsc --noEmit         Catches type mismatches
  guard:features    Playwright BDD       Business rules (PM wrote these)
  guard:contract    Schemathesis         Protocol conformance (auto-generated)
  guard:workflow    Redocly Respect      End-to-end status codes

DECISION TRAIL

  docs/decisions/ADR-*.md   ← Why decisions were made
  docs/rfcs/RFC-*.md        ← Open questions and options
  docs/status.md            ← Current project health snapshot
```

Owner per layer:

| Layer | Owner | Can change? |
|-------|-------|-------------|
| openapi.yaml | PM + Engineer | Yes — drives all other changes |
| feature files | PM | Yes — PM owns behaviour decisions |
| Arazzo workflows | Engineer | Yes — describes flow logic |
| src/generated/ | Kubb (auto) | No — regenerate after spec changes |
| src/routes.ts | Framework | No — derived from spec |
| src/implementation/handlers.ts | AI / Engineer | Yes — only implementation file |
