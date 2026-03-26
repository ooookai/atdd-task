# Fix Acceptance Profile Guide

Fix Profiles use a **two-stage classification**:

1. **Fix Profile** (determined at task creation) — Classified by problem symptoms, determines initial investigation tools
2. **Affected Layer** (determined after investigation) — Classified by architecture layer, determines fix approach and test type

Machine-readable complete definitions can be found at:
- `acceptance/fix-profiles.yml` — Fix Profiles and Affected Layers
- `acceptance/fix-discovery-flows.yml` — 14 Discovery Source investigation workflows

---

## 7 Fix Profiles

### ui — Page/UI Issues

Display or interaction issues discovered from the page.

| Item | Description |
|------|-------------|
| **Symptoms** | Screen, display, buttons, forms, layout breaks, CSS, frontend |
| **Discovery Sources** | D1 (View Layer), D2 (data-related), D3 (flow error), D19 (cross-browser/device) |
| **Possible Affected Layers** | Presentation, Application, Domain |

**Investigation workflows**:

| Source | Flow |
|--------|------|
| D1 Page display error (View Layer) | Browser → Read Code → (?) fix / Git History → (?) fix / return to human |
| D2 Page display error (data-related) | Browser → Read Code → (?) / Server Log → (?) / Rails Runner → (?) / Domain Context → (?) / ask Migrate → Local Debug → (?) / Git History → (?) / return to human |
| D3 Page flow error | Browser → Read Spec → Read Code → (?) / Server Log → Domain Context → (?) / Git History → (?) / return to human |
| D19 Cross-browser/device issue | Browser(problem env) → Browser(reference) → Read Code → (?) / Git History → (?) / return to human |

---

### data — Data Issues

Data errors discovered from reports or databases.

| Item | Description |
|------|-------------|
| **Symptoms** | Data, reports, numbers, statistics, missing, duplicate, inconsistent |
| **Discovery Sources** | D4 (data error), D14 (data duplication/loss) |
| **Possible Affected Layers** | Domain, Infrastructure |

**Investigation workflows**:

| Source | Flow |
|--------|------|
| D4 Data error | Read Code → (?) / Rails Runner → Server Log → (?) / Domain Context → (?) / Migrate → Local concurrency test → (?) / return to human |
| D14 Data duplication/loss | Read Code → (?) / Rails Runner → (?) / Server Log → (?) / Domain Context → RDS/Redis Log → (?) / Migrate → Local concurrency test → (?) / return to human |

---

### worker — Background Task Issues

Task errors discovered from Sidekiq/scheduler interfaces.

| Item | Description |
|------|-------------|
| **Symptoms** | Sidekiq, Job, Queue, Worker, scheduler, background, async |
| **Discovery Sources** | D5 (data-related), D6 (code issue) |
| **Possible Affected Layers** | Application, Domain, Infrastructure |

**Investigation workflows**:

| Source | Flow |
|--------|------|
| D5 Queue/schedule error (data-related) | App Status → Error Tracking → Read Code → (?) / Rails Runner → (?) / Server Log → (?) / Domain Context → (?) / Migrate → Local Sidekiq → (?) / Git History → (?) / return to human |
| D6 Queue/schedule error (code issue) | App Status → Error Tracking → Read Code → (?) / Local Sidekiq → Server Log → (?) / Git History → (?) / return to human |

---

### performance — Performance Issues

Performance issues discovered from monitoring or user reports.

| Item | Description |
|------|-------------|
| **Symptoms** | Slow, timeout, performance, memory, CPU, N+1, slow query |
| **Discovery Sources** | D8 (APM performance alert), D9 (user-reported slowness) |
| **Possible Affected Layers** | Presentation, Application, Domain, Infrastructure |

**Investigation workflows**:

| Source | Flow |
|--------|------|
| D8 APM performance alert | APM → Read Code → (?) / branch(DB: RDS Log / Ruby: analyze logic / External: Server Log / Memory: analyze logic) → (?) / Git History → (?) / return to human |
| D9 User-reported slowness | Browser → branch(frontend slow: DevTools / backend slow: APM / cannot reproduce: Server Log) → Read Code → Local Benchmark → (?) / return to human |

---

### integration — Integration Issues

Errors related to third-party service integrations.

| Item | Description |
|------|-------------|
| **Symptoms** | API, integration, payment, ERP, Webhook, third-party, external |
| **Discovery Sources** | D10 (external service error) |
| **Possible Affected Layers** | Infrastructure |

**Investigation workflows**:

| Source | Flow |
|--------|------|
| D10 External service error | Error Tracking → Server Log → Read Code → (?) / branch(Request error / Response handling error / Intermittent: Script test connection) → (?) / Domain Context → (?) / return to human |

---

### alert — Monitoring Alert Issues

Errors discovered from logs or monitoring systems.

| Item | Description |
|------|-------------|
| **Symptoms** | Log, Error, Alert, monitoring, Rollbar, Sentry |
| **Discovery Sources** | D7 (log record error) |
| **Possible Affected Layers** | Presentation, Application, Domain, Infrastructure |

**Investigation workflows**:

| Source | Flow |
|--------|------|
| D7 Log record error | Server Log → Error Tracking → Read Code → (?) / Rails Runner → (?) / Git History → Read recent deployment → (?) / return to human |

---

### security — Security Issues

Permission errors or security vulnerabilities.

| Item | Description |
|------|-------------|
| **Symptoms** | Permission, login, authorization, security, XSS, injection |
| **Discovery Sources** | D12 (permission error), D13 (security scan) |
| **Possible Affected Layers** | Presentation, Application, Infrastructure |

**Investigation workflows**:

| Source | Flow |
|--------|------|
| D12 Permission error | Read Code → Read Feature Spec → (?) / Rails Runner → (?) / Domain Context → (?) / return to human |
| D13 Security scan | Read Code → branch(Dependency: update requires human confirmation / Code: locate vulnerability) → (?) / return to human |

---

## Affected Layer and Test Type Mapping

After investigation determines the Affected Layer, the fix approach and test type are decided:

| Affected Layer | Aliases | Test Type | Test Focus |
|----------------|---------|-----------|------------|
| **Presentation** | View / UI / Frontend | system | Correct page display, correct interaction behavior |
| **Application** | Controller / Use Case / Service | integration | Correct workflow execution, correct error handling, correct permission checks |
| **Domain** | Model / Business Logic / Entity | unit | Correct calculations, business rules compliance, boundary condition handling |
| **Infrastructure** | Repository / Adapter / External | integration | Correct data access, correct external service calls, correct error handling |

---

## Common Principles

### Confidence Threshold

- **Confidence ≥ 95%** → Proceed with fix
- **Confidence < 95%** → Return to human, provide investigation results and suggestions

When returning to human, must provide: current investigation results, tools used and findings, possibilities already ruled out, specific issues that couldn't be clarified, suggested next steps.

### Production is Immutable

- **Query tools** (Browser, AWS Connect, Read, Error Tracking, APM) — Read-only, no state changes
- **Command tools** (Write, Bash, Migrate, Git, Debugger) — Can change state, **Local/Staging only**
- Direct writes to Production database are prohibited
- Reverse migration from Local/Staging to Production is prohibited

### TDD Workflow

Fix tasks follow TDD-driven development:

```
PreTest (red) → Fix implementation → Test (green)
```

1. Determine test type based on Affected Layer
2. Write PreTest, expected to fail (red)
3. Fix the code
4. Run tests to confirm pass (green)
5. Run related tests to confirm no regression

### Investigation Flow Symbol Legend

| Symbol | Meaning |
|--------|---------|
| `→` | Next step |
| `(?) fix` | If clarified (confidence ≥ 95%) then fix |
| `/` | Otherwise continue to next step |
| `ask` | Requires human confirmation before executing |
| `branch(...)` | Choose path based on situation |
| `return to human` | Insufficient confidence, report investigation results to human |
