# ATDD Hub Operation Manual

This manual explains how to use the ATDD Hub's Command-Driven workflow for project task development.

---

## Table of Contents

1. [Quick Start](#quick-start)
2. [Task Types](#task-types)
3. [Epic Management](#epic-management)
4. [Workflow Details](#workflow-details)
5. [6 Agent Architecture](#6-agent-architecture)
6. [Commands Reference](#commands-reference)
7. [Metrics Tracking](#metrics-tracking)
8. [FAQ](#faq)

---

## Quick Start

### Starting a New Task

Use a Slash Command to start a task:

```
/feature sf_project, Project Approval Workflow
```

The system will automatically:
1. Create a task record (JSON + Kanban)
2. Enter the `requirement` phase
3. Call the `specist` Agent to begin requirements analysis
4. Ask you for requirement details

### Conversing with AI

During the `requirement` phase, you need to answer the specist's questions until the confidence level reaches 95%.

**Example conversation**:
```
specist: I understand you need a project approval workflow. A few things to confirm:
         1. What are the approval stages? (e.g., initial review, secondary review, final review)
         2. Who can perform approvals?
         3. What fields are needed for approval?

         Current confidence: 65%

you: Approval has two stages: "Project Manager Approval" and "Finance Approval".
     Project managers can approve their own projects, finance staff approves amounts.
     Need to record approval comments and approval time.

specist: Understood. A few more questions:
         1. Is there a required order for the two stages? (Must PM approve before Finance?)
         2. After rejection, can it be resubmitted for approval?

         Current confidence: 82%
```

### Continuing to the Next Phase

When confidence reaches the threshold, enter:

```
/continue
```

The system will automatically enter the next phase and call the corresponding Agent.

### Checking Progress

Enter at any time:

```
/status
```

Displays current task progress, completed phases, and Agents used.

---

## Task Types

| Type | Command | Use Case | Full Workflow |
|------|---------|----------|---------------|
| Feature | `/feature` | New feature development | requirement → specification → testing → development → review → gate |
| Fix | `/fix` | Bug fix | requirement → testing → development → review → gate |
| Refactor | `/refactor` | Code refactoring | requirement → specification → testing → development → review → gate |
| Test | `/test` | Add tests | requirement → testing → gate |

### Selection Guide

- **New feature**: Use `/feature`, includes a full specification design phase
- **Bug fix**: Use `/fix`, skips specification design, quickly enters testing and fixing
- **Refactoring**: Use `/refactor`, focuses on verifying behavior remains unchanged
- **Adding tests**: Use `/test`, minimal workflow, only writes tests without changing code
- **Large feature**: Use `/epic`, splits into multiple sub-tasks

---

## Epic Management

### What is an Epic?

An **Epic** is a large feature that needs to be split into multiple sub-tasks (Feature/Fix/Test) to complete.

### When to Use an Epic?

When a Feature meets the following conditions, consider elevating it to an Epic:

| Condition | Threshold |
|-----------|-----------|
| Number of Domains involved | ≥ 3 |
| Requires preliminary investigation/cleanup | Yes |
| Estimated acceptance scenarios | > 15 |
| Cannot deliver complete value in a single iteration | Yes |

### Epic Creation Flow

```
/epic sf_project, Invoice Allowance System
         │
         ▼
┌─────────────────────────────────┐
│ specist: Epic Planning (Proposal)│
│ • Identify involved Domains      │
│ • Split into Phases              │
│ • Identify sub-tasks             │
│ • Establish dependencies         │
└─────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────┐
│ Output proposal, await user     │
│ confirmation                     │
│ • Confirm → Create Epic + tasks │
│ • Adjust → Modify and reconfirm │
│ • Cancel → Abandon creation      │
└─────────────────────────────────┘
         │
         ▼ After confirmation
┌─────────────────────────────────┐
│ Create files:                    │
│ • Epic YAML                      │
│ • Sub-task JSON (T1.1, T1.2...) │
│ • Update Kanban                  │
└─────────────────────────────────┘
```

### Feature → Epic Elevation Flow

During `/feature` execution, if the specist determines the scope is too large:

```
/feature sf_project, Implement Allowance Feature
         │
         ▼
    Confidence Assessment (Clarification)
         │
         │ Confidence ≥ 95%
         ▼
    Scope Assessment ← This is where Epic elevation is decided
         │
    ┌────┴────┐
    │         │
 Maintain   Suggest Epic
 Feature    Elevation
    │       (Ask user)
    ▼         │
 Continue     ▼
            /epic ...
```

**Note**: Scope assessment is performed **after** confidence reaches 95%, to avoid making incorrect judgments without understanding the requirements.

### Executing Epic Sub-tasks

Epic sub-tasks are executed using standard commands, formatted as `{epic-id}:{task-id}`:

```bash
/feature core_web, erp-period-domain:T1-1    # Execute Epic sub-task T1-1
/fix core_web, erp-period-domain:T2-3        # Execute Epic sub-task T2-3
```

When executed, the system will automatically:
1. Read `epic.yml` to get the task definition
2. Check if dependencies are completed
3. If there are uncompleted dependencies, prompt the user and block startup
4. Create task JSON (with `epic` field for association)
5. Update the task's status to `in_progress` in the Epic

### Epic Sync Mechanism

When an Epic sub-task is completed (`/done` or `/close`), the system automatically syncs Epic status:

#### Sync Content

| File | Updated Content |
|------|----------------|
| `epic.yml` | task status → `completed`, metrics update |
| `tasks.md` | Progress overview, completed tasks list, next task indicator |

#### Identification Method

The `epic` field in the task JSON is used for association:

```json
{
  "epic": {
    "id": "erp-period-domain",
    "taskId": "T2-3",
    "phase": "Phase 2: Core UseCases"
  }
}
```

#### Progress Tracking

**Important**: The Epic's actual progress is recorded in `tasks.md`, not in `epic.yml`'s metrics.

The `/status` command reads progress from `tasks.md` to display:

```markdown
| Metric | Value |
|--------|-------|
| **Completed Tasks** | 14 / 32 |
| **Progress** | 44% |
| **Current Phase** | Phase 2 in progress |
| **Next Task** | T2-3: Implement ExportPeriod UseCase |
```

### Viewing Epic Progress

```bash
/status
```

Example output:

```
┌──────────────────────────────────────────────────────┐
│ 📊 sf_project Task Status                            │
├──────────────────────────────────────────────────────┤
│                                                      │
│ 🎯 Epic: Invoice Allowance System                    │
│    Progress: 13/17 (76%)  ████████░░ 4 tasks remain  │
│                                                      │
│    ✅ Phase 1: Investigation       5/5               │
│    ✅ Phase 2: Tech Debt Cleanup   3/3               │
│    🔄 Phase 4: Allowance System    1/5               │
│       ├─ ✅ T4.1 Create tables                       │
│       ├─ ⏳ T4.2 Implement AR Allowance (ready)      │
│       └─ 🔒 T4.4 Integrate EcPay (awaiting T4.2, T4.3) │
│                                                      │
└──────────────────────────────────────────────────────┘
```

### Epic Directory Structure

```
epics/
└── {project}/
    ├── active/           # In-progress Epics
    │   └── {epic-id}.yml
    └── completed/        # Completed Epics
        └── {epic-id}.yml

tasks/
└── {project}/
    ├── active/
    │   ├── T1.1.json     # Epic sub-tasks
    │   └── T4.2.json
    └── completed/
```

---

## Workflow Details

### Phase Descriptions

#### 1. Requirement (Requirements Clarification)

**Goal**: Ensure requirements are clear, reaching sufficient confidence

**Agent**: specist

**What you need to do**:
- Answer the specist's questions
- Provide business context and constraints
- Confirm whether assumptions are correct

**When it ends**: Confidence reaches the threshold (95% for all)

---

#### 2. Specification (Specification Design)

**Goal**: Create formal feature specifications

**Agent**: specist

**Output**:
- Acceptance criteria in Given-When-Then format
- Edge case definitions
- Error handling specifications

**What you need to do**:
- Review whether specifications are correct
- Confirm whether edge cases are complete

---

#### 3. Testing (Test Adjustment)

**Goal**: Generate test code

**Agent**: tester

**Output**:
- Test files (RSpec/Jest/Pytest)
- Test skeletons (initially pending)

**What you need to do**:
- Confirm tests cover all acceptance criteria
- (Optional) Adjust test details

---

#### 4. Development

**Goal**: Implement code to make tests pass

**Agent**: coder

**Output**:
- Business logic code
- Passing tests

**What you need to do**:
- Usually no intervention needed
- May need to assist if tests repeatedly fail

---

#### 5. Review (Result Review)

**Goal**: Ensure code quality

**Agents**:
- style-reviewer (code style)
- risk-reviewer (security/performance)

**Output**:
- Review report
- Grade (A/B/C/D)

**What you need to do**:
- Review the report
- Decide whether to accept (can request corrections)

---

#### 6. Gate (Quality Gate)

**Goal**: Final quality control

**Agent**: gatekeeper

**Check items**:
- All tests pass
- Review grade meets standards
- Specification alignment is complete
- No remaining TODOs

**What you need to do**:
- Confirm Go/No-Go decision
- Sign off on completion

---

## 6 Agent Architecture

### specist (Specification Expert)

**Responsibilities**:
- Requirements analysis and clarification
- Confidence assessment
- Writing Given-When-Then specifications
- Loading Domain knowledge

**Tools**: Read, Glob, Grep, SlashCommand, WebSearch

**Note**: Cannot write code, can only write specifications through spec-kit

---

### tester (Testing Expert)

**Responsibilities**:
- Generating tests from specifications
- Running tests and analyzing failures
- Suggesting fix directions

**Tools**: Read, Glob, Grep, Write, Edit, Bash

**Supported frameworks**: RSpec, Jest, Pytest

---

### coder (Development Expert)

**Responsibilities**:
- Implementing business logic
- Making tests pass
- Following DDD architecture

**Tools**: Read, Glob, Grep, Write, Edit, Bash

**Patterns**: Entity, Service, Repository, Use Case

---

### style-reviewer (Style Review)

**Responsibilities**:
- Checking naming conventions
- Checking language idioms
- Calculating complexity
- Grading (A/B/C/D)

**Tools**: Read, Glob, Grep (read-only)

**Reference**: Language guides under `style-guides/`

---

### risk-reviewer (Risk Review)

**Responsibilities**:
- Security vulnerability checks (OWASP Top 10)
- Performance issue detection
- Risk assessment (Critical/High/Medium/Low)

**Tools**: Read, Glob, Grep (read-only)

---

### gatekeeper (Quality Gate)

**Responsibilities**:
- Checking all quality thresholds
- Go/No-Go decisions
- Knowledge curation (recording new rules)

**Tools**: Read, Glob, Grep, Write

**Thresholds**:
- Test Gate: All tests pass
- Review Gate: Grade ≥ C
- Spec Gate: Complete specification alignment
- Doc Gate: No remaining TODOs

---

## Commands Reference

### Task Startup

| Command | Description | Example |
|---------|-------------|---------|
| `/feature {project}, {title}` | New feature | `/feature sf_project, Project Approval` |
| `/fix {project}, {title}` | Bug fix | `/fix core_web, Login Error` |
| `/refactor {project}, {title}` | Refactoring | `/refactor sf_project, Remove Old Reports` |
| `/test {project}, {title}` | Add tests | `/test sf_project, Approval Service Tests` |
| `/epic {project}, {title}` | Large feature (Epic) | `/epic sf_project, Invoice Allowance System` |

### Task Control

| Command | Description | When to Use |
|---------|-------------|-------------|
| `/continue` | Enter next phase | When current phase is complete |
| `/status` | Check progress (including Epic) | Any time |
| `/abort` | Abandon task | When you don't want to continue |

---

## Metrics Tracking

The system automatically tracks resource consumption for each Agent.

### Tracked Metrics

| Metric | Description | Example |
|--------|-------------|---------|
| **Tool Uses** | Number of tools used by Agent | 21 |
| **Tokens** | Token consumption | 41.9k |
| **Duration** | Execution time | 2m 12s |

### Viewing Methods

#### 1. `/status` Command

```
═══ Agents Used ═══

1. specist (Requirements Analysis)
   └─ 15 tools · 28.5k tokens · 1m 45s

2. tester (Test Generation)
   └─ 21 tools · 41.9k tokens · 2m 12s

═══ Resource Consumption ═══
🔧 Tools: 36
📊 Tokens: 70.4k
⏱️ Time: 3m 57s
```

#### 2. Kanban Card

Completed tasks show in the description:

```markdown
**Agents**: specist(15/28.5k), tester(21/41.9k), coder(18/35.2k)
**Total**: 54 tools / 105.6k tokens / 5m 42s
```

#### 3. JSON Task File

```json
{
  "agents": [
    {
      "name": "specist",
      "metrics": { "toolUses": 15, "tokens": 28500, "duration": "1m 45s" }
    }
  ],
  "metrics": {
    "totalToolUses": 54,
    "totalTokens": 105600,
    "totalDuration": "5m 42s"
  }
}
```

### Use Cases

- **Cost analysis**: Understand token consumption per task
- **Performance optimization**: Identify time-consuming phases
- **Benchmarking**: Resource requirements for different task types

---

## FAQ

### Q: Can I `/clear` to clean the conversation during phase transitions?

**A**: Yes, and it's recommended.

The ATDD workflow's inter-Agent context transfer **does not rely on conversation memory**, but instead uses:
- Task JSON (`tasks/{project}/active/*.json`)
- Specification files
- Test files
- E2E Fixtures

**Safe times to clear**:
```
✅ After /continue, before Agent starts
✅ After /fix-critical, /fix-high, /fix-all
✅ When any phase transition is complete
```

**Not recommended to clear**:
```
⚠️ During a phase (e.g., still clarifying requirements with specist)
⚠️ You've verbally provided important information not yet written to task JSON
```

The system will prompt during phase transitions: `💡 You can enter: /clear → /continue to clean conversation and continue`

**Recovery flow after `/clear`**:
```
/clear (clear conversation)
    ↓
/continue (system reads JSON)
    ↓
If multiple tasks → list choices
If only one → continue directly
```

---

### Q: Can I skip certain phases?

**A**: Not recommended. Each phase serves a purpose:
- Requirement ensures the direction is correct
- Testing ensures behavior is correct
- Review ensures quality meets standards

Skipping may lead to higher rework costs.

---

### Q: What if tests keep failing?

**A**: The system allows `testing` ↔ `development` loops:
1. tester analyzes failure causes
2. coder attempts to fix
3. Re-run tests

If the loop repeats too many times, you may need to return to requirement to re-clarify needs.

---

### Q: What if the review score is too low?

**A**: Two options:
1. **Fix**: Return to development to improve the code
2. **Accept**: If there's a valid reason (e.g., timeline pressure), you can accept and record the reason

---

### Q: Can I work on multiple tasks simultaneously?

**A**: Yes. The system supports multiple active tasks at the same time.

When there are multiple tasks, `/continue` will list all active tasks for you to choose:

```
┌──────────────────────────────────────────────────────┐
│ 📋 Multiple tasks in progress found                  │
├──────────────────────────────────────────────────────┤
│ 1. [sf_project] Project Approval Workflow (testing)  │
│ 2. [core_web] Login Page Fix (requirement)           │
└──────────────────────────────────────────────────────┘
```

You can also specify directly: `/continue sf_project` or `/continue {task_id}`.

> **Tip**: While multi-tasking is supported, focusing on 1-2 tasks at a time is more efficient.

---

### Q: How to view historical tasks?

**A**:
- Kanban: `tasks/{project}/kanban.md`
- Completed: `tasks/{project}/completed/`
- Failed: `tasks/{project}/failed/`

---

### Q: How is confidence calculated?

**A**: The specist evaluates based on the following factors:

| Factor | Description |
|--------|-------------|
| Business rule clarity | Whether core logic is clear |
| Boundary condition definition | Whether special cases are handled |
| Data structure | Whether input/output formats are clear |
| Error handling | Whether exceptions are defined |
| Integration points | Whether interactions with other systems are clear |

---

## Appendix: File Structure

```
atdd-hub/
├── .claude/
│   ├── agents/           # Agent definitions
│   │   ├── specist.md
│   │   ├── tester.md
│   │   ├── coder.md
│   │   ├── style-reviewer.md
│   │   ├── risk-reviewer.md
│   │   └── gatekeeper.md
│   └── commands/         # Slash Commands
│       ├── feature.md
│       ├── fix.md
│       ├── refactor.md
│       ├── test.md
│       ├── epic.md       # Epic management
│       ├── continue.md
│       ├── status.md
│       └── abort.md
├── epics/                # Epic records
│   ├── sf_project/
│   │   ├── active/
│   │   └── completed/
│   └── README.md
├── tasks/                # Task records
│   ├── sf_project/
│   └── core_web/
├── style-guides/         # Code style guides
│   ├── ruby.md
│   ├── python.md
│   └── javascript.md
├── domains/              # Domain knowledge
├── docs/                 # Documentation
│   └── operation-manual.md
└── CLAUDE.md             # AI instructions
```
