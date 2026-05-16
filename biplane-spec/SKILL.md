---
name: biplane-spec
description: Fetch a Linear issue and rewrite it as a production-ready SDD spec (Fowler model). Writes user stories, GIVEN/WHEN/THEN acceptance criteria, functional and technical layers, and a lightweight testing strategy. Asks clarifying questions before writing. Stays purely at the behavior and contract level — no file paths, no framework names, no codebase grounding. Use when asked to enhance, spec out, or write requirements for a Linear issue. Pair with /biplane-build to ground the spec in the codebase before coding.
argument-hint: <ISSUE-ID> [--context "extra context"] [--focus "section to prioritize"]
disable-model-invocation: true
allowed-tools: Bash Read
---

You are writing a **Spec-Driven Development (SDD) spec** for a Linear issue following Martin Fowler's SDD model:

- **Specs describe intent (behavior), not implementation.** Functional requirements separate from technical design.
- **Acceptance criteria use GIVEN/WHEN/THEN** — behavior-framed, verifiable scenarios.
- **User stories ground the spec** — who needs what and why.
- **Testing strategy is part of the spec**, not an afterthought (kept lightweight).
- **Small, iterative scope** — one well-scoped ticket per spec.
- **Living document** — structured to evolve as implementation progresses.

This skill is project-agnostic and stays at the behavior/contract level. It never
names files, functions, or frameworks. Codebase grounding and concrete file
pointers are the job of `/biplane-build`, which runs after the spec is approved.

## Arguments

Parse `$ARGUMENTS`:
- **First token** (required): Linear issue ID, e.g. `AIT-17`
- `--context "..."` (optional): Background the user supplies (constraints, decisions already made, related tickets)
- `--focus "..."` (optional): Sections to prioritize, e.g. `"error handling"`, `"webhook shape"`, `"testing strategy"`

If the issue ID is missing, print: `Usage: /biplane-spec <ISSUE-ID> [--context "..."] [--focus "..."]` and stop.

---

## Phase 1 — Fetch & Read

1. Fetch the Linear issue via `mcp__linear-server__get_issue` (`id: <ISSUE-ID>`).
2. Read `title`, `description`, `status`, `parentId`.
3. If `parentId` exists, fetch the parent issue for broader goal context.

---

## Phase 2 — Gather Context

1. Fold in `--context` (decisions/constraints already made) and `--focus`
   (sections to prioritize).
2. If repo-level guidance docs are present, read them for terminology and product
   conventions — **discover, do not assume paths**. Look for whatever the project
   actually uses (e.g. an agent/contributor guide, a README, an architecture/design
   doc). Skip silently if none.
3. Do **not** open source files. This skill stays at the behavior level. Source
   exploration belongs to `/triplane-plan`.

---

## Phase 3 — Ask Clarifying Questions

Identify genuine decision forks. Use `AskUserQuestion` (2–4 questions max). Only
ask what's truly ambiguous — skip anything answered by `--context`, `--focus`, or
the parent issue.

**Common forks:**
- Data source (scrape / upstream API / internal)
- Auth/access method (anonymous / credentialed)
- Rate-limiting strategy
- Async vs. sync delivery
- Failure behavior (silent / error webhook / 4xx)
- Scope boundaries (this ticket vs. follow-up)

---

## Phase 4 — Write the SDD Spec

Structure in **three clearly separated layers**.

---

### LAYER 1 — Functional Specification
*(WHAT the system does and for WHOM — no file paths or function names)*

**## Overview**
2–4 sentences: problem solved, who's affected, current broken/mock state → target state.

**## User Stories**
As a [role], I want to [action] so that [outcome]. (1–3 stories, each traceable to a criterion)

**## Functional Requirements**
Numbered list of behavioral mandates — observable outcomes, plain language, no code.

```
1. The system must return a response within 500ms for valid requests.
2. Zero matches must be delivered as an empty result set, not an error.
```

**## Acceptance Criteria**
GIVEN/WHEN/THEN — one verifiable scenario per criterion. Group: Happy Path → Error Cases → Edge Cases → Infra/Config.

```
GIVEN a valid access token and api key
WHEN the create-token endpoint is called with correct credentials
THEN the response is 200 with token, refreshToken, tokenType, expiresIn

GIVEN a refresh token that has already been used
WHEN the refresh endpoint is called with it
THEN the response is 401 UNAUTHORIZED
```

No vague language: "correctly", "properly", "handles errors" are not allowed.

**## Out of Scope**
Explicit list. Name each deferred item and which follow-up ticket will handle it.

---

### LAYER 2 — Technical Design
*(HOW — architecture and contracts. Behavior/contract level only — no file paths
or function names. Those land in the implementation plan.)*

**## Endpoint(s)** — HTTP method + path + Sync/Async label

**## Headers** — Table: Key / Value / Required

**## Request Contract** — Table: Field / Type / Required / Validation / Description.
Describe the contract abstractly. Do not name the project's validation library or schema location.

**## Response Contracts** — Exact JSON for each case (realistic values):
- Synchronous response
- Async webhook success payload
- Async webhook failure envelope (if async)
- Error shape

**## Flow Diagram** — Mermaid `sequenceDiagram`, every actor and hop, alt blocks for every error path

**## Data Model Changes** — New persistence/columns/keys at the conceptual level
(entity, fields, constraints). State "No schema changes" if none. Migration
mechanics belong in the plan.

**## Error Handling** — Table: Scenario / HTTP Status / Error Code / Retry-safe?

**## Config / Env** — Table: Variable / Default / Purpose. List every config the
behavior depends on. Whether each is new or already present is determined by the plan.

---

### LAYER 3 — Testing Strategy  *(lightweight)*
*(What proves the spec is satisfied — written before code)*

**## Testing Strategy**

A brief outline across three tiers — a few bullets each, not exhaustive plans:

- **Unit** — pure logic to isolate; what to mock vs. not mock (described by role, not by file).
- **Integration** — component-boundary behaviors; happy path + key error scenarios; which external deps need test doubles.
- **E2E / Contract** (if applicable) — full async/webhook flow.

Describe scope by behavior. Concrete test file locations are the plan's job.

---

## Phase 5 — Self-Review Checklist

Before posting, verify all pass:

- [ ] Every user story has ≥1 GIVEN/WHEN/THEN criterion
- [ ] Every criterion is independently verifiable without running the full system
- [ ] No criterion uses vague language ("correctly", "properly", "handles errors")
- [ ] Functional layer has zero file paths or function names
- [ ] Technical layer has zero file paths, function names, or framework names
- [ ] Technical layer uses only "must"/"returns"/"stores" — no "should"/"might"/"could"
- [ ] Testing strategy present (lightweight tiers); describes scope by behavior, not by file
- [ ] Out of scope explicitly names deferred items
- [ ] No invented file paths, library names, or framework names anywhere

Spec size limits (Fowler: verbose specs = known failure mode):
- Functional Requirements: ≤8 items
- Acceptance Criteria: ≤12 GIVEN/WHEN/THEN scenarios
- Total: 5-minute read target

---

## Phase 6 — Update Linear & Summary

1. Call `mcp__linear-server__save_issue` (`id: <ISSUE-ID>`, `description: <full spec>`).
2. Print a concise summary (≤8 bullets): sections added, key decisions baked in,
   open questions deferred to out-of-scope.
3. Remind the user:
   - **This spec is a living document** — update it when implementation reveals new constraints; acceptance criteria are the definition of done.
   - **Next step:** run `/biplane-build <ISSUE-ID>` to produce a codebase-grounded implementation plan, review it, then implement.
