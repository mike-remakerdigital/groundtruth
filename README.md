# Membase for Claude

**Persistent, version-controlled, self-auditing project memory for Claude Code — backed by an append-only SQLite database.**

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

> Claude Code needed a project knowledge database, so it built one.

---

## What This Is

Membase is a **pattern** (not a library) for giving Claude Code durable memory that survives across sessions. It replaces fragile markdown backlogs with a structured, append-only SQLite database where every change is versioned and every claim is machine-verifiable.

Developed across 125+ sessions on a commercial SaaS project, it solves three problems that emerge in long-running Claude Code projects:

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

## Key Concepts

- **8 managed artifact types** — Specifications, tests, test plans, work items, backlog snapshots, operational procedures, documents, and environment config. Each is versioned, queryable, and under change control.
- **Append-only** — No rows are ever updated or deleted. Every change creates a new versioned record. Current state = latest version per ID. Full audit trail, no accidental data loss.
- **Machine-verifiable assertions** — Each specification can have grep/glob assertions that Claude runs automatically at session start. Turns "Claude remembers" into "Claude proves."
- **Governance principles** — 11 governance principles (GOV-01 through GOV-11) evolved through real project use. Specifications are the negotiation artifact between human and AI. They function as a decision log (what was agreed and why), not a build specification (how to construct).
- **Orchestrating artifacts** — Test plans and backlogs reference other artifacts by ID without duplicating content. Each referenced artifact is independently managed and versioned.
- **Session handoff** — The previous session stores a structured prompt for the next one, so Claude automatically knows what to work on.
- **Owner input classification** — A hook detects specification language ("must do," "should include") in user prompts and enforces the spec-first workflow before any implementation begins.
- **Audit cadence** — Every Nth session (default: 5) is flagged for a fresh-context integrity review, catching drift that accumulates across sessions.
- **Never-delete retention** — At ~20 KB/session with a 400 GB budget, the database can run indefinitely (~57,000 years at 1 session/day). Storage is not a constraint.

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

The current database is ~13 MB with 1,767 specifications, 2,725 test artifacts, 1 test plan (16 phases), 881 work items, 13 operational procedures, 138 documents, 27 environment config entries, 165 machine-verifiable assertions (117 machine-checkable, all passing), and nearly 2,000 test-to-spec coverage mappings — all accumulated across 125+ sessions with zero data loss.

---

*The implementation approach is freely reusable under the MIT license. Adapt the schema to your project's needs — the core principles (append-only, machine-verifiable assertions, session handoff, audit cadence) are universal.*

*© 2026 Remaker Digital, a DBA of VanDusen & Palmeter, LLC. All rights reserved.*
