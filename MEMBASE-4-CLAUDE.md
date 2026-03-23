# Membase for Claude — Persistent Knowledge Database Pattern

**A pattern for giving Claude Code persistent, version-controlled, self-auditing project memory using an append-only SQLite database.**

> This pattern was developed across 212 sessions on a commercial SaaS project (Agent Red Customer Experience). It evolved from markdown-only memory into a structured database with 9 managed artifact types after discovering that markdown backlogs drift, context windows forget, and session boundaries lose state. The approach below is extractable to any project.

---

## The Problem

Claude Code sessions have three persistent memory challenges:

1. **Context window saturation** — Long sessions accumulate stale context that biases decisions.
2. **Session boundary amnesia** — Each new session starts cold. CLAUDE.md and MEMORY.md help, but they are unstructured and drift over time.
3. **Undetected regression** — When code changes break previously-verified behavior, there is no automated signal unless you have tests that specifically check for it.

The markdown-only approach (CLAUDE.md + MEMORY.md + topic files) works well for the first ~20 sessions. After that, the files grow unwieldy, contradictions accumulate, and there is no machine-verifiable way to know if what Claude "remembers" is still true.

---

## Architecture Overview

```
project/
  CLAUDE.md                    # Rules & behavior (HOW to work)
  memory/MEMORY.md             # State & bootstrap (WHAT has been done)
  tools/knowledge-db/
    db.py                      # Append-only SQLite API (~1,900 lines)
    knowledge.db               # The database (auto-created)
    seed.py                    # Initial data loader
    assertions.py              # Machine-verifiable checks (~280 lines)
    app.py                     # Read-only web UI (~230 lines)
  .claude/
    hooks/
      assertion-check.py       # SessionStart hook: run assertions + inject handoff
      scheduler.py             # UserPromptSubmit hook: process scheduled prompts
    settings.local.json        # Hook registration
    SCHEDULE.md                # Pre-planned prompts (FIFO queue)
```

### Foundational Principles

1. **Append-only change control** — No rows are ever updated or deleted. Every change creates a new versioned record with `changed_by`, `changed_at`, and `change_reason`. Current state = latest version per ID (via SQL views).
2. **No phantom artifacts** — If Claude references something, it must exist. If it exists, it must be under change control. If it is under change control, its history must be retrievable.
3. **Machine-verifiable assertions** — Specifications can carry grep/glob assertions. "Claude remembers" becomes "Claude proves."
4. **Never-delete retention** — At ~20 KB/session, storage supports ~57,000 years at 1 session/day. Never purge data.

---

## Step 1: Artifact System Design

The database stores **9 managed artifact types** and **2 supporting record types**. Start with the core 3 (specifications, operational procedures, assertion runs) and add more as your project grows.

### Core Tables (Start Here)

| Table | Purpose |
|-------|---------|
| `specifications` | Requirements — testable descriptions of system behavior |
| `operational_procedures` | SOPs, deployment procedures, repeatable processes |
| `assertion_runs` | History of machine-verifiable assertion executions |
| `session_prompts` | Event-sourced session handoff prompts |

### Extended Tables (Add When Needed)

| Table | Purpose |
|-------|---------|
| `tests` | Individual test artifacts linked to specifications |
| `test_plans` + `test_plan_phases` | Orchestrating artifact: ordered test phases with gate criteria |
| `work_items` | Units of work: regression, defect, or new capability |
| `backlog_snapshots` | Point-in-time snapshots of active work items |
| `documents` | General-purpose project knowledge (migrated topic files, guides) |
| `environment_config` | Environment-specific values under change control |
| `test_coverage` | Many-to-many test-to-spec mapping |

### Key Design Decisions

#### Versioning Pattern (All Tables Follow This)

```sql
CREATE TABLE specifications (
    rowid INTEGER PRIMARY KEY AUTOINCREMENT,
    id TEXT NOT NULL,              -- e.g. "SPEC-0245" or "245.1" for sub-specs
    version INTEGER NOT NULL,
    title TEXT NOT NULL,
    description TEXT,
    priority TEXT,                 -- e.g. "P0", "P1", "P2"
    scope TEXT,                    -- grouping / module scope
    section TEXT,                  -- logical section within the project
    handle TEXT,                   -- short mnemonic (e.g. "max-turns-align")
    tags TEXT,                     -- JSON array of tags for filtering
    type TEXT NOT NULL DEFAULT 'requirement',  -- requirement | governance | protected_behavior
    status TEXT NOT NULL,          -- specified | implemented | verified | retired
    assertions TEXT,               -- JSON array of machine-verifiable checks
    changed_by TEXT NOT NULL,      -- 'claude', 'seed', or session ID
    changed_at TEXT NOT NULL,      -- ISO 8601 UTC
    change_reason TEXT NOT NULL,   -- Human-readable explanation
    UNIQUE(id, version)
);

-- Current state = latest version per ID (SQL view)
CREATE VIEW current_specifications AS
SELECT s.* FROM specifications s
INNER JOIN (
    SELECT id, MAX(version) AS max_v
    FROM specifications GROUP BY id
) m ON s.id = m.id AND s.version = m.max_v;
```

#### Specification Types

| Type | Purpose |
|------|---------|
| `requirement` | Business requirement — "would a different choice affect the customer or the business?" |
| `governance` | Process rules — GOV-01 through GOV-18 (how the human-AI team works) |
| `protected_behavior` | Machine-verifiable assertions that must always pass |

#### Tests Table (Spec-Linked)

```sql
CREATE TABLE tests (
    rowid INTEGER PRIMARY KEY AUTOINCREMENT,
    id TEXT NOT NULL,              -- e.g. "TEST-0001"
    version INTEGER NOT NULL,
    title TEXT NOT NULL,
    spec_id TEXT NOT NULL,         -- links to specifications.id
    test_type TEXT NOT NULL,       -- unit | integration | e2e | security | regression | performance
    test_file TEXT,                -- file path
    test_class TEXT,
    test_function TEXT,
    description TEXT,
    expected_outcome TEXT NOT NULL,
    last_result TEXT,              -- PASS | FAIL | null
    last_executed_at TEXT,
    changed_by TEXT NOT NULL,
    changed_at TEXT NOT NULL,
    change_reason TEXT NOT NULL,
    UNIQUE(id, version)
);
```

#### Test Plans (Orchestrating Artifact)

An orchestrating artifact contains ordering, criteria, and execution context. It references other artifacts by ID **without duplicating their content**. Each referenced artifact is independently managed and versioned.

```sql
CREATE TABLE test_plans (
    rowid INTEGER PRIMARY KEY AUTOINCREMENT,
    id TEXT NOT NULL,              -- e.g. "PLAN-001"
    version INTEGER NOT NULL,
    title TEXT NOT NULL,
    description TEXT,
    status TEXT NOT NULL,          -- active | completed | retired
    changed_by TEXT NOT NULL,
    changed_at TEXT NOT NULL,
    change_reason TEXT NOT NULL,
    UNIQUE(id, version)
);

CREATE TABLE test_plan_phases (
    rowid INTEGER PRIMARY KEY AUTOINCREMENT,
    id TEXT NOT NULL,              -- e.g. "PHASE-001"
    version INTEGER NOT NULL,
    plan_id TEXT NOT NULL,         -- links to test_plans.id
    phase_order INTEGER NOT NULL,
    title TEXT NOT NULL,
    description TEXT,
    gate_criteria TEXT NOT NULL,   -- what must pass to proceed
    test_ids TEXT,                 -- JSON array of test IDs in this phase
    last_result TEXT,
    last_executed_at TEXT,
    changed_by TEXT NOT NULL,
    changed_at TEXT NOT NULL,
    change_reason TEXT NOT NULL,
    UNIQUE(id, version)
);
```

#### Work Items (Origin + Component Taxonomy)

```sql
CREATE TABLE work_items (
    rowid INTEGER PRIMARY KEY AUTOINCREMENT,
    id TEXT NOT NULL,              -- e.g. "WI-001"
    version INTEGER NOT NULL,
    title TEXT NOT NULL,
    description TEXT,
    origin TEXT NOT NULL,          -- regression | defect | new | hygiene
    component TEXT NOT NULL,       -- e.g. infrastructure, database, customer_interface
    stage TEXT NOT NULL DEFAULT 'created', -- created | tested | backlogged | implementing | resolved
    source_spec_id TEXT,           -- what spec is this about?
    source_test_id TEXT,           -- what test revealed it?
    failure_description TEXT,
    resolution_status TEXT NOT NULL, -- open | in_progress | resolved | wont_fix
    priority TEXT,
    changed_by TEXT NOT NULL,
    changed_at TEXT NOT NULL,
    change_reason TEXT NOT NULL,
    UNIQUE(id, version)
);
```

#### Documents (General Knowledge)

```sql
CREATE TABLE documents (
    rowid INTEGER PRIMARY KEY AUTOINCREMENT,
    id TEXT NOT NULL,
    version INTEGER NOT NULL,
    title TEXT NOT NULL,
    category TEXT,                 -- e.g. "architecture", "operational", "reference"
    content TEXT NOT NULL,
    source_file TEXT,              -- original file path if migrated from markdown
    changed_by TEXT NOT NULL,
    changed_at TEXT NOT NULL,
    change_reason TEXT NOT NULL,
    UNIQUE(id, version)
);
```

### Indexes

Add indexes on frequently-queried columns for performance as the database grows:

```sql
CREATE INDEX IF NOT EXISTS idx_specs_id ON specifications(id);
CREATE INDEX IF NOT EXISTS idx_specs_status ON specifications(status);
CREATE INDEX IF NOT EXISTS idx_specs_version ON specifications(id, version);
CREATE INDEX IF NOT EXISTS idx_specs_changed_at ON specifications(changed_at);
CREATE INDEX IF NOT EXISTS idx_assertion_runs_spec ON assertion_runs(spec_id);
CREATE INDEX IF NOT EXISTS idx_tests_id ON tests(id);
CREATE INDEX IF NOT EXISTS idx_tests_spec ON tests(spec_id);
CREATE INDEX IF NOT EXISTS idx_test_plans_id ON test_plans(id);
CREATE INDEX IF NOT EXISTS idx_work_items_id ON work_items(id);
CREATE INDEX IF NOT EXISTS idx_work_items_status ON work_items(resolution_status);
CREATE INDEX IF NOT EXISTS idx_session_prompts_session ON session_prompts(session_id);
CREATE INDEX IF NOT EXISTS idx_documents_id ON documents(id);
```

### Python API Pattern

```python
class KnowledgeDB:
    """Append-only knowledge database. Claude is the sole writer."""

    AUDIT_INTERVAL = 5  # every Nth session is an audit

    _UNSET = object()  # Sentinel: distinguishes "not provided" from "set to None"

    def __init__(self, db_path=None):
        self.db_path = db_path or Path(__file__).parent / "knowledge.db"
        self._conn = None
        self._ensure_schema()

    def _get_conn(self):
        if self._conn is None:
            self._conn = sqlite3.connect(str(self.db_path))
            self._conn.row_factory = sqlite3.Row
            self._conn.execute("PRAGMA journal_mode=WAL")  # Safe for concurrent reads
        return self._conn

    def get_spec(self, spec_id):
        """Get latest version of a spec."""
        row = self._get_conn().execute(
            "SELECT * FROM current_specifications WHERE id = ?", (spec_id,)
        ).fetchone()
        return dict(row) if row else None

    def insert_spec(self, id, title, status, changed_by, change_reason,
                    type="requirement", **kwargs):
        """Insert a new spec (version 1)."""
        ...

    def update_spec(self, spec_id, changed_by, change_reason, **kwargs):
        """Create new version with changes. Append-only -- never modifies existing rows."""
        current = self.get_spec(spec_id)
        if not current:
            raise ValueError(f"Spec {spec_id} not found")
        next_version = current["version"] + 1
        # Merge: use new value if provided (via _UNSET sentinel), else keep current
        ...

    def insert_test(self, id, title, spec_id, test_type, expected_outcome,
                    changed_by, change_reason, **kwargs):
        """Insert a new test artifact linked to a specification."""
        ...

    def insert_test_plan(self, id, title, status, changed_by, change_reason, **kwargs):
        """Insert a new test plan."""
        ...

    def insert_work_item(self, id, title, origin, component, resolution_status,
                         changed_by, change_reason, **kwargs):
        """Insert a new work item."""
        ...

    def insert_document(self, id, title, content, changed_by, change_reason, **kwargs):
        """Insert a project knowledge document."""
        ...

    def get_summary(self):
        """Counts by status for all artifact types."""
        ...

    def get_open_work_items(self):
        """All work items with resolution_status = open."""
        ...

    def get_untested_specs(self):
        """Specs with no linked test artifacts."""
        ...

    def get_active_test_plan(self):
        """The currently active test plan and its phases."""
        ...

    def create_backlog_snapshot_from_current(self, changed_by, change_reason):
        """Snapshot all open work items into a backlog record."""
        ...

    def export_json(self, output_path=None):
        """Full logical backup as JSON. Safe to run anytime."""
        ...

    def close(self):
        if self._conn:
            self._conn.close()
            self._conn = None
```

### Usage in Claude Sessions

```python
import sys; sys.path.insert(0, "tools/knowledge-db")
from db import KnowledgeDB

db = KnowledgeDB()
db.get_spec("SPEC-0245")                  # Latest version of spec
db.get_summary()                           # Counts by status across all types
db.get_open_work_items()                   # Active work
db.get_untested_specs()                    # Coverage gaps
db.update_spec("SPEC-0245", changed_by="claude",
               change_reason="Verified in S42",
               status="verified")
db.insert_work_item("WI-001", title="Fix rate limiter",
                    origin="defect", component="infrastructure",
                    resolution_status="open",
                    changed_by="claude", change_reason="Found during S42 testing")
db.export_json()                           # Full logical backup
db.close()
```

---

## Step 2: Machine-Verifiable Assertions

This is the key innovation. Each specification can have JSON assertions that Claude can run automatically:

```python
# Three assertion types:
assertions = [
    # grep -- pattern must be found in file (with min_count)
    {"type": "grep", "file": "src/config.py", "pattern": "MAX_TURNS",
     "min_count": 1, "description": "Max turns constant defined"},

    # grep_absent -- pattern must NOT be found in file
    {"type": "grep_absent", "file": "src/wizard.py", "pattern": "DuplicateStepName",
     "description": "Wizard does not duplicate sidebar content"},

    # glob -- file path pattern must match at least one file
    {"type": "glob", "pattern": "admin/shared/HelpTooltip.tsx",
     "description": "HelpTooltip component exists"},
]
```

### Assertion Runner

The assertion runner (`assertions.py`) iterates all specs with assertions, runs each check, and records results in `assertion_runs`. It returns a structured summary:

```python
def run_all_assertions(db, triggered_by="manual"):
    """Run all assertions across all specs. Returns summary dict."""
    specs = db.list_specs_with_assertions()
    results = []
    for spec in specs:
        parsed = json.loads(spec["assertions"])
        checks = [run_single_assertion(a) for a in parsed]
        overall_passed = all(c["passed"] for c in checks)
        db.record_assertion_run(spec["id"], spec["version"],
                                overall_passed, checks, triggered_by)
        results.append({"spec_id": spec["id"], "title": spec["title"],
                        "overall_passed": overall_passed, "checks": checks})
    return {"passed": sum(1 for r in results if r["overall_passed"]),
            "failed": sum(1 for r in results if not r["overall_passed"]),
            "details": results}
```

### Why This Matters

Without assertions, Claude "believes" specs are implemented based on session memory. With assertions, Claude **proves** specs are implemented by checking the actual codebase. When a future code change accidentally breaks a behavior, the assertion catches it at session start.

---

## Step 3: Governance Principles

These governance principles evolved over 189 sessions. They are not mandatory — adopt the ones that fit your project.

### GOV-01: Specs Are the Negotiation Artifact

When the human requests a change or identifies a flaw, Claude's first priority is creating/updating a specification. Testing and implementation proceed only after mutual understanding is established.

### GOV-02: Specs Are Immutable Without Owner Consent

Claude proposes; the human decides. Specifications are the shared contract between human and AI.

### GOV-03: Spec Granularity Is Driven by Test Unambiguity

Every spec should produce an unambiguous PASS/FAIL when tested. If a spec cannot be tested unambiguously, it needs to be decomposed.

### GOV-04: Specs Mature Through Use

Iterative refinement is normal maturation, not a defect. Specs start as `specified`, get `implemented`, then `verified`. Each transition creates a new version.

### GOV-05: Fix the Spec First, Not the Code

When a test fails, first verify the specification is correct. Then verify the test is correct. Only then fix the implementation. Correct the requirement before changing code.

### GOV-06: Specify on Contact

When Claude touches unspecified code, it becomes controlled. Any existing behavior that is modified should first be recorded as a specification.

### GOV-07: No Bug Fixes During Testing

Record defects as work items during test phases; fix them in separate sessions. This keeps testing phases clean and prevents scope creep.

### GOV-08: Knowledge Database Is the Single Source of Truth

All project knowledge lives in the KB. Markdown files store rules (CLAUDE.md) and operational state (MEMORY.md), but specifications, tests, procedures, and documents belong in the database.

### GOV-09: Owner Input Classification

When the owner describes what the system **must do**, **should do**, **must include**, or states numbered criteria, classify the input as **specification language**. Before writing any code: (1) record or verify specifications, (2) identify work items for gaps, (3) add work items to the backlog, (4) present the backlog for prioritization. A `UserPromptSubmit` hook mechanically enforces this, but Claude must also self-enforce.

### GOV-10: Tests Must Exercise Exposed Production Interfaces

Source inspection tests (reading TypeScript files, checking for string literals) are useful as regression supplements but are not Test artifacts. Each Test must produce PASS/FAIL against observable outcomes on live/staging systems. Tests are written and linked to specifications **before** implementation.

### GOV-11: Design Decision Checkpoint Discipline

At each work item or phase completion boundary, Claude must review implementation decisions for specification coverage before proceeding. This is a batched checkpoint, not a real-time pause — review decisions at natural boundaries rather than after every line of code.

### GOV-12: Work Item Creation Triggers Test Creation

Creating a work item initiates test creation; the backlog initiates implementation. This ensures no work proceeds without a testable definition of "done." Tests may be logical assertions, user story descriptions, or abstract descriptions.

### GOV-13: Phase Assignment at Creation

Every Test artifact must be assigned to at least one test plan phase upon creation. No orphan tests — if a test exists, it belongs to a phase. This prevents accumulating tests that are never executed.

### GOV-14: UI Test Sync

When a UI element changes (label, layout, component), the corresponding E2E tests must be updated in the same work item. UI changes without matching test updates create silent drift.

### GOV-15: Test Fix Approval Gate

No fixing failed tests without explicit owner approval. Failed tests may indicate either a test bug or a product bug — the owner decides which. Claude must report diagnostics and the proposed fix, then wait for approval. Enforced via `owner_approved=True` parameter in the KB `update_work_item()` method.

### GOV-16: Deployment Approval Gate

No deployment to any environment without explicit owner approval. Claude prepares the build and presents the deployment plan; the owner approves. This prevents accidental production deployments.

### GOV-17: Quality First

Prioritize quality (correctness, completeness, absence of defects) over effort or speed. Software engineering excellence is the primary objective.

### GOV-18: Assertion Quality Standard

Machine-verifiable assertions must target specific, meaningful code identifiers — not generic patterns that match broadly. An assertion that matches "MUST" in any file is noise; an assertion that matches "ActivationService" in `activation_service.py` is signal. Batch assertion generation scripts must use semantic categorization (what does this spec describe?) rather than keyword heuristics (does the title contain this word?).

### The Specification Litmus Test

"Would a different choice affect the customer or the business?" If yes, it is a specification. If no, it is an implementation detail.

**IS a specification:** Intent taxonomy, tier pricing, conversation handling rules, privacy commitments, UI field inventory, integration choices, quality criteria.

**NOT a specification:** Database schema, middleware ordering, startup sequence, API response shapes, env vars, internal module structure.

---

## Step 4: Session Hooks

### SessionStart Hook — Assertion Check + Handoff Injection

Register in `.claude/settings.local.json`:

```json
{
  "hooks": {
    "SessionStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "python3 .claude/hooks/assertion-check.py",
            "timeout": 15
          }
        ]
      }
    ]
  }
}
```

The hook does five things:
1. **Runs all assertions** and classifies failures as either "regressions" (implemented/verified specs now failing) or "expected" (specified but not yet implemented).
2. **Checks untested work items** (GOV-12 drift detection) — flags open work items missing linked tests.
3. **Displays the quality dashboard** — 4 metrics: assertion coverage (target ≥60%), test traceability (target >80%), defect velocity (net positive = good), defect escape rate (target: 0 production incidents).
4. **Prunes old assertion runs** — keeps only the latest 5 runs per spec to prevent unbounded table growth (reduced 229K → 9K rows, 93 → 18 MB database size).
5. **Reads the session handoff prompt** stored by the previous session, so Claude automatically knows what to work on next.

```python
# Classification logic in the hook:
for failure in failures:
    spec = db.get_spec(failure["spec_id"])
    status = spec["status"]
    if status in ("implemented", "verified"):
        # REGRESSION -- something broke
        regressions.append(failure)
    else:
        # Expected -- not yet implemented
        expected.append(failure)
```

### UserPromptSubmit Hook — Session Scheduler

The scheduler hook (`.claude/hooks/scheduler.py`) processes pre-planned prompts from `.claude/SCHEDULE.md`. This enables deferred automation — Claude can schedule future tasks during a session that execute in subsequent sessions.

```markdown
<!-- .claude/SCHEDULE.md format -->
## Group: post-deploy-checks
trigger: always

Update staging deployment status in MEMORY.md after confirming revision is active.
```

**Trigger types:**
- `always` — inject with the next user prompt
- `session_end` — inject when wrap-up keywords are detected
- `after:N` — inject after N user prompts have been processed

The scheduler uses file locking to prevent race conditions when multiple hooks fire concurrently.

### Session Handoff Prompts

At the end of each session, Claude stores a structured prompt for the next session:

```python
db.insert_session_prompt("S42",
    prompt_text="Continue work on Feature X. Group 3 complete, "
        "start Group 4. 4,539 tests, 0 failures.",
    context={
        "production_version": "1.58.3",
        "test_count": 4539,
        "next_tasks": ["WI-243", "WI-244"]
    })
```

The next session's hook automatically retrieves and displays this, then marks it as consumed.

---

## Step 5: Audit Cadence

Every Nth session (default: 5) is automatically flagged as an **audit session**. During wrap-up, the handoff generator checks and prepends audit instructions:

```python
def is_audit_session(self, next_session_id, interval=None):
    n = self.parse_session_number(next_session_id)  # "S100" -> 100
    if n is None:
        return False
    return n % (interval or self.AUDIT_INTERVAL) == 0

def get_audit_directive(self):
    return (
        "AUDIT SESSION: Perform a fresh-context review before new work:\n"
        "1. Knowledge DB integrity -- run assertions, check for status drift\n"
        "2. MEMORY.md and CLAUDE.md -- accuracy, staleness, contradictions\n"
        "3. Repeatable Procedures -- still accurate?\n"
        "4. Open design debt -- TODOs, type safety, large files\n"
        "5. Hooks and scheduler -- verify all hooks execute without errors\n"
        "Report findings before proceeding with regular work."
    )
```

This compensates for the inevitable context drift that accumulates across sessions.

---

## Step 6: CLAUDE.md + MEMORY.md Boundary

The database does not replace your markdown files — it complements them:

| File | Role | Content | Updates |
|------|------|---------|---------|
| **CLAUDE.md** | Rules & behavior | How to work: procedures, mandates, evaluation criteria | Rarely — only when rules change |
| **MEMORY.md** | State & bootstrap | What has been done: versions, sessions, quick reference | Every session |
| **Knowledge DB** | Artifacts & assertions | Specifications, tests, work items, documents, procedures | Every code change |

**Rule of thumb:** If it tells Claude *what to do*, it goes in CLAUDE.md. If it tells Claude *what has been done* or *how to access something*, it goes in MEMORY.md. Formal artifacts with machine-verifiable truth go in the database.

**Anti-drift rules for CLAUDE.md:**
- All project knowledge lives in the KB — do not create new markdown files for project knowledge
- Permitted markdown: CLAUDE.md (rules), MEMORY.md + topic files (operational state), external-facing docs
- Topic files are Claude's operational memory, not the canonical source of truth

### CLAUDE.md Should Reference the DB

```markdown
### Knowledge Database
The Knowledge Database (`tools/knowledge-db/knowledge.db`) is the canonical
source of truth for all project artifacts. Claude is the sole writer. The owner
observes through the read-only web UI at localhost:8090.

Python API: `tools/knowledge-db/db.py`
  db.get_spec("SPEC-0245")
  db.insert_work_item(...)
  db.get_open_work_items()
  db.get_untested_specs()
  db.get_summary()
```

---

## Step 7: Read-Only Web UI

A simple Flask app provides the owner with visibility without giving them write access:

```python
# app.py -- ~230 lines, runs at localhost:8090
from flask import Flask, render_template_string
from db import KnowledgeDB

app = Flask(__name__)

@app.route("/")
def index():
    db = KnowledgeDB()
    summary = db.get_summary()
    specs = db.list_specs()
    db.close()
    return render_template_string(TEMPLATE, summary=summary, specs=specs)
```

The owner sees all artifacts, their statuses, assertion results, and version history. They tell Claude what to fix; Claude creates a corrected version.

---

## Step 8: Seed Script

Bootstrap the database from your existing project artifacts:

```python
# seed.py -- Run once to populate initial data
# IMPORTANT: Guard against accidental re-seeding
def main():
    db = KnowledgeDB()
    existing = db.get_summary()
    if existing["spec_total"] > 0 and "--force" not in sys.argv:
        print(f"Database already has {existing['spec_total']} specs. "
              "Use --force to re-seed.")
        sys.exit(1)

    # Load from whatever source you have (markdown backlog, JIRA export, etc.)
    for item in your_backlog:
        db.insert_spec(
            id=item["id"],
            title=item["title"],
            status="specified",
            changed_by="seed",
            change_reason="Initial import from backlog"
        )
    db.close()
```

---

## Step 9: Claude Code Skills (Executable Governance)

Skills convert prose procedures into executable playbooks that Claude follows step-by-step. Each skill is a `SKILL.md` file with YAML frontmatter and markdown instructions, stored in `.claude/skills/<name>/SKILL.md`. Skills follow the [Agent Skills](https://agentskills.io) open standard.

### Why Skills Matter for Membase

Governance rules (GOV-01 through GOV-18) tell Claude *what to do*, but multi-step chains are easy to partially execute. GOV-12 ("work item creation triggers test creation") + GOV-13 ("every test assigned to a plan phase") requires three consecutive database writes — miss one and the system silently drifts. Skills make the chain atomic by encoding all steps in a single invocable playbook.

### Two Invocation Modes

```yaml
# Owner-only: destructive operations, deployment, session wrap-up
---
name: deploy
description: Deploy to staging or production
disable-model-invocation: true
allowed-tools: Bash, Read, Grep
---

# Auto-invocable: knowledge work Claude should do proactively
---
name: kb-query
description: Query the Knowledge Database
allowed-tools: Bash, Read
---
```

- **`disable-model-invocation: true`** — only the human can trigger. Use for side-effect-heavy operations (deploy, seed, session wrap-up).
- **Default (omitted)** — both human and Claude can invoke. Claude loads the skill when the conversation context matches the description.

### Recommended Skills for Membase Projects

| Skill | Mode | What It Mechanizes |
|-------|------|-------------------|
| `/deploy` | Owner only | Build-deploy-verify pipeline with GOV-16 deploy gate |
| `/seed-tenant` | Owner only | Database seeding / tenant provisioning |
| `/kb-session-wrap` | Owner only | End-of-session procedure (KB update → MEMORY → git push → handoff) |
| `/run-tests` | Auto | Test execution (batch runner + E2E pipeline) |
| `/kb-query` | Auto | Read-only KB lookups (specs, tests, WIs, docs) |
| `/kb-spec` | Auto | Guided spec creation with duplicate detection (GOV-01) |
| `/kb-work-item` | Auto | WI → Test → Phase assignment chain (GOV-12 + GOV-13) |
| `/kb-promote` | Auto | Assertion-gated spec status promotion |

### Skill Template

```yaml
---
name: kb-work-item
description: Create a work item with automatic test creation and backlog assignment.
allowed-tools: Bash, Read, Grep
---

# Create Work Item (GOV-12 Chain)

## Step 1: Determine next IDs
Query KB for next available WI and TEST IDs.

## Step 2: Create work item
```python
db.insert_work_item(id=..., title=..., origin=..., component=..., ...)
```

## Step 3: Create linked test (MANDATORY — GOV-12)
```python
db.insert_test(id=..., spec_id=..., test_type=..., expected_outcome=..., ...)
```

## Step 4: Assign to PLAN-001 phase (MANDATORY — GOV-13)
Record the phase assignment.

## Step 5: Report summary
Print the full chain: WI → TEST → Phase.
```

### Design Principle

**CLAUDE.md states the rule, skills encode the execution, KB procedures are the audit trail.** This three-layer separation means rules don't contain implementation details, skills don't repeat governance rationale, and KB procedures remain the canonical historical record. When a procedure changes, update the skill (executable) and the KB procedure (record) — CLAUDE.md only changes if the *rule itself* changes.

---

## Recurring Instructions for CLAUDE.md

Add these to your CLAUDE.md so Claude maintains the database correctly across sessions:

```markdown
### Knowledge Database Maintenance

**Claude is the sole writer.** The owner observes through the read-only web UI.

**When to update the database:**

| Trigger | Action |
|---------|--------|
| Implement a feature | `update_spec(id, status="implemented", change_reason="...")` |
| Verify via tests | `update_spec(id, status="verified", change_reason="...")` |
| Discover a wrong status | `update_spec(id, status=corrected, change_reason="...")` |
| Find a defect | `insert_work_item(origin="defect", ...)` |
| Test reveals regression | `insert_work_item(origin="regression", ...)` |
| Complete a procedure | Create new version via `insert_op_procedure()` |

**Assertion rules:**
- Valid types: `grep`, `grep_absent`, `glob`
- Pass assertions as raw Python lists -- the API handles JSON serialization
- After adding new fields to schemas, update count assertions (test drift)
- Run assertions after every batch of changes

**Session wrap-up:** Store a handoff prompt via `db.insert_session_prompt()`
so the next session starts with context.

**Audit cadence:** Every 5th session performs a fresh-context integrity review
before new work.
```

---

## Step 9: Skills Framework

Claude Code skills (`.claude/skills/`) define reusable workflows as slash commands. For Membase projects, skills should be **KB-aware** — they read from and write to the Knowledge Database rather than operating generically.

### Skill Anatomy

Each skill is a `SKILL.md` file with YAML frontmatter and markdown instructions:

```markdown
---
name: deploy
description: Build, deploy, and verify a release with GOV-16 gate enforcement
allowed-tools: Bash, Read, Edit, Write, WebFetch
---

# Deploy Skill

## Prerequisites
- Owner must have approved deployment (GOV-16)
- All regression assertions must pass

## Steps
1. Run assertion check against Knowledge Database
2. Build container image with current PRODUCT_VERSION
3. Push to container registry
4. Update container app revision
5. Run upgrade verification script
6. Record deployment event in KB
```

### Project-Specific vs Generic Skills

Generic development toolkits (e.g., gstack) provide slash commands like `/ship`, `/review`, `/qa` that work on any codebase. These are useful for greenfield projects but **conflict with governed projects** because they bypass governance rules.

For Membase projects, create **project-specific skills** that enforce your governance:

| Generic Skill | KB-Aware Equivalent | Why It's Better |
|--------------|-------------------|----------------|
| `/ship` (auto-PR) | `/deploy` (GOV-16 gated) | Requires owner approval before deployment |
| `/review` (code review) | `/kb-promote` (assertion-gated) | Validates assertions before status promotion |
| `/qa` (browser testing) | `/run-tests` (pipeline-aware) | Runs through test plan phases, records results to KB |

### Template-Driven Skill Generation

For projects with many skills (>5), consider a template pattern to prevent drift:

```
skills/
  deploy/SKILL.md.tmpl      # Source of truth (template)
  deploy/SKILL.md            # Generated output (committed)
```

A generation script processes templates into final skills, and CI checks that generated files match their templates. This prevents manual edits from silently drifting away from the canonical definition.

### Skill Catalog in MEMORY.md

List all available skills in MEMORY.md so each session knows what is available:

```markdown
## Claude Code Skills
- `/deploy` — Build-deploy-verify pipeline (owner approval required)
- `/run-tests` — Execute test pipeline with thermal management
- `/kb-query` — Read-only KB lookups
- `/kb-spec` — Guided specification creation (GOV-01)
- `/kb-work-item` — WI → Test → Phase chain (GOV-12+13)
- `/kb-promote` — Assertion-gated status promotion
```

Skills are local to the developer machine (`.claude/` is typically gitignored). Document them in MEMORY.md so they survive across sessions even if the directory listing is not in context.

---

## Step 10: Test Pipeline Architecture

Membase Step 2 describes assertions and Step 3 defines governance around testing. This step covers the **execution infrastructure** — how tests actually run and how results flow back into the KB.

### Pipeline-as-Code Pattern

Create a single entry point that orchestrates all test phases:

```python
# scripts/test_pipeline.py
# Single-invocation test runner with phase ordering

PHASES = [
    Phase("unit", "tests/unit/", timeout=120),
    Phase("multi_tenant", "tests/multi_tenant/", timeout=180),
    Phase("integration", "tests/integration/", timeout=300),
    Phase("chat", "tests/chat/", timeout=120),
    Phase("e2e_live", "tests/e2e_live/", timeout=600, requires_staging=True),
    Phase("regression", "tests/regression/", timeout=60),
    Phase("fuzzing", "tests/fuzzing/", timeout=300),
    Phase("property", "tests/property/", timeout=180),
]

def run_pipeline(phases=None, env="staging"):
    results = {}
    for phase in PHASES:
        if phases and phase.name not in phases:
            continue
        results[phase.name] = phase.execute(env=env)
    return PipelineReport(results)
```

### Phase Taxonomy

Organize test phases from fastest/cheapest to slowest/most expensive:

| Phase | Speed | Cost | What It Catches |
|-------|-------|------|----------------|
| **Unit** | ~1 min | Free | Logic errors, type mismatches |
| **Integration** | ~3 min | Free | Component interaction bugs |
| **E2E (live)** | ~10 min | ~$0.50 (staging infra) | Real deployment issues, UI regressions |
| **Regression** | ~1 min | Free | Previously-fixed bugs returning |
| **Fuzzing** (Schemathesis) | ~5 min | Free | API contract violations, edge cases |
| **Property-based** (Hypothesis) | ~3 min | Free | Algebraic invariant violations |
| **Mutation** (mutmut) | ~15 min | Free | Weak assertions, undertested code paths |

Cost-annotating your tiers helps teams make informed decisions about test frequency and CI integration — run cheap phases on every commit, expensive phases on PR or nightly.

### Test Result Flow to KB

Pipeline results should flow back into the Knowledge Database:

```python
# After pipeline execution
for phase_name, result in report.items():
    for test in result.tests:
        kdb.insert_test(
            id=test.kb_id,
            spec_id=test.spec_id,
            last_result="pass" if test.passed else "fail",
            last_executed_at=datetime.now().isoformat(),
            ...
        )
```

### Regression Gate Pattern

Run assertions and regression tests **before every build**:

```python
# Pre-build check (wired into deploy skill or CI)
def pre_build_gate():
    assertions = run_assertions()
    if assertions.failures > 0:
        raise BuildGateError(f"{assertions.failures} assertion failures")
    regressions = run_phase("regression")
    if regressions.failures > 0:
        raise BuildGateError(f"{regressions.failures} regression failures")
```

This ensures no build ships with known regressions. The assertion runner (Step 2) is the first gate; the regression test phase is the second.

---

## Step 11: Deployment Automation

For projects that deploy to cloud infrastructure, automate the build-deploy-verify pipeline and enforce governance gates.

### Build-Deploy-Verify Pattern

Structure deployment as three distinct phases with a governance gate between "build" and "deploy":

```
Build → [Artifacts in registry] → GOV-16 Gate → Deploy → Verify
```

```python
# Deployment pipeline (simplified)
def deploy(env: str, version: str, owner_approved: bool = False):
    # Phase 1: Build
    image_tag = build_container(version)
    push_to_registry(image_tag)

    # GOV-16 Gate
    if not owner_approved:
        raise DeployGateError("Owner approval required (GOV-16)")

    # Phase 2: Deploy
    update_container_app(env, image_tag)
    wait_for_healthy(env, timeout=120)

    # Phase 3: Verify
    results = run_upgrade_verification(env)
    record_deployment_event(env, version, results)
    return results
```

### Environment Promotion

Promote through environments with increasing verification:

```
Local tests → Staging deploy → Staging E2E → Owner approval → Production deploy → Production verification
```

Each environment gate prevents defects from propagating:

| Gate | What It Checks | Blocks On |
|------|---------------|-----------|
| Pre-staging | Unit + integration tests | Any failure |
| Post-staging | E2E pipeline against staging | >N failures or regression failures |
| Pre-production | Owner approval (GOV-16) | Missing approval |
| Post-production | Upgrade verification script | Health check or endpoint failures |

### Seed Scripts for Data Bootstrapping

Multi-tenant systems need deterministic data seeding:

```python
# scripts/seed_tenant.py — phased, idempotent
PHASES = [
    Phase(1, "core_config", seed_core_config),
    Phase(2, "auth_keys", seed_auth_keys),
    Phase(3, "knowledge_base", seed_knowledge_documents),
    Phase(4, "entitlements", seed_entitlement_docs),
    # ...
]

def seed(tenant_id: str, env: str, phases: list[int] | None = None):
    for phase in PHASES:
        if phases and phase.number not in phases:
            continue
        phase.execute(tenant_id, env)
```

Phase-based seeding lets you re-run individual phases without re-seeding everything, and makes it safe to add new phases as the system grows.

### Rollback Strategy

Document rollback as a first-class deployment artifact:

- **Container apps:** Redeploy the previous image tag (keep last 3 tags in registry)
- **Database migrations:** Design forward-compatible schemas (additive columns, not destructive changes)
- **Configuration:** Version config documents in the KB; rollback = deploy previous version

---

## Step 12: Parallel Execution & Multi-Agent Coordination

As projects grow beyond ~100 sessions, single-agent sessions become a bottleneck. Multiple AI agents can work in parallel — but only with structured coordination to prevent conflicts.

### Background Agent Delegation

Claude Code supports launching background agents for independent tasks while the primary session continues implementation:

```
Primary session: Implementing SPEC-1234 (writes code)
Background agent 1: Investigating test failures (read-only research)
Background agent 2: Exploring codebase for pattern X (read-only research)
```

**Rule:** Background agents should be **read-only** (research, exploration, analysis). Only the primary session writes code and modifies the KB. This prevents merge conflicts and ensures governance gates (GOV-01, GOV-16) are enforced by a single actor.

### Loyal Opposition Model

A second AI agent (different model or role) provides independent review of the primary agent's work:

```
Prime Builder (Claude Code) ←→ Message Bridge ←→ Loyal Opposition (Codex/GPT)
```

The Loyal Opposition:
- Inspects implementation, plans, and documentation
- Files evidence-based reports with severity ratings (P0–P3)
- Does NOT write code or modify the KB
- Communicates findings via structured message bridge (not shared files)

This pattern catches blind spots, architectural drift, and security issues that a single-agent workflow misses. The message bridge ensures findings are structured and actionable rather than lost in conversation context.

### Session Scheduler

Pre-plan prompts that inject automatically at session start:

```markdown
<!-- .claude/SCHEDULE.md -->
## Scheduled Prompts (FIFO)

### Prompt 1
Run the E2E test pipeline and report results.

### Prompt 2
Perform 5th-session audit: assertion check, MEMORY.md accuracy, procedure verification.
```

A `UserPromptSubmit` hook processes the first pending prompt, removes it from the queue, and injects it into the session. This enables unattended batch workflows — start a session, and the scheduled work executes automatically.

### Declarative Parallel Work Config

For complex sprints requiring multiple simultaneous work streams, define a manifest:

```json
{
  "sprint": "BACKLOG-018",
  "streams": [
    {"name": "Phase 3 - Integration Framework", "agent": "primary", "priority": 1},
    {"name": "Test infrastructure fixes", "agent": "background", "priority": 2},
    {"name": "Security review", "agent": "loyal-opposition", "priority": 3}
  ]
}
```

This formalizes what most teams do ad-hoc — assigning parallel work streams to different agents with clear priority ordering and ownership boundaries.

---



## Step 13: Defense-in-Depth Hook Architecture

Steps 4 and 12 describe individual hooks. This step describes the **layered defense model** — three enforcement layers that catch different failure modes at different times, creating redundancy so that no single layer's failure silently degrades quality.

### The Three Layers

```
Layer 1: Real-time (PreToolUse)      — catches mistakes AS Claude writes
Layer 2: Commit-time (git pre-commit) — catches mistakes BEFORE they enter history
Layer 3: Session-time (SessionStart)  — catches drift BETWEEN sessions
```

Each layer addresses a distinct failure mode:

| Layer | When It Fires | What It Catches | Failure Mode |
|-------|--------------|----------------|-------------|
| PreToolUse | Every Write/Edit/Bash call | Credential leaks, destructive commands | Claude writes insecure code or runs dangerous commands |
| Pre-commit | Every `git commit` | Assertion count regression, test deletion, architecture erosion, untraced frontend changes, hardcoded secrets | Claude commits code that silently degrades quality |
| SessionStart | Once per session | KB assertion failures, untested specs, GOV-12 drift, quality metric decline | Drift accumulated across sessions |

### Layer 1: Real-Time PreToolUse Hooks

PreToolUse hooks intercept tool calls **before execution**. They are the fastest feedback loop — Claude sees the rejection immediately and adjusts.

**Credential Scanner** (`credential-scan.py`) — intercepts Write and Edit tool calls:
- Blocks hardcoded Azure FQDNs, Redis/Cosmos/KeyVault connection strings, API keys with known prefixes
- Enforces SPEC-0058 (no hardcoded environment attributes)
- Excludes memory files, hook scripts, deployment configs, and `.env` files
- Fail-open on parse errors (only blocks on positive credential match)

**Destructive Command Gate** (`destructive-gate.py`) — intercepts Bash tool calls:
- Blocks recursive deletion, force-push, hard reset, DROP TABLE, DELETE without WHERE
- Blocks `--no-verify` on git commands (prevents hook bypass)
- Blocks `curl`/`wget` with credential parameters (exfiltration prevention)
- Allows safe deletions: `__pycache__`, `node_modules`, `.pytest_cache`, `dist/`, `build/`, temp files
- Fail-closed on exceptions (unknown commands are blocked, not allowed)

**Design principle:** Layer 1 hooks should be **fast** (< 100ms) and **conservative**. A false positive blocks one tool call; a false negative lets a mistake into the codebase. Prefer false positives — Claude will rephrase and retry.

### Layer 2: Commit-Time Git Pre-Commit Hooks

Pre-commit hooks run 5 checks sequentially. If any check fails, the commit is rejected.

**Assertion Count Ratchet** (`check_assertion_ratchet.py`):
- Maintains a baseline JSON file mapping every test file to its assertion count
- On commit: if any staged test file has FEWER assertions than baseline, reject
- If assertion count INCREASED, auto-update baseline and stage the updated JSON
- Prevents "test simplification" that silently reduces coverage
- Counted patterns: `assert`, `self.assert*`, `pytest.raises`, `pytest.warns`, `pytest.approx`

**Test File Deletion Guard** (`check_test_deletion.py`):
- If any `test_*.py` file is deleted in the staged changes, reject
- Tests can be added or modified but never removed without owner approval

**Architectural Grep Guards** (`check_arch_guards.py`):
- 6 guards that verify critical architectural patterns exist when related files are staged
- Example: if `src/agents/` files are staged, verify `SLIMTransport` or `NatsTransport` imports exist
- Example: if `src/multi_tenant/` files are staged, verify `EntitlementService` class exists
- Smart triggering: only checks guards whose trigger paths overlap with staged files

**TSX Commit Gate** (`check_tsx_gate.py`):
- If any `.tsx` file is staged, the commit message MUST contain `SPEC-NNNN`
- Every frontend change must trace to a specification (GOV-01 enforcement)

**Hardcoded Environment Scanner** (`check_hardcoded_env.py`):
- Same detection logic as the PreToolUse credential scanner, but operates on staged file content
- Redundant with Layer 1, deliberately — catches credentials that entered via external tools or manual editing

### Layer 3: Session-Time Hooks

Described in Step 4. The SessionStart hook catches drift that accumulates between sessions — assertions that used to pass but now fail (regressions), work items missing tests (GOV-12 violations), and quality metric decline.

### Layered Defense in Practice

The three layers are deliberately redundant. Consider a scenario where Claude removes an assertion from a test file:

1. **Layer 1 would not catch this** — Write/Edit hooks check for credentials, not assertion counts
2. **Layer 2 catches it** — the assertion ratchet detects the count decrease and rejects the commit
3. **Layer 3 would catch it next session** — if Layer 2 were bypassed, the SessionStart assertion runner would detect the spec is no longer verified

This redundancy means no single failure mode goes undetected. The cost is maintenance overhead (5 hook scripts + 5 pre-commit checks + 1 session hook), but the payoff is that quality regressions require bypassing multiple independent enforcement mechanisms.

### Hook Registration

PreToolUse hooks are registered in `.claude/settings.json` (or `settings.local.json`):

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [{"type": "command", "command": "python3 .claude/hooks/credential-scan.py"}]
      },
      {
        "matcher": "Bash",
        "hooks": [{"type": "command", "command": "python3 .claude/hooks/destructive-gate.py"}]
      }
    ],
    "UserPromptSubmit": [
      {
        "hooks": [{"type": "command", "command": "python3 .claude/hooks/spec-classifier.py"}]
      },
      {
        "hooks": [{"type": "command", "command": "python3 .claude/hooks/scheduler.py"}]
      }
    ],
    "SessionStart": [
      {
        "hooks": [{"type": "command", "command": "python3 .claude/hooks/assertion-check.py", "timeout": 15}]
      }
    ]
  }
}
```

Pre-commit hooks are a standard bash script in `.git/hooks/pre-commit` that runs the 5 Python checks sequentially.

### Custom Agents as Quality Reviewers

Beyond hooks, Claude Code supports **custom agents** (`.claude/agents/`) that function as specialized reviewers:

| Agent | Model | Purpose |
|-------|-------|---------|
| `code-reviewer` | Sonnet | Confidence-filtered code review (only reports >80% confidence findings). Checks correctness, security (OWASP), governance compliance, multi-tenant isolation. |
| `security-analyzer` | Sonnet | OWASP Top 10 scanner plus project-specific checks: tenant isolation (partition keys, Redis prefixes), widget XSS, webhook HMAC verification, credential handling. |

Agents are invoked on-demand (not automatically), but can be wired into skills or session procedures. They run on a lighter model (Sonnet) to save cost while the primary session uses the most capable model (Opus).

---

## Step 14: Automated Testing Strategy

Step 10 describes the pipeline architecture. This step describes the **testing philosophy** — what kinds of tests exist, how they are generated, what gaps remain, and the tradeoffs involved.

### Testing Taxonomy (12 Suite Types)

The project uses 12 distinct test suite types, organized from fastest/cheapest to slowest/most expensive:

| Suite | Est. Tests | Speed | What It Catches | Generation Method |
|-------|-----------|-------|----------------|-------------------|
| **Unit** | 950 | ~2 min | Logic errors, type mismatches, pure function bugs | Claude-generated from specs |
| **Core** (multi-tenant) | 3,700 | ~5 min | Tenant isolation, API routing, auth, middleware | Claude-generated from specs |
| **Integration** | 270 | ~3 min | Component interaction, database queries, cache behavior | Claude-generated from specs + WIs |
| **Agents** | 300 | ~2 min | MCP agent dispatch, tool execution, guardrails | Claude-generated from SPEC-1706..1712 |
| **Security** | 150 | ~2 min | Auth bypass, injection, tenant leaks, OWASP patterns | Claude-generated from security specs |
| **Regression** | 47 | ~1 min | Previously-fixed bugs | Auto-created when WI is resolved |
| **Widget** | 60 | ~1 min | Embed behavior, postMessage, launcher, resize | Claude-generated from widget specs |
| **Ops** | 80 | ~1 min | Deployment checks, config validation, health probes | Claude-generated from procedures |
| **E2E Live** | 1,100 | ~15 min | Real deployment: Playwright against staging/production | Claude-generated from user stories |
| **Load** | variable | ~10 min | Throughput, latency under load, rate limit behavior | Locust scenarios from capacity specs |
| **Fuzzing** | 307 ops | ~10 min | API contract violations, edge cases, unexpected inputs | Schemathesis auto-generated from OpenAPI |
| **Property-based** | 46 | ~3 min | Algebraic invariants, roundtrip properties | Hypothesis strategies from data models |

### Test Generation Patterns

Tests are generated through four distinct mechanisms:

**1. Specification-Driven (Primary):** When a spec is created via `/kb-spec`, Claude generates tests that verify the spec's requirements. The GOV-12 chain (WI -> Test -> Phase) ensures every work item has a linked test before implementation begins. This is the dominant generation mode (~85% of tests).

**2. Assertion-Driven:** Machine-verifiable assertions attached to specs (grep/glob patterns) auto-generate verification at session start. These are not pytest tests — they are codebase-state checks that run before any work begins.

**3. Auto-Generated (Fuzzing + Property):** Schemathesis generates test cases from the OpenAPI schema (307 operations across all endpoints). Hypothesis generates property-based tests from data model invariants (e.g., "serialization roundtrips preserve all fields"). These catch edge cases that specification-driven tests miss.

**4. Regression-Driven:** When a bug is found and fixed, a regression test is created as part of the work item resolution. These tests specifically reproduce the original failure condition to prevent recurrence.

### Live-Only Testing Principle (GOV-10)

All tests must exercise **production interfaces** (HTTP endpoints, rendered UI, real databases), never source code inspection. This principle emerged from a concrete failure: mock-based tests passed while the production system failed due to a database migration mismatch.

Implications:
- Unit tests call API handler functions with real request objects, not by importing internal helpers
- E2E tests use Playwright against a running server, not by reading TSX files
- Integration tests hit real Redis/Cosmos instances (or test containers), never mocked stores

The tradeoff: live tests are slower and environment-dependent. The project mitigates this with a **two-container test host pattern** — a dedicated test runner container (`agent-red-test-host`) with internal-only ingress runs tests against the main API container, keeping test infrastructure isolated from production traffic.

### Known Testing Gaps

Honest accounting of what is NOT covered:

| Gap | Why It Exists | Mitigation |
|-----|--------------|------------|
| **Visual regression** | No screenshot comparison tool integrated | Manual UI verification during deploy |
| **Cross-browser** | E2E tests run Chromium only | Widget tested on Chrome; other browsers untested |
| **Offline/degraded network** | Test host has reliable connectivity | Resilience specs cover Cosmos/Redis failures but not network partitions |
| **Real payment processing** | Stripe/PayPal adapters are tested with mocks | Integration framework specs cover adapter protocols; live payment testing deferred |
| **Mobile viewport** | Playwright tests use desktop viewport | Widget is responsive but not systematically tested on mobile dimensions |
| **Concurrent multi-tenant load** | Load tests target single tenant | Isolation specs verify partition-key separation but not concurrent-access behavior under load |

### Thermal-Safe Local Testing

On Windows 11 with thermal throttling concerns, the local test runner (`run-tests-thermal-safe.ps1`) distributes load across batches with configurable cooling pauses between them. This prevents thermal BSODs that occurred when running 9,000+ tests without throttling.

```
Batch 1: core-a (2,400 tests, parallel)  -> 30s cooldown
Batch 2: core-b (680 tests, parallel)    -> 30s cooldown
Batch 3: agents-chat (600 tests, parallel) -> 30s cooldown
Batch 4: integrations (400 tests, parallel) -> 30s cooldown
Batch 5: sequential (120 tests, serial)
```

### Test Result Flow

Test results flow into the KB through two paths:
1. **Cloud path:** The test host writes progressive results to Cosmos DB; the SPA admin console displays them with drill-down and performance charts
2. **Local path:** The thermal-safe runner produces JUnit XML; pipeline scripts parse results and update KB test records

---

## Step 15: Encoding Procedures to Reduce Orchestration Load

This is arguably the most important architectural insight from 212 sessions: **Claude is too error-prone for reliable multi-step orchestration when operating from prose instructions alone.** The solution is to encode repeatable procedures into executable artifacts (skills, hooks, scripts) that Claude follows mechanically rather than reasoning about dynamically.

### The Orchestration Problem

Claude Code operates as an agentic system — it reads instructions, reasons about them, and takes actions. This works well for novel problems but fails predictably for repeatable procedures:

1. **Step omission:** A 7-step procedure executed from memory will skip steps ~15% of the time. The skipped step is often the "boring" one (update the database, push to git, update MEMORY.md) rather than the "interesting" one (write the code).

2. **Order violation:** Steps that must execute in a specific order get reordered when Claude optimizes for perceived efficiency. Example: deploying before running the pre-deploy assertion check.

3. **Drift over sessions:** The same procedure executed in session S50 and S150 will differ because Claude's interpretation of prose instructions shifts with accumulated context. The procedure itself has not changed — Claude's execution of it has.

4. **Partial execution on error:** When step 3 of 7 fails, Claude often skips to a later step rather than stopping and reporting the failure. This creates partially-applied state that is harder to debug than a clean failure.

### The Encoding Principle

**Prompts encode intent. Hooks encode enforcement. Skills encode execution.**

| Mechanism | What It Encodes | When It Runs | Failure Mode |
|-----------|----------------|-------------|-------------|
| CLAUDE.md | Rules and principles | Read at session start | Claude "forgets" mid-session |
| Hooks | Enforcement gates | Automatically on events | Silent unless a violation occurs |
| Skills | Step-by-step procedures | On invocation (human or auto) | Claude follows steps mechanically |
| Scripts | Deterministic logic | Called by skills/hooks | Standard software failure (traceable) |

The key insight: **move logic DOWN this stack whenever possible.** A rule in CLAUDE.md is the weakest form of enforcement (relies on Claude reading and remembering). A hook is stronger (fires automatically). A script is strongest (executes deterministically with no AI reasoning involved).

### Practical Examples

**Before encoding (unreliable):**
CLAUDE.md says: "When creating a work item, also create a linked test and assign it to a PLAN-001 phase."

Claude follows this correctly ~80% of the time. The other 20%, it creates the WI but forgets the test or the phase assignment. GOV-12 violations accumulate silently.

**After encoding (reliable):**
The `/kb-work-item` skill contains explicit steps:
```
Step 1: Query KB for next WI and TEST IDs
Step 2: Create work item -> db.insert_work_item(...)
Step 3: Create linked test -> db.insert_test(...)  [MANDATORY — GOV-12]
Step 4: Assign to PLAN-001 phase [MANDATORY — GOV-13]
Step 5: Print summary of full chain: WI -> TEST -> Phase
```

Claude follows these steps mechanically. The skill is the executable form of the governance rule. If Claude still skips a step, the SessionStart hook catches the GOV-12 violation next session.

**Before encoding (unreliable):**
CLAUDE.md says: "Never commit code with hardcoded credentials."

Claude follows this correctly ~95% of the time. The other 5%, it writes a test file with a staging FQDN in a string literal, reasoning "this is just a test, not a real credential."

**After encoding (reliable):**
Three enforcement layers:
1. PreToolUse hook rejects the Write/Edit call immediately
2. Pre-commit hook scans staged content and rejects the commit
3. Next session's assertion check flags the SPEC-0058 violation

### The Skill Catalog as Operational Surface

The project's 10 skills define the complete operational surface — every repeatable procedure has a skill:

| Skill | What It Mechanizes | Steps | Orchestration Replaced |
|-------|-------------------|-------|----------------------|
| `/deploy` | Build, deploy, verify pipeline | 4 phases, ~20 checks | "Build the image, push it, deploy, check health..." |
| `/seed-tenant` | 9-phase database provisioning | 9 phases + post-seed | "Create the config, generate keys, seed articles..." |
| `/kb-session-wrap` | End-of-session procedure | 5 phases | "Update KB, update memory, commit, push, generate handoff..." |
| `/run-tests` | Test execution with suite selection | 6 modes | "Run the multi-tenant tests, then agents, then..." |
| `/kb-spec` | Specification creation with validation | 5 checks + creation | "Check for duplicates, validate fields, insert..." |
| `/kb-work-item` | GOV-12 chain (WI, Test, Phase, Backlog) | 5 steps | "Create work item, then test, then assign phase..." |
| `/kb-promote` | Assertion-gated status promotion | 4 checks + promotion | "Run assertions, check WIs, verify code exists..." |
| `/kb-query` | Read-only KB access | Query + format | "Query the specs table where status = ..." |
| `/kb-adr` | Architecture Decision Record creation | 4 sections | "Document the decision, rationale, consequences..." |
| `/prime-bridge-sync` | Multi-agent handoff coordination | Claim, execute, resolve | "Check inbox, claim task, do work, send result..." |

### Measuring Encoding Effectiveness

The effectiveness of procedure encoding can be measured by tracking GOV-12 violations over time:

- **Sessions S1-S50 (prose only):** GOV-12 violations detected in ~30% of sessions
- **Sessions S50-S100 (early skills):** GOV-12 violations detected in ~15% of sessions
- **Sessions S100-S200 (full encoding + hooks):** GOV-12 violations detected in ~3% of sessions

The remaining 3% represents edge cases where Claude operates outside any skill (novel tasks, debugging, exploration). These are acceptable — the goal is not zero violations, but zero *silent* violations. The SessionStart hook ensures every violation is surfaced.

### When NOT to Encode

Not everything should be a skill or hook:

- **Novel problem-solving** — debugging, architecture decisions, creative implementation. These require reasoning, not procedure-following.
- **One-time tasks** — migration scripts, data fixes, spike investigations. If it will only run once, the overhead of encoding exceeds the reliability benefit.
- **Rapidly-evolving workflows** — procedures that change every few sessions. Encode only after the workflow stabilizes (typically after ~5 manual executions).

The rule of thumb: **if Claude has executed the same procedure more than 3 times across different sessions, encode it.** The first execution is discovery, the second is validation, the third is the signal to automate.


---

## Lessons Learned (212 Sessions)

1. **The assertion runner is the single most valuable piece.** It turns "Claude remembers" into "Claude proves." Regressions caught at session start save hours of debugging.

2. **Append-only is not a limitation, it is a feature.** We never need to worry about lost data. At ~20 KB/session, storage is a non-issue for any realistic project lifetime.

3. **The double-serialization trap is real.** `update_spec(assertions=json.dumps([...]))` will double-encode because the method already calls `json.dumps()`. Always pass raw Python objects.

4. **Test drift is the #1 recurring failure mode.** When you add fields, enums, or schema entries, count-based assertions break. The fix is mechanical but easy to forget. The assertion runner catches it immediately.

5. **Session handoff prompts eliminate cold-start friction.** Instead of the human crafting "Continue work on X..." prompts, the previous session stores exactly what the next one needs.

6. **The 5-session audit cadence catches accumulated drift.** Individual sessions maintain the DB well, but small errors compound. Periodic fresh-context reviews are essential.

7. **MEMORY.md and CLAUDE.md still matter.** The database stores formal artifacts and assertions. The markdown files store rules, preferences, and operational patterns that do not fit a relational model.

8. **The read-only web UI builds trust.** The owner sees everything Claude knows without needing to read code or run scripts. This transparency is crucial for long-running commercial projects.

9. **Use a sentinel for "not provided" vs "set to None."** A bare `None` check cannot distinguish "caller omitted this argument" from "caller explicitly set it to None." A module-level `_UNSET = object()` sentinel resolves the ambiguity in merge logic.

10. **Index early.** As the database grows past ~100 specs, queries on `status`, `id`, and `changed_at` benefit noticeably from indexes. Add them in the schema creation, not as an afterthought.

11. **Python method name shadowing is silent.** When a class defines two methods with the same name, the second silently replaces the first. No error raised. Always use unique helper names per table (e.g., `_next_test_proc_version` vs `_next_test_version`).

12. **Governance principles emerge from use.** Do not try to design all the rules upfront. Let them crystallize through real sessions. Our 8 governance principles were all discovered through actual failures, not anticipated.

13. **The orchestrating artifact principle prevents content duplication.** Test plans reference test IDs, backlogs reference work item IDs. Never duplicate artifact content inside another artifact — reference by ID and keep each artifact independently versioned.

14. **Specification types enable different behaviors.** Requirements, governance rules, and protected behaviors have different change frequencies and verification patterns. A `type` column lets you filter and handle them differently.

15. **Migrate topic files to the database.** Markdown topic files inevitably drift from reality. Migrating project knowledge into documents under change control catches contradictions and enables search across all knowledge.

16. **Detect specification language mechanically.** When the owner says "must include" or "should do," that is a specification, not an implementation instruction. A `UserPromptSubmit` hook can detect these patterns and inject a reminder to follow the spec-first workflow before Claude starts writing code.

17. **Work item stages prevent premature implementation.** Adding a `stage` column (created → tested → backlogged → implementing → resolved) with transition enforcement prevents work items from reaching implementation without first having tests and backlog placement. The database enforces the workflow, not human vigilance.

18. **Route interception in E2E tests has blast radius.** Browser-level request interception (e.g., Playwright `page.route()`) can inadvertently capture requests beyond the intended scope. Glob patterns like `**/api/team/*` match more URLs than expected, including safe GET endpoints. Design tests to avoid overlapping route handlers rather than relying on method-based filtering.

19. **Governance principles compound.** Each governance principle builds on the others. GOV-12 (work item → test) only works because GOV-10 (tests must exercise production interfaces) defines what a valid test is, which only works because GOV-03 (test unambiguity) defines the quality bar. Adopt them incrementally, but expect later principles to reference earlier ones.

20. **The glossary is the pattern, not the code.** The most important thing Membase establishes is shared vocabulary. When the human says "backlog" and Claude says "backlog," both must refer to the same real, versioned, queryable artifact. Without this agreement, every other benefit (assertions, versioning, governance) is undermined.

21. **Live-only testing catches what mocks miss.** Source inspection tests (reading TypeScript files, checking string literals) and mocked API tests pass locally but miss integration failures. Converting to live external interface tests (HTTP calls to staging, Playwright against real UI) exposed real bugs that mock-based tests hid. The tradeoff: live tests are slower and environment-dependent, but their signal is trustworthy.

22. **Phase restoration preserves taxonomy.** When restructuring a test plan, restore phase numbers with new implementations rather than creating new phases or scattering tests. This preserves the plan's taxonomy, maps to existing KB records, and produces clear pipeline output. Remove phases only when the entire category is genuinely obsolete.

23. **Owner approval gates prevent autonomous damage.** GOV-15 (test fix approval) and GOV-16 (deployment approval) emerged from incidents where Claude "fixed" a failing test by relaxing expectations (hiding a real bug) or deployed without confirmation. The `owner_approved=True` parameter on `update_work_item()` makes the approval gate machine-enforced, not honor-system.

24. **Backlog snapshots are workflow gates.** The KB enforces `created → tested → backlogged → implementing → resolved` transitions. Advancing to `implementing` requires the work item to exist in a backlog snapshot. This prevents bypassing prioritization — you cannot implement work that was never formally planned.

25. **Governance principles compound and accelerate.** GOV-01 through GOV-18 were discovered over 189 sessions. Early principles (GOV-01–06) took many sessions to crystallize. Later ones (GOV-13–18) emerged within a few sessions because the pattern was established. Expect governance discovery to accelerate as the system matures.

26. **Batch assertion generation requires semantic categorization.** A 3-tier strategy (grep in known file → glob for file existence → keyword-to-file mapping) achieved broad assertion coverage across 2,016 specs. But keyword heuristics (matching "MUST" in titles) create systematic mis-mappings. Categorizing specs by their subject (what does this spec describe?) is the only reliable approach. GOV-18 codifies this.

27. **Quality dashboards make metrics actionable.** Displaying 4 key metrics (assertion coverage, test traceability, defect velocity, defect escape rate) at every session start changed behavior immediately. When a metric is red, it becomes the first priority. Without visibility, quality degradation is silent and gradual.

28. **Testable element inventory closes coverage gaps.** Inventorying every UI component, form field, and interactive element (520 across 12 admin pages) revealed that "we have tests" is different from "everything is tested." The 14-dimension taxonomy (A–N: visibility, content, interaction, etc.) ensures tests cover what users actually experience, not just what developers thought to test.

29. **Assertion pruning is essential for long-running projects.** Without pruning, the assertion_runs table grew to 229K rows and 93 MB (larger than all other data combined). Keeping only the latest 5 runs per spec reduced this dramatically with no loss of diagnostic value. Automate this in the SessionStart hook.

30. **Monolith splits require assertion remapping.** When a large source file is split into a package of submodules, every spec assertion referencing the old file path silently fails. The session-start hook reports these as individual failures without identifying the root cause pattern. After splitting a 5,085-line monolith into 5 domain submodules (S169), 50 specs needed file path remapping (S173). Always audit assertion file paths after structural refactors.

31. **Mock E2E tests complement live tests, not replace them.** A zero-backend mock development environment (527 tests across 14 files) enables rapid UI development without API dependencies. But mock tests verify UI behavior against fixture data — they cannot catch integration failures, auth issues, or data format mismatches. Both layers are necessary: mocks for speed, live tests for truth.

32. **Batch description enrichment using source file context.** Specs linked to source files (via assertions) can generate meaningful descriptions automatically: restate the title as a requirement, add context from the assertion's source file path. This enriched 885 NULL-description specs in one pass, achieving 92% description coverage.

33. **Skills are prompt templates, not code.** The most valuable Claude Code skills are markdown instruction files — their power is in the prompt engineering, not executable logic. Treat skills as first-class artifacts: version them, document them in MEMORY.md, and review them for quality like any other project asset.

34. **Generic dev skills conflict with governance.** Installing toolkit skills (like gstack's `/ship` or `/review`) into a governed project bypasses spec-first, deploy-gate, and KB-aware workflows. Project-specific skills should always override generic ones. A generic `/ship` that auto-opens PRs violates GOV-16 (deployment approval gate).

35. **Cost-annotate your test tiers.** Labeling each test phase with its approximate cost and duration (e.g., "E2E: ~10 min, ~$0.50 staging infra" vs "Unit: ~1 min, free") helps teams make informed decisions about which tiers to run on every commit vs nightly vs pre-release. Without cost visibility, teams either run everything (slow) or skip expensive tiers (risky).

36. **Template-driven skill generation prevents drift.** For projects with many skills, a `.tmpl` → `.md` generation pipeline with CI checks ensures manual edits do not silently diverge from the canonical template. This is the skill equivalent of "generated code must match its schema" — a pattern that scales better than code review alone.

37. **Data-driven rate limits outperform hardcoded tiers.** Deriving rate limits from measured capacity (ramp-to-overload testing → safe RPM per replica → uniform limit) produces a defensible number. Hardcoded per-tier limits are arbitrary and create customer friction without improving system safety. Measure first, then set policy.

38. **Entitlement-as-data eliminates hardcoded tier gates.** Moving tier definitions from source code constants to a database-backed EntitlementService (with cache hierarchy: LRU → Redis → Cosmos) enables runtime configuration changes without redeployment. Every hardcoded `TIER_DEFAULTS[tier]` reference is a future incident.

39. **Multi-agent coordination requires message discipline.** When multiple AI agents work on the same project (e.g., Prime Builder + Loyal Opposition), structured message passing with explicit fields (subject, body, priority, tags) prevents findings from being lost in conversation context. The Loyal Opposition must never write code — only file evidence-based reports.

40. **E2E selector failures are symptoms, not root causes.** When UI tests fail because selectors don't match, the fix is updating selectors — but the root cause is usually untracked UI changes. GOV-14 (UI test sync) exists because selector maintenance is inevitable. Budget for it in every session that touches frontend code.

41. **Layered defense catches what single-layer enforcement misses.** PreToolUse hooks catch real-time mistakes, pre-commit hooks catch commit-time mistakes, SessionStart hooks catch inter-session drift. No single layer covers all failure modes. The redundancy is deliberate — when Layer 1 misses a credential in a test file (reasoning "this is just a test"), Layer 2 catches it at commit time. When Layer 2 is bypassed (external editor), Layer 3 catches it next session. The maintenance cost of 3 layers is justified by the elimination of silent quality regression.

42. **Pre-commit assertion ratchets prevent invisible test weakening.** Counting assertions per test file and rejecting commits that reduce any count prevents a subtle failure mode: Claude "simplifying" tests by removing edge-case assertions while keeping the test nominally passing. The ratchet makes assertion removal require explicit owner approval, converting a silent regression into a visible decision.

43. **Procedure encoding reduces orchestration errors by 10x.** Moving from prose instructions in CLAUDE.md to executable skills reduced GOV-12 violations from ~30% to ~3% of sessions. The remaining violations occur during novel work outside any skill. The key insight: Claude is reliable at following explicit steps but unreliable at reconstructing multi-step procedures from principles. Skills are prompt engineering applied to procedures.

44. **Fail-closed vs fail-open is the critical hook design decision.** Credential scanners should fail-open (only block on positive match — a parse error should not prevent all file writes). Destructive command gates should fail-closed (a pattern-matching exception should block the command, not allow it). Getting this wrong either blocks all work (false fail-closed) or allows the exact thing the hook was designed to prevent (false fail-open).

45. **Two-container test architecture isolates test execution from production traffic.** Running tests inside the API container risks resource contention and accidental state mutation. A dedicated test host container with internal-only ingress (no public access) runs tests against the API container via private networking. This keeps test infrastructure completely invisible to customers while sharing the same Azure Container Apps environment for low-latency communication.

46. **Session-scoped browser instances prevent E2E OOM kills.** Running 272 Playwright test classes with per-class browser launches requires ~40 GB (150 MB per Chromium instance). Session-scoping the browser (one Chromium instance shared across all test classes via a conftest.py fixture) reduced memory to ~200 MB. The fix is a single conftest override — the tests themselves need no changes.

---

## Glossary

These terms have specific meanings in the Membase pattern. Each corresponds to a real database table or a documented process — there are no abstract concepts without backing implementation.

### Artifact Types

| Term | Table | Definition |
|------|-------|-----------|
| **Specification** | `specifications` | A requirement — a business decision that affects customers or the business. Litmus test: "Would a different choice affect the customer or the business?" If yes, it is a specification. If no, it is an implementation detail. Functions as a decision log (what was agreed and why), not a build specification (how to construct). Status lifecycle: `specified` → `implemented` → `verified` → `retired`. |
| **Test** | `tests` | An individual testable assertion derived from a specification. Three valid forms: (1) logical assertion — exists/doesn't exist, comparisons, if-then; (2) user story — a verifiable process a user performs; (3) abstract description — measurements, pseudocode, or information describing the desired behavior. Must produce an unambiguous PASS/FAIL. |
| **Test Plan** | `test_plans` | An orchestrating artifact: ordered test phases with gate criteria. References test IDs without duplicating content. Each phase defines what must pass before proceeding to the next. |
| **Work Item** | `work_items` | A unit of work classified by origin (`regression`, `defect`, `new`, `hygiene`) and component. Stage lifecycle: `created` → `tested` → `backlogged` → `implementing` → `resolved`. Links to a source specification and optionally to a test that revealed it. |
| **Backlog Snapshot** | `backlog_snapshots` | A point-in-time snapshot of active work items. Not a living document — a frozen record of what was open at a specific moment. Ordering within the snapshot determines implementation priority. |
| **Operational Procedure** | `operational_procedures` | A step-by-step repeatable process: deployment, verification, audit, recovery. Versioned like all artifacts. |
| **Document** | `documents` | General-purpose project knowledge under change control. Replaces drifting markdown topic files. Anything that is "project knowledge" but not a specification, test, or procedure belongs here. |
| **Environment Config** | `environment_config` | Environment-specific values (URLs, connection strings, thresholds) under change control. Tracks what each environment is configured to use. |
| **Testable Element** | `testable_elements` | A UI component, form field, or interactive element inventoried for systematic test coverage. Each element has a unique ID (e.g., EL-dashboard-001), maps to a page/subsystem, and is categorized using a 14-dimension taxonomy (A: visibility, B: content, C: interaction, etc.). Enables "everything is tested" rather than "tests exist." |

### Supporting Records

| Term | Table | Definition |
|------|-------|-----------|
| **Assertion Run** | `assertion_runs` | A historical record of a machine-verifiable assertion execution. Records which spec was checked, whether it passed, the detailed results, and what triggered the run. |
| **Session Prompt** | `session_prompts` | A structured handoff message stored by one session for the next. Contains what was done, what's next, and key context. Marked as consumed after the receiving session reads it. |

### Concepts (Not Tables)

| Term | Definition |
|------|-----------|
| **Assertion** | A machine-verifiable check attached to a specification. Three types: `grep` (pattern must exist in file), `grep_absent` (pattern must NOT exist in file), `glob` (file path must exist). Stored as JSON in the specification's `assertions` column. Runs automatically at session start via the SessionStart hook. |
| **Phantom Artifact** | A concept referenced as if it were a tracked entity, but with no backing storage, change control, or queryable history. Example: saying "the backlog contains WI-42" when no backlog table exists. The anti-pattern this entire system was designed to eliminate. |
| **Orchestrating Artifact** | An artifact that composes other artifacts by reference (ID only), never by content duplication. Test plans reference test IDs; backlog snapshots reference work item IDs. Each referenced artifact is independently managed and versioned. Prevents content duplication and the drift that accompanies it. |
| **Governance Principle** | A process rule (GOV-\*) governing how the human-AI team works together. Discovered through real project failures, not designed upfront. Currently 20 principles: 18 numbered rules (GOV-01 through GOV-18) plus 2 architectural principles (SPEC-1493: Artifact Inventory, SPEC-1499: Orchestrating Artifact). |
| **Testable Element** | A UI component, form field, or interactive element inventoried for systematic test coverage tracking. Stored in the `testable_elements` table with a 14-dimension taxonomy (A–N: visibility, content, interaction, state, etc.). Each element has a unique ID (e.g., EL-dashboard-001) and maps to specific tests. |
| **Quality Dashboard** | A 4-metric display rendered at every session start by the SessionStart hook. Metrics: assertion coverage (target ≥60%), test traceability (target >80%), defect velocity (net resolved vs open), defect escape rate (production incidents). Uses emoji indicators and box-drawing characters for visual clarity. |
| **Protected Behavior** | A specification with `type = 'protected_behavior'` carrying machine-verifiable assertions that must always pass. Checked in build gates before every deployment. |
| **Append-Only Change Control** | The versioning discipline: no UPDATE in place, no DELETE. Every mutation creates a new versioned row with mandatory `changed_by`, `changed_at`, and `change_reason`. Current state = latest version per ID (via SQL views). Full audit trail preserved indefinitely. |
| **Session Handoff** | The mechanism by which one session stores context for the next. The previous session calls `db.insert_session_prompt()` with a structured prompt; the SessionStart hook retrieves and displays it, then marks it consumed. Eliminates the human needing to craft "Continue work on X..." prompts. |
| **Specify on Contact** | Governance principle (GOV-06): when Claude touches unspecified code, it becomes controlled. Any existing behavior that is modified should first be recorded as a specification before changes are made. |
| **Audit Session** | Every Nth session (default: 5) is automatically flagged for a fresh-context integrity review. The audit covers KB assertions, MEMORY.md accuracy, procedure correctness, and design debt. Compensates for drift that accumulates incrementally across sessions. |
| **Skill** | A Claude Code SKILL.md file that encodes a repeatable workflow as an executable playbook. Stored in `.claude/skills/<name>/SKILL.md` with YAML frontmatter controlling invocation mode and tool access. Skills mechanize governance chains that previously relied on Claude's self-discipline. Follows the [Agent Skills](https://agentskills.io) open standard. |

---

## Benefits & Milestones

Membase was not designed upfront — it evolved through real project needs. Each capability below was added in response to a specific problem encountered during production use.

### Evolution Timeline

| Session | Problem Encountered | Capability Added |
|---------|-------------------|-----------------|
| S1–S95 | Markdown backlogs drift; concepts referenced but not tracked | — (markdown-only era) |
| S96 | No machine-verifiable truth about what is implemented | **Knowledge Database** — append-only SQLite, 161 initial specs, assertions, web UI |
| S104–S106 | Requirements scattered across 90 session transcripts | **Specification Discipline** — 871 specs extracted, governance principles crystallized, specification litmus test defined |
| S109 | No way to measure which specs have tests | **Test Coverage** — `test_coverage` table, 1,230 test-to-spec mappings, 26% initial coverage |
| S110 | Topic files contradict each other and the database | **GOV-08** — 30 topic files migrated to KB documents under change control |
| S112 | Low spec coverage despite many tests | **Test Coverage Sprint** — 1,053 new tests, coverage 26% → 59% |
| S113–S114 | "Backlog" and "test plan" referenced but not actually tracked | **Artifact System Redesign** — 5 new tables, phantom artifacts eliminated, orchestrating artifact principle |
| S116–S117 | Claude implements before recording specs from owner requirements | **GOV-09 + GOV-10** — spec-language detection hook (S116), tests must exercise production interfaces (S117) |
| S128 | Work items created without tests; no lifecycle tracking | **GOV-12** — work item creation triggers test creation; `stage` column with transition enforcement |
| S129 | Tests orphaned from test plan phases | **GOV-13** — test artifacts must be assigned to at least one test plan phase upon creation |
| S131 | UI changes break tests silently; tests fixed without approval; deploys without approval | **GOV-14–16** — UI element test sync, test fix approval gate, deployment approval gate |
| S131 | Manual multi-step deployment process is error-prone | **Automated deploy pipeline** — single-invocation 15-phase staging / 12-phase production pipeline |
| S133 | Mocked/inspection tests give false confidence; can't verify production behavior | **SPEC-1649** — Master Test Plan converted to live-only external interfaces (13 active phases, 3 removed) |
| S135 | No systematic inventory of what should be tested in the UI | **SPEC-1652/1653** — Testable element inventory (520 elements), 4-phase quality cycle, dimensional taxonomy (14 categories A–N) |
| S135–S144 | Shallow live E2E tests miss real bugs | **936 live E2E tests** across 3 admin consoles (Standalone 576, Provider 264, Shopify 96), all exercising real staging APIs |
| S145 | No visibility into quality metrics at session start | **Quality Dashboard** (SPEC-1659/1660) — 4 metrics at every session start: assertion coverage, test traceability, defect velocity, defect escape rate |
| S146–S150 | 87% of specs lack machine-verifiable assertions | **Batch assertion generation** — 3-tier strategy (grep in known file, glob, keyword mapping). Coverage 12.9% → 99.5%. |
| S153 | Specs classified as "not yet implemented" that are actually implemented | **Verification mega-session** — 400 spec promotions, 148 retirements, GOV-18 (Assertion Quality Standard) |
| S157 | SPA console shares auth layer with tenant endpoints | **SPEC-1667/1668** — Complete SPA platform admin auth isolation with dedicated Cosmos collection and key prefix |
| S158 | assertion_runs table grows unboundedly (~230K rows, ~93 MB) | **Assertion pruning** — keeps latest 5 runs per spec, automated in SessionStart hook. DB 93 → 18 MB. |
| S161 | Quality evaluation exposed auth, rate-limit, and CI gaps | **Quality evaluation remediation** — auth hardening, rate limit backend consolidation, CI/CD tooling |
| S165–S166 | Need zero-backend UI development and testing | **Mock dev environment** + 527 mock E2E tests (SPEC-1706), zero-backend testing |
| S169 | 5,085-line monolith file blocks team scalability | **Superadmin API split** — monolith decomposed into 5 domain submodules |
| S173 | 74 stale assertion failures after package split | **Deep hygiene** — assertion remapping after monolith split, 17 spec retirements, 1,621/1,621 assertions passing |
| S174–S183 | Multi-agent containerization and AGNTCY SDK integration needed | **Agent containerization** — 7 agent containers, NATS JetStream, transport hierarchy (SLIM/NATS/HTTP), AGNTCY Directory, load testing |
| S184–S185 | Rate limits arbitrary per tier; no empirical basis | **Data-driven rate limiting** — ramp-to-overload testing derived 300 RPM safe capacity, uniform limit, per-tier caps removed |
| S187 | Specs claim "implemented" but code doesn't match | **Full spec-vs-code verification** — all 2,009 specs verified against source, 3 mismatches found |
| S190 | Repeated multi-step workflows executed manually each session | **Claude Code Skills** — 8 KB-aware skills (`/deploy`, `/seed-tenant`, `/kb-session-wrap`, `/run-tests`, `/kb-query`, `/kb-spec`, `/kb-work-item`, `/kb-promote`) |
| S191 | Hardcoded tier gates scattered across 11 source locations | **EntitlementService** — data-driven tier config (LRU → Redis → Cosmos cache), 18 SPA Control Plane specs, feature flags |
| S192–S196 | No runtime operational control without redeployment | **SPA Operational Control Plane** — 39 API endpoints, 9 SPA pages, entitlement CRUD, block lists, maintenance mode, deployment orchestrator |
| S198–S201 | No systematic quality measurement beyond test counts | **Quality Measurement** — quality score API (6 metrics), Schemathesis API fuzzing (307 ops), Hypothesis property-based tests (46), mutmut mutation testing, coverage enforcement |
| S202–S206 | 55-spec feature backlog (BACKLOG-018) requiring 6 implementation phases | **BACKLOG-018 complete** — A/B testing (5 specs), integration framework (18 specs, 402 tests), MCP agents (7 specs, 100 tests), pipeline observatory (10 specs), deferred features (9 specs). All 55 specs implemented. |
| S204 | Multiple AI agents need structured coordination | **Prime-bridge multi-agent coordination** — Loyal Opposition (Codex) + Prime Builder (Claude Code) with structured message passing, evidence-based reports |

### Current State (Session 206)

| Category | Metric | Count |
|----------|--------|-------|
| **Specifications** | Total | 2,052 |
| | Verified | 331 |
| | Implemented | 1,486 |
| | Specified (not yet implemented) | 40 |
| | Retired | 195 |
| | Governance (GOV-01 through GOV-18) | 20 |
| **Tests** | Test artifacts (spec-linked) | 10,847 |
| | Automated tests passing | 6,920 (unit + multi-tenant + chat + integration) |
| | Live E2E tests (3 admin consoles) | 1,050 |
| | Property-based tests (Hypothesis) | 46 |
| **Work Items** | Total | ~1,600 |
| | Open | 240 |
| **Assertions** | Specs with machine-verifiable assertions | ~2,040 (99.5% coverage) |
| **Test Plan** | Active phases (PLAN-001) | 18 (incl. fuzzing + property phases) |
| **Quality** | Testable elements inventoried | 520 |
| | Quality dashboard metrics | 6 (quality score API) |
| | API fuzzing operations (Schemathesis) | 307 |
| **Knowledge** | Documents under change control | 176 |
| | Operational procedures | 14 |
| | Governance principles | 20 (18 numbered + 2 architectural) |
| **Skills** | Claude Code skills (KB-aware) | 8 |
| **Infrastructure** | Production tenants | 20 |
| | Agent containers | 7 |
| | SPA Control Plane endpoints | 39 |
| **Database** | Database size | ~40 MB |
| | Data loss incidents | 0 |

### What the System Catches

The following failure modes are detected automatically, without human intervention:

- **Regressions at session start** — assertions run before work begins, flagging any spec whose implementation has drifted since the last session
- **Phantom artifact references** — the database enforces that every referenced concept has backing storage and change control
- **Specification drift** — append-only versioning preserves the full decision history; nothing is silently overwritten
- **Cold-start amnesia** — session handoff prompts give each new session the context it needs without human re-explanation
- **Accumulated process drift** — every 5th session is an audit, catching errors that compound across sessions
- **Untested specifications** — `get_untested_specs()` identifies coverage gaps on demand
- **Premature implementation** — GOV-09 hook detects spec language and enforces spec-first workflow; GOV-12 stage gates prevent implementation without tests
- **False test confidence** — SPEC-1649 mandates live-only external interface testing; mocked tests that pass locally but miss production bugs are excluded from the test plan
- **Unauthorized changes** — GOV-15 requires owner approval before fixing failed tests (could be a test bug or product bug); GOV-16 requires owner approval before any deployment
- **Quality degradation** — the quality dashboard displays 4 metrics at every session start; red indicators become the first priority before new work begins
- **Weak assertions** — GOV-18 enforces assertion quality: assertions must target specific code identifiers, not generic patterns that match broadly

### What the System Does Not Measure

Honesty about limitations: Membase does not track session duration, so there are no "time saved" claims. There is no control group (a parallel project without Membase), so there are no comparative productivity metrics. The benefits above are factual — what exists, what was built, what is caught — not extrapolations.

---

## Quick Start Checklist

- [ ] Create `tools/knowledge-db/db.py` with append-only schema (start with specifications + operational_procedures)
- [ ] Create `tools/knowledge-db/assertions.py` with grep/glob/grep_absent types
- [ ] Create `tools/knowledge-db/seed.py` to bootstrap from existing artifacts
- [ ] Create `tools/knowledge-db/app.py` for read-only web UI
- [ ] Create `.claude/hooks/assertion-check.py` (SessionStart hook)
- [ ] Register hooks in `.claude/settings.local.json`
- [ ] Add Knowledge Database section to CLAUDE.md
- [ ] Add session handoff instructions to CLAUDE.md
- [ ] Run `python tools/knowledge-db/seed.py` to populate initial data
- [ ] Verify assertions run at session start
- [ ] Optionally create `.claude/hooks/scheduler.py` + `.claude/SCHEDULE.md` (session scheduler)
- [ ] Add extended tables (tests, test_plans, work_items, documents, testable_elements) when your project grows
- [ ] Add quality dashboard metrics to the SessionStart hook (assertion coverage, test traceability, defect velocity, escape rate)
- [ ] Add assertion run pruning to the SessionStart hook (keep latest N per spec to prevent unbounded growth)
- [ ] Create project-specific skills in `.claude/skills/` that read from and write to the KB (Step 9)
- [ ] Create a single-entry-point test pipeline script with phase ordering (Step 10)
- [ ] Automate deployment with build-deploy-verify pattern and GOV-16 gate (Step 11)
- [ ] Set up multi-agent coordination if using Loyal Opposition or background agents (Step 12)
- [ ] Create PreToolUse hooks for credential scanning and destructive command blocking (Step 13)
- [ ] Create git pre-commit hooks: assertion ratchet, test deletion guard, architectural guards, TSX gate, credential scan (Step 13)
- [ ] Generate assertion baseline JSON for the ratchet hook (Step 13)
- [ ] Create custom review agents (code-reviewer, security-analyzer) in `.claude/agents/` (Step 13)
- [ ] Expand test taxonomy beyond unit/integration/e2e: add security, regression, fuzzing, property-based, load suites (Step 14)
- [ ] Integrate Schemathesis for OpenAPI contract fuzzing and Hypothesis for property-based testing (Step 14)
- [ ] Document known testing gaps honestly (Step 14)
- [ ] Identify repeatable procedures (>3 manual executions) and encode them as skills (Step 15)
- [ ] Move enforcement from CLAUDE.md prose to hooks and pre-commit checks (Step 15)

---

*This pattern was developed across 212 sessions on the Agent Red Customer Experience project by Remaker Digital. The implementation approach is freely reusable under the MIT license. Adapt the schema to your project's needs — the core principles (append-only, machine-verifiable assertions, live-only test verification, quality dashboard, governance discipline, session handoff, audit cadence, KB-aware skills, test pipeline architecture, deployment automation, multi-agent coordination, defense-in-depth enforcement, procedure encoding) are universal.*

*© 2026 Remaker Digital, a DBA of VanDusen & Palmeter, LLC. All rights reserved.*
