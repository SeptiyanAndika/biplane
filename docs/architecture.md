# Biplane — Architecture

## The Two-Wing Model

Biplane splits spec-driven development into two separate Claude Code skills that must run in sequence.

| Wing | Skill | Scope |
|------|-------|-------|
| Upper | `/biplane-spec` | Behavioral intent — **no codebase access** |
| Lower | `/biplane-build` | Codebase-grounded plan + implementation |

The split is intentional. Specs written against the codebase drift toward describing what the code *does* rather than what it *should do*. Keeping the upper wing codebase-free means the spec stays valid across refactors and the grounding work happens explicitly and reviewably in the plan.

---

## Pipeline

### `/biplane-spec` — 6 phases

```
Phase 1  Fetch & Read
         └── mcp__linear-server__get_issue
         └── fetch parent issue if parentId exists

Phase 2  Gather Context
         └── read --context, --focus args
         └── discover project docs (docs/, README, architecture guides) — skip silently if none
         └── never open source files

Phase 3  Ask Clarifying Questions
         └── AskUserQuestion (2–4 questions max)
         └── only genuine decision forks not answered by context/focus/parent

Phase 4  Write the SDD Spec (3 layers)
         ├── LAYER 1: Functional Specification
         │   user stories · functional requirements · GIVEN/WHEN/THEN ACs · out of scope
         ├── LAYER 2: Technical Design
         │   endpoints · request/response contracts · Mermaid flow · data model · errors · env
         └── LAYER 3: Testing Strategy
             unit · integration · E2E tiers — described by behavior, not by file

Phase 5  Self-Review Checklist
         └── 9-point checklist; size limits: ≤8 functional reqs, ≤12 ACs, 5-min read

Phase 6  Update Linear & Summary
         └── mcp__linear-server__save_issue (writes spec to issue description)
         └── print summary ≤8 bullets
         └── remind user: next step is /biplane-build
```

### `/biplane-build` — Half 1 (Plan) + Half 2 (Execute)

```
HALF 1 — PLAN

Phase 1  Fetch the Spec
         └── mcp__linear-server__get_issue
         └── validate LAYER 1 / LAYER 2 structure exists — stop if not

Phase 2  Read Project Conventions
         └── discover docs/ (or documentation/, guides/, README, architecture doc)
         └── fold in --context, --focus

Phase 3  Explore the Codebase (parallel Explore agents)
         ├── Agent A — Implementation seam
         │   entry point · validation layer · service layer · mock code · existing tests
         └── Agent B — Conventions & reuse
             sibling feature structure · error/response patterns · config handling · test framework · utilities

Phase 4  Ask Clarifying Questions (max 2–3, only real implementation forks)

Phase 5  Write Pointer-Style Plan
         summary · files table · reuse · integration seams · new deps · migration · test plan · gotchas · order

Phase 6  Plan Self-Review Checklist (8-point)

Phase 7  Present Plan for Approval (plan mode)
         └── optionally post to Linear if --linear passed
         └── STOP — do not proceed until human approves

HALF 2 — EXECUTE

Phase 8   Build Verification Map
          └── each GIVEN/WHEN/THEN → test file or command → status: not started
          └── stop if any criterion has no verification method

Phase 9   Confirm Scope
          └── print file list · wait for explicit user confirmation

Phase 10  Implement, One Step at a Time
          └── follow Suggested Implementation Order
          └── verify each criterion after its step · mark green

Phase 11  Final Verification
          └── full test suite (or touched-files subset)
          └── walk every AC, confirm green

Phase 12  Execute Self-Review Checklist (9-point)

Phase 13  Report Back
          └── print function changes per file · AC verification table · notes
          └── ask whether to post summary to Linear
          └── NEVER commit or push
```

---

## State & Data Flow

```
Linear issue
     │  description ← spec written by /biplane-spec (Phase 6)
     │
     ├─ /biplane-build reads spec (Phase 1)
     │
     ├─ plan mode (Half 1 output) — ephemeral, shown inline
     │       optionally posted as Linear comment (--linear flag)
     │
     └─ implementation summary (Phase 13)
             posted as Linear comment only on user confirmation
```

Linear is the single source of truth. No local state files are written by either skill. The spec is the definition of done; the verification map is ephemeral within the session.

---

## Agent Topology

`/biplane-build` Phase 3 fans out to parallel subagents using the `Task` tool with `subagent_type: Explore`.

```
biplane-build
├── Explore Agent A  (implementation seam)
│   entry point · validation · service layer · mock replacements · existing tests
└── Explore Agent B  (conventions & reuse)
    sibling features · error patterns · config/env · test framework · reusable utilities
```

Agents return concrete paths and symbols. The plan is built from these findings — no invented paths.

---

## Size Limits

These are hard limits defined in the skill prompts, not guidelines:

| Artifact | Limit |
|----------|-------|
| Functional Requirements | ≤ 8 items |
| Acceptance Criteria | ≤ 12 GIVEN/WHEN/THEN scenarios |
| Spec total | 5-minute read |
| Plan: Files to Create/Modify | ≤ 12 rows |
| Plan: Gotchas | ≤ 6 bullets |
| Plan: Implementation Order | ≤ 8 steps |
| Plan total | 3-minute read |
