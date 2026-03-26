# Feature Acceptance Profile Guide

Feature Profiles are classified based on "**how to effectively verify acceptance**", not by feature type. The selection is based on when results are visible and whether a simulated environment is needed.

Machine-readable complete definitions can be found at `acceptance/registry.yml`.

---

## e2e — End-to-End Acceptance

Verifies complete user flows through the browser, with results visible in real-time. This is a single-point acceptance for task-shaped work, managed by atdd-hub.

| Item | Description |
|------|-------------|
| **Runner** | Chrome MCP |
| **Test Format** | Gherkin (Given-When-Then) |
| **Storage Location** | atdd-hub `tests/` or `acceptance/fixtures/` |

**Applicable when**:
- Results are immediately visible on screen
- Wait time < 60 seconds
- No time manipulation needed (freeze/travel)
- No external service mocking needed

**Not applicable when**:
- Results require waiting more than 1 minute
- Depends on specific time points (weekly settlement, monthly settlement)
- Depends on external service responses and is unstable
- Needs to simulate concurrency/Race Conditions

**Example scenarios**: Form submission updates page, button click shows Modal, search filters list, login redirects to page

---

## integration — Integration Acceptance

Verifies cross-component/cross-Domain business results through test frameworks. This is project-level regression testing (defensive wall), stored in each project repo, triggered automatically by CI/CD.

| Item | Description |
|------|-------------|
| **Runner** | RSpec / Jest |
| **Test Format** | RSpec / Jest |
| **Storage Location** | Each project's `spec/domains/**/integration/` |

**Applicable when**:
- Results require waiting more than 1 minute
- Time manipulation needed (freeze/travel_to)
- External service mocking needed
- Concurrency/Race Condition simulation needed
- Cross-Domain data flow verification
- Background jobs/scheduled tasks

**Example scenarios**: Upload file then wait for processing completion, weekly/monthly settlement schedules, external API integrations, cross-Domain data flows, background job processing

---

## calculation — Calculation/Logic Acceptance

Verifies backend calculation logic and business rules, no UI interaction. This is project-level regression testing, triggered automatically by CI/CD.

| Item | Description |
|------|-------------|
| **Runner** | RSpec / Jest |
| **Test Format** | RSpec / Jest |
| **Storage Location** | Each project's `spec/domains/**/` |

**Applicable when**:
- Pure backend logic changes
- Calculation formulas or business rules
- Entity/Service method changes
- No UI interaction
- Results need to be verified via Console or API

**Example scenarios**: Entity method logic changes (e.g., `b2b?` determination), tax calculations, business rule decisions (e.g., routing selection), data transformation logic, Service/UseCase changes without UI

---

## unit — Unit Acceptance

Verifies pure logic calculations, rules, and algorithm correctness. This is project-level regression testing, triggered automatically by CI/CD.

| Item | Description |
|------|-------------|
| **Runner** | RSpec / Jest |
| **Test Format** | RSpec / Jest |
| **Storage Location** | Each project's `spec/domains/**/unit/` |

**Applicable when**:
- Pure calculation logic (formulas, algorithms)
- Single business rule verification
- Value Object behavior
- Domain Service logic
- No I/O or external dependencies involved

**Test focus**: Formula correctness, boundary value handling, abnormal input handling, precision verification

**Example scenarios**: Amount calculation formulas, date format conversions, permission rule decisions, state machine transitions

---

## Profile Selection Decision Tree

```
Q1: Are results immediately visible on screen (< 60 seconds)?
    YES → e2e
    NO  ↓
Q2: Does it require time manipulation (weekly settlement, monthly settlement, delayed execution)?
    YES → integration
    NO  ↓
Q3: Does it depend on external services and require mocking?
    YES → integration
    NO  ↓
Q4: Is it a backend logic change with no UI interaction, requiring Console verification?
    YES → calculation
    NO  ↓
Q5: Is it pure calculation/rule logic?
    YES → unit
    NO  → integration
```
