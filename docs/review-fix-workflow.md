# Post-Review Fix Workflow Design

This document describes how to perform fixes through a TDD workflow when the Review phase discovers issues, and the knowledge transfer mechanism between Agents.

---

## Table of Contents

1. [Design Background](#design-background)
2. [Workflow Overview](#workflow-overview)
3. [Commands Definition](#commands-definition)
4. [reviewFindings Data Structure](#reviewfindings-data-structure)
5. [Agent Knowledge Transfer](#agent-knowledge-transfer)
6. [Implementation Example](#implementation-example)

---

## Design Background

### Problem

The original workflow had only one option after the Review phase was completed: `/continue` to enter the Gate phase. However, Review may discover issues that need fixing, and the original workflow didn't handle this branch.

### Goals

1. Provide multiple fix options (by issue severity)
2. Use TDD workflow for fixes (write tests first, then implement)
3. Transfer knowledge through task JSON to avoid knowledge loss between Agents

---

## Workflow Overview

```
                    review completed
                         │
         ┌───────────────┼───────────────┐
         │               │               │
         ▼               ▼               ▼
   /continue        /fix-critical    /fix-all
         │               │               │
         ▼               │               │
       gate              │               │
                         ▼               ▼
                  ┌────────────────────────┐
                  │  Update fixScope       │
                  │  Return to testing     │
                  └──────────┬─────────────┘
                             │
                             ▼
                  ┌────────────────────────┐
                  │  tester adds tests     │
                  │  Reads reviewFindings  │
                  │  Generates failing     │
                  │  tests (red)           │
                  └──────────┬─────────────┘
                             │
                             ▼
                  ┌────────────────────────┐
                  │  coder fixes           │
                  │  Reads reviewFindings  │
                  │  Implements to pass    │
                  │  tests                 │
                  └──────────┬─────────────┘
                             │
                             ▼
                  ┌────────────────────────┐
                  │  Return to review      │
                  │  (simplified review)   │
                  └──────────┬─────────────┘
                             │
                             ▼
                           gate
```

### Phase Description

| Step | Phase Status | Agent | Description |
|------|-------------|-------|-------------|
| 1 | review → testing | - | Set fixScope, update status |
| 2 | testing | tester | Read reviewFindings, add tests |
| 3 | development | coder | Read reviewFindings + tests, fix |
| 4 | review | risk-reviewer | Simplified review (only verify fixed items) |
| 5 | gate | gatekeeper | Final quality control |

---

## Commands Definition

### /fix-critical

Fixes issues with severity `critical`.

**Behavior**:
1. Set `context.reviewFindings.fixScope = "critical"`
2. Update `status = "testing"`
3. Call tester agent

**Use case**: Security vulnerabilities, data integrity issues

---

### /fix-high

Fixes issues with severity `critical` and `high`.

**Behavior**:
1. Set `context.reviewFindings.fixScope = "high"`
2. Update `status = "testing"`
3. Call tester agent

**Use case**: Includes performance issues, N+1 queries

---

### /fix-all

Fixes issues of all severities (including suggestions).

**Behavior**:
1. Set `context.reviewFindings.fixScope = "all"`
2. Update `status = "testing"`
3. Call tester agent

**Use case**: Comprehensive code quality improvement

---

## reviewFindings Data Structure

### Complete Structure

```json
{
  "context": {
    "reviewFindings": {
      "fixScope": null,
      "styleReview": {
        "score": "92/100",
        "grade": "A",
        "issues": [],
        "suggestions": [
          {
            "id": "STYLE-001",
            "severity": "suggestion",
            "category": "readability",
            "file": "repositories/accounts_receivable.rb",
            "line": "71-88",
            "title": "Complex conditional logic",
            "description": "should_clear_invoice_number? logic is complex",
            "suggestion": "Extract partial sub-conditions as named methods",
            "example": null
          }
        ]
      },
      "riskReview": {
        "riskLevel": "High",
        "findings": [
          {
            "id": "SEC-001",
            "severity": "critical",
            "category": "security",
            "file": "use_cases/void_current_invoice.rb",
            "line": "7-14",
            "title": "Missing authorization control",
            "description": "Anyone who knows the serial can void an invoice",
            "impact": "Horizontal/vertical privilege escalation risk",
            "suggestion": "Add current_user parameter, check permissions",
            "example": "yield authorize_user(receivable, current_user)",
            "testHint": "Test different roles, different projects' permissions"
          },
          {
            "id": "SEC-002",
            "severity": "critical",
            "category": "concurrency",
            "file": "use_cases/void_current_invoice.rb",
            "line": "7-14",
            "title": "Insufficient concurrency control",
            "description": "Possible Race Condition",
            "suggestion": "Add Redis Lock or Optimistic Locking",
            "example": "with_redis_lock(lock_key) { ... }",
            "testHint": "Use Thread to simulate concurrent voiding"
          },
          {
            "id": "SEC-003",
            "severity": "critical",
            "category": "validation",
            "file": "use_cases/void_current_invoice.rb",
            "line": "8",
            "title": "Missing input validation",
            "description": "void_reason is not validated",
            "suggestion": "Validate not empty, length limit 5-500 characters",
            "example": "return Failure('Void reason cannot be empty') if void_reason.blank?",
            "testHint": "Test empty values, overly long strings, special characters"
          }
        ]
      }
    }
  }
}
```

### Field Descriptions

| Field | Type | Description |
|-------|------|-------------|
| `fixScope` | string | Fix scope: `null`/`critical`/`high`/`all` |
| `severity` | string | Severity: `critical`/`high`/`medium`/`low`/`suggestion` |
| `category` | string | Issue category: `security`/`concurrency`/`validation`/`performance`/`readability` |
| `file` | string | File where issue is located (relative to domain) |
| `line` | string | Line number where issue is located |
| `title` | string | Issue title (brief) |
| `description` | string | Issue description (detailed) |
| `impact` | string | Impact description (optional) |
| `suggestion` | string | Fix suggestion |
| `example` | string | Code example (optional) |
| `testHint` | string | Test suggestion (for tester use) |

### Severity Filtering Logic

| fixScope | Included severities |
|----------|-------------------|
| `critical` | critical |
| `high` | critical, high |
| `all` | critical, high, medium, low, suggestion |

---

## Agent Knowledge Transfer

### Tester Agent

When entering the testing phase (fix mode), the tester's prompt:

```
Project: {project}
Task: Add test cases to cover issues found in review
Task JSON: {task_json_path}
Mode: fix-review

Please execute:
1. Read context.reviewFindings from the task JSON
2. Filter issues to handle based on fixScope
3. Generate test cases for each issue
   - Use testHint as a test design reference
   - Tests should fail first (red)
4. Run tests to confirm failure
5. Update context.testFiles to include new test files

Output format should follow the tester agent's standard format.
```

### Coder Agent

When entering the development phase (fix mode), the coder's prompt:

```
Project: {project}
Task: Fix issues found in review, make tests pass
Task JSON: {task_json_path}
Mode: fix-review

Please execute:
1. Read context.reviewFindings from the task JSON
2. Read context.testFiles to understand test cases
3. Filter issues to fix based on fixScope
4. Fix each issue sequentially
   - Use suggestion and example as implementation references
   - Ensure tests pass
5. Update context.modifiedFiles

Output format should follow the coder agent's standard format.
```

### Knowledge Flow Diagram

```
┌─────────────────┐
│ risk-reviewer   │
│ style-reviewer  │
└────────┬────────┘
         │ Write reviewFindings
         ▼
┌─────────────────┐
│ Task JSON       │
│ (context)       │
└────────┬────────┘
         │ Read reviewFindings
    ┌────┴────┐
    ▼         ▼
┌───────┐ ┌───────┐
│tester │ │coder  │
└───────┘ └───────┘
```

---

## Implementation Example

### Example: Fixing Authorization Control Issue

#### 1. reviewFindings Content

```json
{
  "id": "SEC-001",
  "severity": "critical",
  "category": "security",
  "file": "use_cases/void_current_invoice.rb",
  "line": "7-14",
  "title": "Missing authorization control",
  "description": "Anyone who knows the serial can void an invoice",
  "suggestion": "Add current_user parameter, check permissions",
  "example": "yield authorize_user(receivable, current_user)",
  "testHint": "Test different roles, different projects' permissions"
}
```

#### 2. Tests Generated by Tester

```ruby
# spec/domains/accounting/accounts_receivable/use_cases/void_current_invoice_spec.rb

describe 'Scenario: Authorization control (SEC-001)' do
  describe 'When user lacks project permission' do
    let(:other_project_user) { create(:user, :accountant) }

    it 'returns failure result' do
      result = use_case.call(
        serial: 'AR-2025-001',
        void_reason: 'Test void',
        current_user: other_project_user
      )

      expect(result).to be_failure
      expect(result.failure).to eq('No permission to operate on this project')
    end
  end

  describe 'When user role does not allow voiding' do
    let(:viewer_user) { create(:user, :viewer) }

    it 'returns failure result' do
      result = use_case.call(
        serial: 'AR-2025-001',
        void_reason: 'Test void',
        current_user: viewer_user
      )

      expect(result).to be_failure
      expect(result.failure).to eq('No permission to void invoices')
    end
  end
end
```

#### 3. Fix Implemented by Coder

```ruby
# domains/accounting/accounts_receivable/use_cases/void_current_invoice.rb

def steps(serial:, void_reason:, current_user:)
  receivable = yield retrieve_receivable(serial: serial)
  yield authorize_user(receivable: receivable, user: current_user)  # Added
  yield validate_input(void_reason: void_reason)
  yield check_erp_settlement(serial: serial)
  # ...
end

private

def authorize_user(receivable:, user:)
  # Check project permission
  project = Project.find_by(serial: receivable.project_serial)
  return Failure('No permission to operate on this project') unless user.can_manage?(project)

  # Check role permission
  return Failure('No permission to void invoices') unless user.has_role?(:accountant, :admin)

  Success()
end
```

---

## Design Principles

### 1. Single Source of Truth

reviewFindings is the sole knowledge transfer medium — all Agents read from the Task JSON.

### 2. TDD Workflow

Fixes must have tests first (red), then implementation (green), ensuring fix quality.

### 3. Traceability

Each issue has a unique ID (e.g., SEC-001), enabling fix status tracking.

### 4. Incremental Fixes

Fix scope is controlled via fixScope, allowing you to choose to fix only Critical issues or fix everything.

---

## Related Files

- [continue.md](.claude/commands/continue.md) — Phase transition logic
- [tester.md](.claude/agents/tester.md) — Tester Agent definition
- [coder.md](.claude/agents/coder.md) — Coder Agent definition
- [operation-manual.md](docs/operation-manual.md) — Operation manual
