# Biplane

<p align="center">
  <img src="assets/biplane.png" alt="Biplane illustration" width="400" />
</p>

<h2 align="center">Two wings. One lift.</h2>
<p align="center">Spec what it should do. Build what it actually takes.</p>

**Biplane** is a pair of [Claude Code](https://claude.ai/code) skills for spec-driven development — translating Linear issues into production-ready code through a deliberate two-stage workflow.

---

## Philosophy

A [biplane](https://en.wikipedia.org/wiki/Biplane) generates lift through two wings working in concert. The upper wing and lower wing are structurally separate but aerodynamically inseparable — neither alone is sufficient. Remove one and the aircraft cannot fly.

Biplane the toolkit works the same way.

```
┌─────────────────────────────────────────────────────────────┐
│                         BIPLANE                             │
│                                                             │
│   ══════════════════════════════════   ← /biplane-spec      │
│         (upper wing — behavioral intent)                    │
│                 ╲     ╱                                     │
│                  ╲   ╱                                      │
│                   ╲ ╱                                       │
│   ══════════════════════════════════   ← /biplane-build     │
│         (lower wing — codebase reality)                     │
│                                                             │
│               ↓↓↓  LIFT  ↓↓↓                               │
│         (working, verified software)                        │
└─────────────────────────────────────────────────────────────┘
```

Early biplanes were built this way intentionally: two wings gave structural rigidity, better lift at lower speeds, and a natural separation of concerns. You knew exactly which surface was doing what. Biplane the toolkit applies the same principle to software delivery — separating the *what* (spec) from the *how* (build) so each can be reasoned about, reviewed, and corrected independently.

**The failure mode this prevents:** jumping straight to code against a vague ticket. That's not a monoplane — it's a wing with no support structure, and it will stall.

The lower wing carries its own structural discipline: `/biplane-build` flies on the [Karpathy coding principles](https://github.com/multica-ai/andrej-karpathy-skills) — the rules that keep an LLM from over-building and drifting off-task once it has code in its hands.

---

## The Three Pillars

These three pillars form the backbone of the spec — moving from *what* the system does, to *how* it's built, to *how* we prove it works.

### 1. Functional Specification

Describes what the system should do from a user's perspective. Bridges the gap between high-level business goals and actual implementation.

**Focus:** Features, workflows, and user interactions.

| Element | Description |
|---------|-------------|
| User Stories & Use Cases | Scenarios of how different users interact with the system |
| Requirements | "The system shall..." statements with observable outcomes |
| Business Rules | Constraints or logic the system must follow |
| Acceptance Criteria | GIVEN/WHEN/THEN — verifiable, unambiguous scenarios |

**Goal:** Align all stakeholders on expected deliverables before coding begins.

---

### 2. Technical Design

Provides the blueprint for how the functional requirements will be realized — a roadmap for the engineering team.

**Focus:** System architecture, data structures, and contracts.

| Element | Description |
|---------|-------------|
| System Architecture | Infrastructure, integrations, and technology decisions |
| Data Models | Database schemas, entity relationships, data flow |
| API Definitions | Endpoint structures, request/response formats |
| Error Handling | Status codes, failure modes, retry safety |
| Security & Config | Auth protocols, env vars, performance constraints |

**Goal:** Give developers clear, unambiguous instructions — minimize ambiguity, minimize technical debt.

---

### 3. Testing Strategy

Outlines the methodology for verifying the product meets both functional and technical requirements.

**Focus:** Confidence that the software is fit for purpose.

| Level | What it tests |
|-------|--------------|
| Unit | Individual logic in isolation |
| Integration | Component boundaries and how modules interact |
| System / E2E | The complete flow in a production-like environment |
| Acceptance (UAT) | End-user confirmation the product meets business needs |

**Goal:** Gain confidence in quality before a single line of production code ships.

---

## How It Works

### Stage 1 — `/biplane-spec <ISSUE-ID>`

The upper wing. Fetches a Linear issue and rewrites it as a production-ready **behavioral spec** using a spec-first approach.

It stays entirely at the **behavior and contract level** — no file paths, no framework names, no codebase assumptions. Its output is a living document structured in three layers:

| Layer | Content | Question answered |
|-------|---------|-------------------|
| **Functional Specification** | User stories, functional requirements, GIVEN/WHEN/THEN acceptance criteria, out-of-scope | *What should the system do, and for whom?* |
| **Technical Design** | Endpoints, request/response contracts, flow diagrams, data model changes, error handling | *What is the system's external contract?* |
| **Testing Strategy** | Unit, integration, and E2E tiers described by behavior | *What proves the spec is satisfied?* |

The spec is written back to the Linear issue description, making it the canonical definition of done. It asks clarifying questions before writing — only genuine decision forks, never questions answered by the ticket itself.

**Key constraint:** `/biplane-spec` never opens source files. Grounding the spec in the codebase is the job of the next stage.

---

### Stage 2 — `/biplane-build <ISSUE-ID>`

The lower wing. Reads the spec produced by `/biplane-spec`, grounds it in the real codebase, writes a pointer-style implementation plan, gets human approval, then implements.

It works in two halves:

#### Half 1 — Plan

1. Fetches the spec from Linear and validates it has the SDD structure
2. Reads project conventions (docs, READMEs, architecture guides — discovered, not assumed)
3. Launches up to 2 parallel **Explore agents** to map the codebase:
   - **Agent A** — Implementation seam: entry point, validation layer, service layer, mocks to replace, existing tests
   - **Agent B** — Conventions & reuse: sibling features, error patterns, config handling, test framework, utilities
4. Produces a **pointer-style plan** (skimmable in 3 minutes): files to change, reuse opportunities, integration seams, test plan, gotchas, implementation order
5. Enters **plan mode** for human review — nothing is written until approved

#### Half 2 — Execute

Only reached after the human approves the plan.

1. Builds a **verification map** — each acceptance criterion mapped to a concrete test or command before any file is touched
2. Confirms the exact file scope with the user
3. Implements one step at a time, verifying each criterion as it goes
4. Runs the full test suite; surfaces pre-existing failures without touching them
5. Reports function changes per file and asks whether to post the summary to Linear

**Four binding coding principles** govern the entire execution ([adapted from the Karpathy guidelines](https://github.com/multica-ai/andrej-karpathy-skills)):
- **Think before coding** — state assumptions, surface tradeoffs, ask when uncertain
- **Simplicity first** — minimum code that makes acceptance criteria pass, nothing more
- **Surgical changes** — touch only files in the plan; no "while I'm here" improvements
- **Goal-driven execution** — acceptance criteria are the verification loop; when all are green, stop

---

## Workflow

```
Linear issue
     │
     ▼
/biplane-spec <ISSUE-ID>
     │  • Asks 2–4 clarifying questions
     │  • Writes 3-layer SDD spec back to Linear
     │  • Spec becomes the definition of done
     ▼
Review & approve spec
     │
     ▼
/biplane-build <ISSUE-ID>
     │  • Grounds spec in codebase via Explore agents
     │  • Writes pointer-style implementation plan
     │  • Human approves plan (plan mode)
     │  • Builds verification map per acceptance criterion
     │  • Human confirms file scope
     │  • Implements step by step, verifies as it goes
     │  • Full test suite pass
     │  • Reports function changes
     ▼
git diff → review → commit
```

---

## Installation

Clone this repo and symlink (or copy) the skill directories into your Claude Code skills path:

```bash
git clone https://github.com/SeptiyanAndika/biplane.git
cd biplane

# Claude Code looks for skills in ~/.claude/skills/ by default
ln -s "$(pwd)/biplane-spec" ~/.claude/skills/biplane-spec
ln -s "$(pwd)/biplane-build" ~/.claude/skills/biplane-build
```

> Restart Claude Code after adding skills for them to be discovered.

---

## Requirements

- [Claude Code](https://claude.ai/code) CLI
- A Linear workspace with MCP integration (`mcp__linear-server` tools available in your Claude Code session)

> **Tip:** Add `docs/architecture.md` and `docs/conventions.md` to your project. Both skills auto-discover `docs/` during Phase 2 — the better your project docs, the better the plan.

---

## Usage

```
/biplane-spec <ISSUE-ID> [--context "..."] [--focus "..."]
/biplane-build <ISSUE-ID> [--linear] [--dry-run] [--context "..."] [--focus "..."]
```

**Options for `/biplane-spec`:**
- `--context` — constraints or decisions already made
- `--focus` — sections to prioritize (e.g. `"error handling"`, `"webhook shape"`)

**Options for `/biplane-build`:**
- `--linear` — also post the plan as a Linear comment before execution
- `--dry-run` — build the plan and verification map only; do not edit files
- `--context` — recent refactors, related PRs, or decisions to factor in
- `--focus` — narrow execution to a specific area or acceptance criterion

---

## Design Decisions

**Why two separate skills instead of one?**

The spec and the plan serve different audiences at different times. The spec is a stakeholder artifact — it describes intent, is written back to Linear, and should be readable without knowing the codebase. The plan is an engineer artifact — it names real files and real functions, and is only meaningful in context. Conflating them produces documents that are too technical for review and too abstract for implementation.

**Why does `/biplane-spec` never open source files?**

Specs grounded in implementation drift toward describing the current code rather than the desired behavior. Keeping the spec implementation-free means it stays valid even after a refactor, and it forces the *plan* phase to make the grounding explicit and reviewable.

**Why plan mode approval before execution?**

Because a wrong plan produces an unreviewable diff. Catching seam mismatches, scope creep, or missing reuse opportunities before a single file is touched is cheaper by an order of magnitude.

---

## Credits

`/biplane-build`'s four coding principles are adapted from [multica-ai/andrej-karpathy-skills](https://github.com/multica-ai/andrej-karpathy-skills) by [@jiayuan_jy](https://x.com/jiayuan_jy), itself derived from [Andrej Karpathy's observations](https://x.com/karpathy/status/2015883857489522876) on LLM coding pitfalls.

---

## License

MIT
