# Fix Task Workflow

> Fix tasks are used to fix bugs, using a simplified workflow focused on quickly locating and resolving issues.

## Overview

Differences between Fix and Feature tasks:

| Item | Feature | Fix |
|------|---------|-----|
| Workflow | requirement → specification → testing → development → review → gate | requirement → testing → development → review → gate |
| Goal | Implement new features | Fix existing issues |
| Confidence threshold | 90% | 80% (quick entry to fix) |
| Investigation tools | None | 14 Discovery Source workflows |

## Starting a Fix Task

```bash
/fix {project_id}, {issue description}
```

**Examples**:
```bash
/fix sf_project, Invoice amount shows NaN
/fix core_web, Login page doesn't render on Safari
```

## Fix Phase Workflow

```
requirement → testing → development → review → gate
    ↓           ↓           ↓           ↓       ↓
 specist     tester      coder    reviewers  gatekeeper
```

### 1. Requirement Phase (Specist)

Specist is responsible for:
1. **Identifying Discovery Source (Dx)** — Determine which Dx based on the issue description
2. **Executing investigation workflow** — Follow the corresponding Dx's tool workflow
3. **Confirming Affected Layer** — Determine which architecture layer the issue resides in
4. **Reaching confidence** — Must reach 80% or above to proceed to the next phase

### 2. Testing Phase (Tester)

Tester is responsible for:
1. **Writing failing tests** — Write tests expected to fail based on the issue description
2. **Confirming red tests** — Tests must fail first, proving the issue exists
3. **Preparing acceptance criteria** — Define expected behavior after the fix

### 3. Development Phase (Coder)

Coder is responsible for:
1. **Fixing the issue** — Fix the code based on investigation results
2. **Green tests** — Confirm tests pass
3. **Regression testing** — Confirm no other features are affected

### 4. Review Phase (Reviewers)

- **Risk Reviewer**: Checks security/performance/risk
- **Style Reviewer**: Checks code style (only required for Refactor tasks)

### 5. Gate Phase (Gatekeeper)

Gatekeeper is responsible for:
1. **Final quality confirmation**
2. **Generating reports**
3. **Knowledge update suggestions**

## Discovery Source (Dx) Investigation Workflows

### Flow Symbol Legend

| Symbol | Description |
|--------|-------------|
| `→` | Next step |
| `(?) fix` | If clarified (confidence ≥ 95%) then fix |
| `/` | Otherwise continue to next step |
| `ask` | Requires human confirmation before executing |
| `branch(...)` | Choose path based on situation |
| `return to human` | Insufficient confidence, report investigation results |

### UI Profile

| Dx | Name | Flow |
|----|------|------|
| D1 | Page display error (View Layer) | Browser → Read Code → (?) fix / Git History → (?) fix / return to human |
| D2 | Page display error (data-related) | Browser → Read Code → (?) fix / Server Log → (?) fix / Rails Runner → (?) fix / Domain Context → (?) fix / ask Migrate → Local Debug → (?) fix / Git History → (?) fix / return to human |
| D3 | Page flow error | Browser → Read Spec → Read Code → (?) fix / Server Log → Domain Context → (?) fix / Git History → (?) fix / return to human |
| D19 | Cross-browser/device issue | Browser(problem env) → Browser(reference) → Read Code → (?) fix / Git History → (?) fix / return to human |

### Data Profile

| Dx | Name | Flow |
|----|------|------|
| D4 | Data error | Read Code → (?) fix / Rails Runner → Server Log → (?) fix / Domain Context → (?) fix / Migrate → Local concurrency test → (?) fix / return to human |
| D14 | Data duplication/loss | Read Code → (?) fix / Rails Runner → (?) fix / Server Log → (?) fix / Domain Context → RDS/Redis Log → (?) fix / Migrate → Local concurrency test → (?) fix / return to human |

### Worker Profile

| Dx | Name | Flow |
|----|------|------|
| D5 | Queue/schedule error (data-related) | App Status → Error Tracking → Read Code → (?) fix / Rails Runner → (?) fix / Server Log → (?) fix / Domain Context → (?) fix / Migrate → Local Sidekiq → (?) fix / Git History → (?) fix / return to human |
| D6 | Queue/schedule error (code issue) | App Status → Error Tracking → Read Code → (?) fix / Local Sidekiq → Server Log → (?) fix / Git History → (?) fix / return to human |

### Performance Profile

| Dx | Name | Flow |
|----|------|------|
| D8 | APM performance alert | APM → Read Code → (?) fix / branch(DB: RDS Log / Ruby: analyze logic / External: Server Log / Memory: analyze logic) → (?) fix / Git History → (?) fix / return to human |
| D9 | User-reported slowness | Browser → branch(frontend slow: DevTools / backend slow: APM / cannot reproduce: Server Log) → Read Code → Local Benchmark → (?) fix / return to human |

### Integration Profile

| Dx | Name | Flow |
|----|------|------|
| D10 | External service error | Error Tracking → Server Log → Read Code → (?) fix / branch(Request error / Response handling error / Intermittent: Script test connection) → (?) fix / Domain Context → (?) fix / return to human |

### Alert Profile

| Dx | Name | Flow |
|----|------|------|
| D7 | Log record error | Server Log → Error Tracking → Read Code → (?) fix / Rails Runner → (?) fix / Git History → Read recent deployment → (?) fix / return to human |

### Security Profile

| Dx | Name | Flow |
|----|------|------|
| D12 | Permission error | Read Code → Read Feature Spec → (?) fix / Rails Runner → (?) fix / Domain Context → (?) fix / return to human |
| D13 | Security scan | Read Code → branch(Dependency: update requires human confirmation / Code: locate vulnerability) → (?) fix / return to human |

## Tool Categories

### Query Tools (Read-only)

| Tool | Description |
|------|-------------|
| browser | Chrome browser operations |
| aws_connect.rails_runner | Remote Rails Runner (query only) |
| aws_connect.server_log | Remote log queries |
| aws_connect.app_status | Sidekiq/Redis status |
| aws_connect.rds_log | RDS/Redis logs |
| aws_connect.script | Upload and execute scripts (deleted after use) |
| read.local_codebase | Read source code |
| read.domain_context | Read Domain documents |
| read.git_history | Read Git history |
| error_tracking | Sentry/Rollbar (manual operation) |
| apm | NewRelic/Datadog (manual operation) |

### Command Tools (Can change state, Local/Staging only)

| Tool | Description |
|------|-------------|
| write | Modify code |
| bash.rspec | Run tests |
| bash.rails_runner | Local Rails Runner |
| bash.sidekiq_local | Local Sidekiq |
| bash.benchmark | Local performance testing (Skill in development) |
| migrate | Data migration (Production → Local) |
| git_command | Git write operations |
| debugger | Debugging tools |

## Core Principles

1. **Production data is immutable** — All Command operations are limited to Local/Staging
2. **Rails Runner must come after Read Code** — You can't know what to run without reading the code first
3. **Git History is the last resort** — Exhaust all debugging methods before questioning git versions
4. **Confidence ≥ 95% before fixing** — When uncertain, return to human for confirmation

## Information Required When Returning to Human

When confidence < 95% and returning to human, must provide:
1. Current investigation results
2. Tools used and findings
3. Possibilities already ruled out
4. Specific issues that couldn't be clarified
5. Suggested next steps

## Post-Review Fixes

When Review finds issues, you can choose to fix rather than going directly to Gate:

```bash
/fix-critical   # Fix Critical issues
/fix-high       # Fix Critical + High issues
/fix-all        # Fix all issues (including suggestions)
```

Fix workflow (TDD):
```
review → testing (add tests) → development (fix) → review → gate
```

## Related Files

- `fix-profiles.yml` — Profile definitions and Affected Layers
- `fix-tools.yml` — Tool list (Query/Command categories)
- `fix-discovery-flows.yml` — 14 Discovery Source investigation workflows
- `registry.yml` — Acceptance configuration registry
