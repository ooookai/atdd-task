# Confidence Mechanism

ATDD Hub uses a confidence mechanism to ensure AI Agents don't blindly proceed at critical decision points. When information is insufficient, the system actively blocks and requests clarification, rather than guessing and producing incorrect results.

## Mechanism Overview

The system has **two confidence assessment systems**, each guarding different quality gates:

| Confidence | Responsible Agent | Protected Target | Threshold |
|------------|------------------|------------------|-----------|
| Requirement Confidence | specist | Quality of requirement → specification conversion | ≥ 95% to enter specification |
| Knowledge Confidence | curator | Correctness of Domain knowledge base writes | ≥ 95% to write (structure ≥ 70% to propose) |

### Common Design Principles

1. **Deduction system**: Starts at 100%, deducted per issue found
2. **Clear data sources**: Each system defines which files the Agent should read, and which to explicitly exclude
3. **Hard block**: All gates are hard blocks (`exit 1` or `exit 2`), not soft reminders
4. **YAML definitions**: Confidence dimensions and weights are defined in the `.claude/config/confidence/` directory

```
.claude/config/confidence/
├── requirement.yml   # Requirement confidence (used by specist)
└── knowledge.yml     # Knowledge confidence (used by curator)
```

---

## 1. Requirement Confidence

**Definition file**: `.claude/config/confidence/requirement.yml`

### Purpose

The Specist Agent evaluates during the requirement phase whether requirements are sufficiently clear, determining if it's safe to enter the specification phase (Given-When-Then specification writing).

### Thresholds

| Confidence | Status | Action |
|------------|--------|--------|
| < 70% | Severely insufficient | Must clarify, cannot continue |
| 70-94% | Partially clear | Clarification recommended, user can choose to skip |
| **≥ 95%** | **Sufficiently clear** | **Can enter specification phase** |

### Data Sources

The specist reads the following files when evaluating:

| File | Purpose |
|------|---------|
| `domains/{project}/domain-map.md` | Identify primary Domain and cross-domain relationships |
| `domains/{project}/ul.md` | Domain language mapping, confirm terminology consistency |
| `domains/{project}/business-rules.md` | Cross-domain business rule comparison |
| `domains/{project}/strategic/{Domain}.md` | Business purpose, capabilities, scope, state flows |
| `domains/{project}/tactical/{Domain}.md` | Known Pitfalls, Knowledge Gaps (conditional) |

**Explicitly excluded**: Codebase (`app/`, `db/`, `config/`). Code analysis is the coder's responsibility; the specist only evaluates requirements from a business perspective.

### Seven Dimensions

#### 1. Scope Boundary Clarity (20%)

Determines which Domain the requirement belongs to, whether it's cross-domain, and where boundaries are drawn.

**Why highest weight**: Unclear boundaries cause all subsequent analysis to be built on incorrect scope, affecting specification writing, test design, and implementation direction.

| Deduction ID | Score | Reason |
|-------------|-------|--------|
| SB-1 | -20 | Cannot determine which Domain the requirement belongs to |
| SB-2 | -15 | Cross-domain requirement but responsibility division not clarified |
| SB-3 | -10 | Domain assignment is clear but scope boundary is vague |
| SB-4 | -5 | Boundaries mostly clear but with individual gray areas |

#### 2. Logical Consistency (20%)

Whether the user's stated requirements are logically consistent with existing Domain knowledge, and whether new requirements contradict existing business rules.

**Why highest weight**: Contradicting existing rules means the direction is wrong — the earlier discovered, the better.

| Deduction ID | Score | Reason |
|-------------|-------|--------|
| LC-1 | -20 | Requirement directly contradicts existing business rules |
| LC-2 | -15 | Requirement is infeasible under existing state flows |
| LC-3 | -10 | Requirement is feasible but inconsistent with existing patterns, needs confirmation |
| LC-4 | -5 | Mostly consistent but individual rules need confirmation |

#### 3. Business Logic Clarity (20%)

Whether business rules are complete and clear. Whether all involved calculations, validations, and authorization logic are defined.

**Why highest weight**: Business logic is the core of requirements. Even if boundaries are clear and terminology is consistent, if key business rules remain vague, the resulting specification is unreliable.

| Deduction ID | Score | Reason |
|-------------|-------|--------|
| BL-1 | -20 | Core calculation logic or validation rules completely undefined |
| BL-2 | -15 | Business rules exist but are incomplete |
| BL-3 | -10 | Rules mostly clear but details need confirmation |
| BL-4 | -5 | Rules are clear but not yet recorded in the knowledge base |

#### 4. Edge Case Completeness (15%)

Whether boundary conditions and exception cases have been clarified. For example: zero amount, quantity exceeds limit, time conflicts, insufficient permissions.

**Why slightly lower weight**: Edge cases are the most commonly missed part, but compared to boundaries and logic, they can be supplemented in later phases.

| Deduction ID | Score | Reason |
|-------------|-------|--------|
| EC-1 | -15 | Edge cases completely undiscussed |
| EC-2 | -10 | Only happy path discussed, exception paths not confirmed |
| EC-3 | -8 | Known Pitfalls not incorporated into requirement considerations |
| EC-4 | -5 | Most edge cases clarified, but some individual cases unconfirmed |

#### 5. Impact Scope Identification (10%)

Whether downstream and upstream Domain impacts have been inventoried after modifying this requirement.

**Why lower weight**: The specist can determine this from the knowledge base, but complete impact requires coder confirmation.

| Deduction ID | Score | Reason |
|-------------|-------|--------|
| IS-1 | -10 | Cross-domain impacts completely unidentified |
| IS-2 | -8 | Know there are cross-domain impacts but degree not confirmed |
| IS-3 | -5 | Upstream/downstream inventoried but individual integration points need confirmation |

#### 6. Acceptability (10%)

Whether requirements can be transformed into concrete Given-When-Then acceptance scenarios.

**Why important**: This is the core check of ATDD. If requirements cannot be expressed as verifiable scenarios, the requirements themselves are not specific enough.

| Deduction ID | Score | Reason |
|-------------|-------|--------|
| TE-1 | -10 | Requirements too abstract, cannot write any acceptance scenarios |
| TE-2 | -8 | Can write partial scenarios but key acceptance criteria are unclear |
| TE-3 | -5 | Scenarios can be written but expected results are uncertain |

#### 7. Ubiquitous Language Consistency (5%)

Whether terminology used in the user's statements is consistent with `ul.md` definitions.

**Why lowest weight**: Terminology inconsistency is important but easily correctable — it can be calibrated in real-time during conversation.

| Deduction ID | Score | Reason |
|-------------|-------|--------|
| UL-1 | -5 | Used terminology not found in the knowledge base |
| UL-2 | -3 | Terminology exists but user's understanding differs from the definition |
| UL-3 | -2 | Synonyms mixed, already clarified in conversation |

### Calculation Example

```
Requirement: Add coupon feature

- Scope Boundary Clarity: 80% (SB-3: Unclear whether coupons belong to Billing or Promotion)
- Logical Consistency: 90% (LC-4: Need to confirm if coupons apply to existing settlement rules)
- Business Logic Clarity: 60% (BL-1: Coupon calculation logic undefined)
- Edge Case Completeness: 50% (EC-1: What if coupon amount > product amount?)
- Impact Scope Identification: 70% (IS-2: May affect invoicing and ERP settlement)
- Acceptability: 85% (TE-3: Expected discount amount calculation rules need confirmation)
- Ubiquitous Language Consistency: 90% (UL-3: "discount" vs "coupon" already clarified)

Weighted total:
80×0.20 + 90×0.20 + 60×0.20 + 50×0.15 + 70×0.10 + 85×0.10 + 90×0.05
= 16 + 18 + 12 + 7.5 + 7 + 8.5 + 4.5
= 73.5%

Result: < 95%, must clarify before entering specification.
```

### Gate Implementation

Handled by `.claude/hooks/validate-agent-call.sh` when the specist enters the specification phase. Returns `1` to hard block when confidence < 95%.

---

## 2. Knowledge Confidence

**Definition file**: `.claude/config/confidence/knowledge.yml`

### Purpose

The Curator Agent evaluates when inventorying and updating Domain knowledge whether the knowledge is sufficiently complete and correct, determining if it's safe to write to the knowledge base.

### Dual Thresholds

Knowledge confidence uses **dual thresholds**, corresponding to different stages of the Curator workflow:

| Threshold | Score | Corresponding Phase | Meaning |
|-----------|-------|-------------------|---------|
| Structure Threshold | ≥ 70% | Phase 2 → Phase 3 | Structure is clear enough to write an update proposal |
| **Content Threshold** | **≥ 95%** | **Phase 4 → Phase 5** | **Content sufficiently verified, can write to knowledge base** |

**Why dual thresholds**: Structure confidence only needs to know "what's missing" to start writing a proposal for user review. But before actually writing, each item's content correctness must be confirmed by the user, hence the content threshold is set at 95%.

### Data Sources

The curator reads the following files when evaluating:

| File | Purpose |
|------|---------|
| `domains/{project}/ul.md` | Terminology definition completeness, type classification, cross-references |
| `domains/{project}/business-rules.md` | Business rule coverage (VR/ST/CA/AU/CD) |
| `domains/{project}/domain-map.md` | Cross-domain relationships, Context Mapping, boundary definitions |
| `domains/{project}/strategic/{Domain}.md` | Business logic, capability definitions, scope, state flows |
| `domains/{project}/tactical/{Domain}.md` | System design decisions, known Pitfalls (conditional) |

**Explicitly excluded**: Codebase (`app/`, `db/`, `config/`). Code-knowledge consistency is the responsibility of the review/gate phases. The Curator obtains knowledge from business conversations, not from code inference. This follows DDD principles: the knowledge flow direction is "business reality → knowledge documents → code", and must not be reversed.

### Seven Dimensions

#### 1. Terminology Completeness (10%)

Evaluates the completeness of terminology definitions in `ul.md`. Whether all involved concepts have definitions, and whether definitions include type classification, bilingual mapping, examples, and cross-references.

**Why lower weight**: Ubiquitous Language is the foundation of DDD, but terminology is relatively easy to supplement.

| Deduction ID | Score | Reason |
|-------------|-------|--------|
| TE-1 | -15 | Core Entity/Aggregate completely missing definitions |
| TE-2 | -10 | Terminology has definitions but lacks type classification |
| TE-3 | -8 | Terminology lacks concrete examples or bilingual mapping |
| TE-4 | -5 | Related terms lack cross-references |

#### 2. Rule Coverage (25%)

Evaluates the coverage of rules in `business-rules.md`. Each Domain should have complete validation rules (VR), state transitions (ST), calculation rules (CA), authorization rules (AU), and cross-domain chain rules (CD).

**Why highest weight**: Business rules are the core value of knowledge — the knowledge that all downstream Agents (specist, tester, coder) depend on most.

| Deduction ID | Score | Reason |
|-------------|-------|--------|
| RC-1 | -25 | Core validation rules (VR) completely undefined |
| RC-2 | -20 | State transition rules (ST) missing |
| RC-3 | -15 | Calculation rules (CA) missing formulas or calculation logic |
| RC-4 | -10 | Rules exist but missing key conditions or exception handling |
| RC-5 | -5 | Rules mostly complete but individual conditions need confirmation |
| RC-6 | -8 | Cross-domain chain rules (CD) not identified (domain-map shows cross-domain relationships but business-rules has no corresponding CD rules) |

> **RC-6 Note**: When `domain-map.md` shows an upstream-downstream relationship between two Domains, `business-rules.md` should have a corresponding CD rule recording the trigger-reaction chain. For example, SolarProject status changes trigger Accounting to create financial records. Missing CD rules mean cross-domain impacts are undocumented, making downstream development prone to omissions.

#### 3. Domain Boundary Clarity (15%)

Evaluates the clarity of Domain boundaries. Includes responsibility scope (Includes/Excludes), relationships with other Domains, and integration point documentation.

**Why important**: Drawing wrong boundaries is the costliest error in DDD — late-stage correction costs are extremely high.

| Deduction ID | Score | Reason |
|-------------|-------|--------|
| BC-1 | -20 | Domain responsibilities completely unclear |
| BC-2 | -15 | Relationships with other Domains undefined |
| BC-3 | -10 | Integration points (Sync API / Async Events) undocumented |
| BC-4 | -5 | Boundaries mostly clear but with individual gray areas |

#### 4. Object Model Clarity (15%)

Evaluates whether the Domain's core concepts have been identified, whether relationships are clear, and whether business logic ownership is clear.

**Key design decision**: This dimension doesn't care about implementation patterns (Aggregate/UseCase/ViewObject is the coder's decision), only about "what business concepts exist, their relationships, and who owns the logic".

| Deduction ID | Score | Reason |
|-------------|-------|--------|
| OM-1 | -15 | Core concepts not identified |
| OM-2 | -10 | Relationships between concepts not recorded |
| OM-3 | -8 | Business logic ownership unclear |
| OM-4 | -5 | Model mostly clear but individual associations need confirmation |

#### 5. Cross-File Consistency (15%)

Checks consistency across knowledge files. The same concept should have consistent descriptions in `ul.md`, `business-rules.md`, and `strategic/*.md`.

**Why important**: Contradictory knowledge is more dangerous than missing knowledge. Missing means you don't know; contradictions cause different Agents to make opposite decisions.

| Deduction ID | Score | Reason |
|-------------|-------|--------|
| CO-1 | -15 | Direct contradiction found (same concept described oppositely in different files) |
| CO-2 | -10 | Terminology definitions inconsistent between ul.md and strategic/tactical files |
| CO-3 | -8 | State machine description doesn't match rule definitions |
| CO-4 | -5 | Minor inconsistency (different wording but possibly same meaning) |

#### 6. Knowledge Actionability (10%)

Evaluates whether knowledge can be directly used by downstream Agents. Whether the specist can write acceptance criteria from it, whether the coder can implement from it, whether the tester can design tests from it.

**Why lower weight**: This is usually a derivative problem of other dimension deficiencies — fixing other dimensions naturally improves this.

| Deduction ID | Score | Reason |
|-------------|-------|--------|
| AC-1 | -10 | Knowledge too abstract, cannot write any acceptance scenarios from it |
| AC-2 | -8 | Rules lack specific values or thresholds |
| AC-3 | -5 | Knowledge basically usable but with individual ambiguities |

#### 7. Document Structure Completeness (10%)

Evaluates whether strategic/tactical files have the required sections. Other dimensions check knowledge "content" quality; this dimension checks whether "structure" is complete.

**Why needed**: A Strategic file missing its business purpose means even with complete rules, you don't know why the Domain exists; unrecorded Knowledge Gaps means you don't know what you don't know.

| Deduction ID | Score | Reason |
|-------------|-------|--------|
| DS-1 | -10 | `strategic/{Domain}.md` doesn't exist |
| DS-2 | -8 | Business purpose not recorded |
| DS-3 | -5 | Business capabilities not listed |
| DS-4 | -5 | Knowledge Gaps not recorded |

### Calculation Example

```
Knowledge inventory: SolarProject Domain

- Terminology Completeness: 80% (TE-3: SolarProject lacks concrete examples)
- Rule Coverage: 52% (RC-2: Site state machine undefined, RC-6: CD rules with Accounting not identified)
- Domain Boundary Clarity: 70% (BC-3: Integration points with Accounting Domain undocumented)
- Object Model Clarity: 70% (OM-2: Relationship between SolarProject and Contract not recorded)
- Cross-File Consistency: 85% (CO-4: ul.md uses "site" but strategic uses "solar project")
- Knowledge Actionability: 90% (AC-3: Rent calculation decimal rounding rules unclear)
- Document Structure Completeness: 85% (DS-3: Business capabilities not listed)

Weighted total:
80×0.10 + 52×0.25 + 70×0.15 + 70×0.15 + 85×0.15 + 90×0.10 + 85×0.10
= 8 + 13 + 10.5 + 10.5 + 12.75 + 9 + 8.5
= 72.25%

Structure confidence 72.25% ≥ 70% → Can proceed to Phase 3 to write proposal
Content confidence 72.25% < 95% → Must verify each item in Phase 4 before writing
```

### Gate Implementation

Handled by `.claude/hooks/confidence-gate.sh` (Gate 1) when the Curator attempts to write to `domains/**/*.md`. If not confirmed by the user, `exit 2` hard blocks. Confirmation records are stored in `.claude/.knowledge-confirmed`, valid for 5 minutes.

---

## Gate Script Overview

### Requirement Confidence Gate

`.claude/hooks/validate-agent-call.sh` handles:

```
specist enters specification phase
         │
         ▼
  Check confidence >= 95%
         │
         ├── Yes → Allow
         └── No → Hard block, require clarification
```

### Knowledge Confidence Gate

`.claude/hooks/confidence-gate.sh` (Gate 1) handles:

```
PreToolUse (Write|Edit) triggered
         │
         ▼
  Read file_path
         │
         ├── domains/**/*.md → Check if user has confirmed (5-minute validity)
         │       │
         │       ├── Confirmed → Allow
         │       └── Not confirmed → exit 2 hard block
         │
         └── Other → Allow
```

> **Note**: `confidence-gate.sh` also includes Gate 2 (investigation prerequisite check), which is a behavioral gate rather than a confidence mechanism — see [Debug Tips Workflow](debug-tips-workflow.md) for details.
