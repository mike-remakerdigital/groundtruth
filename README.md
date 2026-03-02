# Membase for Claude

**Persistent, version-controlled, self-auditing project memory for Claude Code — backed by an append-only SQLite database.**

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

> Claude Code needed a project knowledge database, so it built one.

---

## What This Is

Membase is a **pattern** (not a library) for giving Claude Code durable memory that survives across sessions. It replaces fragile markdown backlogs with a structured, append-only SQLite database where every change is versioned and every claim is machine-verifiable.

Developed across 128 sessions on a commercial SaaS project, it solves three problems that emerge in long-running Claude Code projects:

1. **Context window saturation** — long sessions accumulate stale context that biases decisions.
2. **Session boundary amnesia** — each new session starts cold; CLAUDE.md and MEMORY.md help but drift over time.
3. **Undetected regression** — code changes silently break previously-verified behavior with no automated signal.

## How It Works

Claude is the **sole writer**. The human observes through a read-only web UI. The database stores 8 managed artifact types — specifications, tests, test plans, work items, backlog snapshots, operational procedures, documents, and environment config — all under append-only change control. Machine-verifiable assertions (grep/glob checks against the actual codebase) run automatically at session start, catching regressions before work begins. A session handoff system eliminates cold-start friction by having each session store context for the next one.

The markdown files (CLAUDE.md, MEMORY.md, topic files) still matter — they store rules, preferences, and operational patterns that don't fit a relational model. The database complements them with formal artifacts and machine-verifiable truth. The governance principle is simple: **if Claude references something, it must exist; if it exists, it must be under change control; if it's under change control, its history must be retrievable.**

## Getting Started

### Prerequisites

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) (CLI)
- Python 3.10+
- SQLite 3 (included with Python)
- Flask (for the optional read-only web UI)

### Usage

1. Download [`MEMBASE-4-CLAUDE.md`](MEMBASE-4-CLAUDE.md) into your project
2. Ask Claude to read it:

```
Read MEMBASE-4-CLAUDE.md and set up the Membase knowledge database for this project.
```

That's it. The file contains the complete implementation pattern — schema, API, assertion runner, session hooks, governance principles, and web UI — with enough detail for Claude to reproduce it and adapt it to your project.

## What's in This Repo

| File | Purpose |
|------|---------|
| [`MEMBASE-4-CLAUDE.md`](MEMBASE-4-CLAUDE.md) | Complete implementation guide (the pattern document Claude reads) |
| [`README.md`](README.md) | This file — context and motivation |
| [`LICENSE`](LICENSE) | MIT License |

## Glossary of Key Terms

These terms have specific meanings in the Membase pattern. Understanding them is essential for working with the system.

| Term | Definition |
|------|-----------|
| **Specification** | A requirement — a business decision that affects customers or the business. Functions as a decision log (what was agreed and why), not a build specification. |
| **Test** | A testable assertion derived from a specification. Must produce an unambiguous PASS/FAIL. |
| **Test Plan** | An orchestrating artifact: ordered test phases with gate criteria. References test IDs without duplicating content. |
| **Work Item** | A unit of work classified by origin (regression, defect, new, hygiene) and component. |
| **Backlog Snapshot** | A point-in-time snapshot of active work items. A frozen record, not a living document. |
| **Assertion** | A machine-verifiable check (grep/glob) attached to a specification. Turns "Claude remembers" into "Claude proves." |
| **Phantom Artifact** | A concept referenced as if tracked, but with no backing storage or change control. The anti-pattern this system eliminates. |
| **Orchestrating Artifact** | An artifact that composes others by reference (ID only), never by content duplication. |
| **Governance Principle** | A process rule (GOV-\*) governing human-AI collaboration. 12 numbered rules (GOV-01–GOV-12) plus 2 architectural principles (Artifact Inventory, Orchestrating Artifact). Discovered through use, not designed upfront. |
| **Append-Only** | The change control discipline: no UPDATE, no DELETE. Every mutation creates a new versioned row. |
| **Session Handoff** | A structured prompt stored by one session for the next, eliminating cold-start friction. |
| **Protected Behavior** | A specification carrying machine-verifiable assertions that must always pass. |

See the [full glossary with examples](MEMBASE-4-CLAUDE.md#glossary) in the implementation guide.

## Benefits & Milestones

Membase was not designed upfront — it evolved through real project needs across 128 sessions. The milestones below trace how each capability was added in response to a specific problem.

### Evolution Timeline

| Session | Problem Encountered | Capability Added |
|---------|-------------------|-----------------|
| S1–S95 | Markdown backlogs drift; concepts referenced but not tracked | — (markdown-only era) |
| S96 | No machine-verifiable truth about what's implemented | **Knowledge Database** — append-only SQLite, assertions, web UI |
| S104–S106 | Requirements scattered across 90 session transcripts | **Specification Discipline** — 871 specs extracted, governance principles crystallized |
| S109 | No way to measure which specs have tests | **Test Coverage** — `test_coverage` table, 1,230 test-to-spec mappings |
| S110 | Topic files contradict each other and the database | **GOV-08** — 30 topic files migrated to KB documents under change control |
| S112 | No way to measure which specs have tests | **Test Coverage Sprint** — 1,053 new tests, spec coverage 26% → 59% |
| S113–S114 | "Backlog" and "test plan" referenced but not actually tracked artifacts | **Artifact System Redesign** — 5 new tables, phantom artifacts eliminated |
| S116–S117 | Claude implements before recording specs from owner requirements | **GOV-09 + GOV-10** — spec-language detection hook (S116), tests must exercise production interfaces (S117) |
| S128 | Work items created without tests; no lifecycle tracking | **GOV-12** — work item creation triggers test creation; stage-gate transitions |

### Current Database (as of Session 128)

| Metric | Count |
|--------|-------|
| Specifications | 1,803 (309 verified, 795 implemented, 695 specified) |
| Test artifacts | 2,797 (linked to specifications) |
| Work items | 917 (854 resolved, 27 open) |
| Machine-verifiable assertions | 180 (127 PASS, 53 expected FAIL — specified but unimplemented) |
| Assertion run records | 19,092 |
| Knowledge documents | 138 |
| Operational procedures | 13 |
| Governance principles | 14 (GOV-01 through GOV-12 + 2 architectural) |
| Versioned artifact rows | 8,443 (complete change history) |
| Automated tests passing | 5,962 |
| Database size | 14.6 MB |
| Data loss incidents | 0 |

### What the System Catches

- **Regressions at session start** — assertions run automatically before work begins, flagging any spec whose implementation has drifted
- **Phantom artifact references** — the database enforces that every referenced concept has backing storage and change control
- **Specification drift** — append-only versioning preserves the full decision history; nothing is silently overwritten
- **Cold-start amnesia** — session handoff prompts give each new session the context it needs without human re-explanation
- **Accumulated process drift** — every 5th session is an audit, catching errors that compound across sessions
- **Untested specifications** — `get_untested_specs()` identifies coverage gaps on demand

## Why Not Just Use Markdown?

The markdown-only approach (CLAUDE.md + MEMORY.md + topic files) works well for the first ~20 sessions. After that:

- Files grow unwieldy and contradictions accumulate
- There's no machine-verifiable way to know if what Claude "remembers" is still true
- MEMORY.md isn't version-controlled — there's no change history
- Claude can't partially load a document, so everything gets loaded every time
- Context window compaction makes drift worse, not better
- Claude silently "drifts," dropping memories and skipping procedures

Membase solves this by adding a structured layer with change control, versioned history, and automated verification — while keeping the markdown files for what they're good at (rules, preferences, operational patterns).

## Background

This pattern was developed incrementally on the [Agent Red Customer Experience](https://agentredcx.com) project by Remaker Digital. Working with an engineering team of one, the full weight of Agile and formal project management tools was unnecessary. What was needed was a shared, persistent, change-controlled memory between a human and an AI engineering partner.

The database is used exclusively by Claude and contains only what Claude needs to remember. The human observes through a lightweight read-only UI (sort, filter, search, tree-view, change history) that deliberately excludes write operations. When the human spots a discrepancy, they tell Claude, and Claude creates a corrected version.

The current database is ~14.6 MB with 1,803 specifications, 2,797 test artifacts, 1 test plan (16 phases), 917 work items, 13 operational procedures, 138 documents, 27 environment config entries, 180 machine-verifiable assertions (127 passing, 53 expected failures for unimplemented specs), and nearly 2,000 test-to-spec coverage mappings — all accumulated across 128 sessions with zero data loss.

---

*The implementation approach is freely reusable under the MIT license. Adapt the schema to your project's needs — the core principles (append-only, machine-verifiable assertions, session handoff, audit cadence) are universal.*

*© 2026 Remaker Digital, a DBA of VanDusen & Palmeter, LLC. All rights reserved.*
