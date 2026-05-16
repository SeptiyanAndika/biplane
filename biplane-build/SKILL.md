---
name: biplane-build
description: Read a spec'd Linear issue, produce a concrete implementation plan grounded in the codebase, get human approval via plan mode, then implement. Posts an implementation summary to Linear after coding (functions changed + how to test). Use after /biplane-spec. Replaces the separate /triplane-plan and /triplane-execute workflow.
argument-hint: <ISSUE-ID> [--linear] [--dry-run] [--context "extra context"] [--focus "area to prioritize"]
disable-model-invocation: true
allowed-tools: Bash Read Grep Glob Edit Write Task
---

You are planning **and** implementing a Linear issue that already has a behavioral spec
(produced by `/biplane-spec`). You work in two halves:

- **Half 1 — Plan**: Ground the spec in the real codebase, write a pointer-style
  implementation plan, and get human approval before touching any files.
- **Half 2 — Execute**: Implement the minimum code that makes the acceptance criteria
  pass, verify it, then post a concise summary to Linear.

## Arguments

Parse `$ARGUMENTS`:
- **First token** (required): Linear issue ID, e.g. `AIT-17`
- `--linear` (optional flag): Post the plan as a Linear comment before execution, and
  post the implementation summary after. Without this flag, the plan is shown inline
  via plan mode only and the post-implementation summary is still posted to Linear.
- `--dry-run` (optional flag): Build the plan and print the verification map, then stop.
  Do not edit any files.
- `--context "..."` (optional): Extra background (recent refactors, related PRs, decisions)
- `--focus "..."` (optional): Narrow to a specific area or acceptance criterion

If the issue ID is missing, print:
`Usage: /biplane-build <ISSUE-ID> [--linear] [--dry-run] [--context "..."] [--focus "..."]`
and stop.

---

## Coding Principles (active for the entire skill — read before writing a single line)

Before any implementation begins, internalize and follow these four rules. They are
binding, not advisory. They override any impulse to be "thorough" or "helpful by
adding extras."

**1. Think Before Coding**
State assumptions explicitly. If uncertain, ask. Surface tradeoffs — don't pick
silently. If something is unclear, name what's confusing and ask. Do not paper over
confusion with code.

**2. Simplicity First**
Minimum code that makes the acceptance criteria pass. No features beyond what the spec
requires. No abstractions for single-use code. No error handling for impossible
scenarios. If you write 200 lines and it could be 50, rewrite it. Ask: "Would a senior
engineer say this is overcomplicated?" If yes, simplify before proceeding.

**3. Surgical Changes**
Touch only files in the plan's "Files to Create or Modify" table. Do not improve
adjacent code, comments, or formatting. Match existing style. If you notice unrelated
dead code, mention it at the end — do not fix it. Remove only what your changes made
unused. Every changed line must trace directly to a spec acceptance criterion.

**4. Goal-Driven Execution**
The spec's acceptance criteria are your verification loop. For each criterion, establish
a way to check it passed before moving on. When all criteria are green, stop — do not
add more.

---

# HALF 1 — PLAN

## Phase 1 — Fetch the Spec

1. Fetch the Linear issue via `mcp__linear-server__get_issue` (`id: <ISSUE-ID>`).
2. Read the description. If it does not contain a Functional Specification and
   Technical Design (the LAYER 1 / LAYER 2 structure from `/biplane-spec`), stop:
   *"This issue does not have a behavioral spec yet. Run `/biplane-spec <ID>` first."*
3. If `parentId` exists, fetch the parent for goal context.

---

## Phase 2 — Read Project Conventions

Discover, do not assume paths. Look for project documentation — typically in `docs/`,
but may also live in `documentation/`, `guides/`, README, or architecture/design doc.
Skip silently if none found. Fold in `--context` and `--focus`.

---

## Phase 3 — Explore the Codebase

Launch up to 2 Explore agents in parallel using the Task tool with
`subagent_type: Explore`. Brief them by *kind of thing*, never fixed paths:

**Agent A — Implementation seam:**
- The entry point this behavior attaches to (route handler, command, worker, etc.)
- Input validation layer and existing schemas
- Service/business layer where logic belongs
- Any mock/fixture/placeholder code this work replaces
- Existing tests covering that seam

**Agent B — Conventions & reuse:**
- How a recently-completed sibling feature is structured
- Error-handling and response-envelope pattern
- Config/env handling pattern
- Test framework, file locations, what is typically mocked
- Utilities that overlap with this spec's needs (don't reinvent)

Capture concrete paths and symbols the agents return. These are the substance of the plan.

---

## Phase 4 — Ask Clarifying Questions (only if needed)

Use `AskUserQuestion` (max 2–3) only for real forks the spec left open at the
implementation level, e.g.:
- Extend existing utility X vs. create a new one
- Add to existing module Y vs. introduce a new module
- Refactor adjacent code now vs. defer

Skip if the path is obvious from grounding.

---

## Phase 5 — Write the Plan

Keep it pointer-style. The reader should skim it in 3 minutes and either approve or
push back on the approach.

**## Summary**
2–3 sentences: the chosen seam and why it fits.

**## Files to Create or Modify**
Table: Path / Action (create/modify/delete) / Purpose. One row per file.
Group by layer (entry → service → data → tests).

**## Reuse**
Bulleted list: `name (path) — what it gives us`. At least one entry, or explicit
"nothing reusable — here's why."

**## Integration Seams**
Exact call sites being replaced or extended. For each:
- Current behavior (1 line)
- New behavior (1 line)
- File and approximate location

**## New Dependencies**
Package name + version + why. State "None" if none.

**## Migration / Infra Steps**
Schema changes, env vars, deploy ordering, feature flags. State "None" if none.

**## Test Plan**
Table: Test file / Scenarios covered / New or extended.

**## Gotchas**
Things grounding surfaced that could bite: edge cases, naming collisions, async
ordering, rate limits. ≤6 bullets.

**## Suggested Implementation Order**
Numbered steps, ≤8. Each step leaves the system in a working state.

---

## Phase 6 — Plan Self-Review

Before presenting the plan:

- [ ] Every file listed actually exists or has a clear reason to be new
- [ ] No invented paths — everything traces to grounding findings
- [ ] Reuse section has at least one entry (or explicit explanation)
- [ ] Test plan maps to acceptance criteria in the spec
- [ ] Each implementation step leaves the system buildable
- [ ] Plan is consistent with the spec — no scope creep, no contradictions
- [ ] Total length: 3-minute read
- [ ] Files to Create/Modify: ≤12 rows; Gotchas: ≤6; Implementation Order: ≤8 steps

---

## Phase 7 — Present Plan for Approval

Enter plan mode to present the plan inline for human review.

If `--linear` was passed, also post the plan as a Linear comment:
`mcp__linear-server__save_comment` (`issueId: <ISSUE-ID>`, `body: <full plan>`).
Note in the plan mode presentation that it has also been posted to Linear.

**Do not proceed to Half 2 until the human approves the plan.**

If the plan needs revision, update it in plan mode and re-present. Do not start
implementing under an unreviewed plan.

---

# HALF 2 — EXECUTE

*(Only reached after human approves the plan in Phase 7)*

## Phase 8 — Build the Verification Map

Before touching any code, map each acceptance criterion to a verification method.
Print it to the user.

For each GIVEN/WHEN/THEN in the spec:
- **Unit-testable behavior** → "test in file X passes"
- **Integration boundary** → "integration test in file Y passes"
- **Infra/config** → "command Z returns expected output"

Format:
```
Acceptance Criterion 1: <one-line summary>
  → Verify: <test file or command>
  → Status: [ ] not started

Acceptance Criterion 2: <one-line summary>
  → Verify: <test file or command>
  → Status: [ ] not started
```

If a criterion has no clear verification method, stop and ask before proceeding.

If `--dry-run` was passed, stop here. Print the verification map and exit.

---

## Phase 9 — Confirm Scope

Print:

```
About to execute <ISSUE-ID>:
  Spec: <N> acceptance criteria
  Plan: <K> files ({create} create, {modify} modify)

Files in scope (nothing outside this list will be touched):
  - path/to/file1   (modify)
  - path/to/file2   (create)
  ...

Proceed?
```

Wait for explicit user confirmation before editing files.

---

## Phase 10 — Implement, One Step at a Time

Follow the plan's "Suggested Implementation Order." For each step:

1. State in one line what you are about to do.
2. Write the smallest change that advances the relevant acceptance criterion.
3. Run the verification for that criterion (discover actual commands from the project —
   do not assume `npm test` or any specific runner).
4. Mark the criterion green if it passes.
5. Stop and ask if you need to touch a file not on the plan's list.

Loop until all acceptance criteria are green or a blocker requires the human.

**Anti-patterns — refuse these:**
- "While I'm here" refactors
- Config options "in case we need it later"
- Logging beyond what the spec's Error Handling table requires
- Helper functions used only once
- Catching exceptions just to re-throw them
- Renaming things for stylistic reasons

---

## Phase 11 — Final Verification

Run the project's full test suite (or at minimum tests touching the files you changed).
Walk every acceptance criterion one more time and confirm green.

If anything is red:
- Your code → fix it surgically.
- Pre-existing failure unrelated to your change → note it, do not fix it.

---

## Phase 12 — Execute Self-Review Checklist

- [ ] Every changed line traces to a spec acceptance criterion or a plan step
- [ ] No file outside the plan's list was touched
- [ ] No speculative features, abstractions, or config options added
- [ ] No adjacent code was "improved" without being asked
- [ ] All acceptance criteria have a passing verification
- [ ] No new dependencies beyond what the plan listed
- [ ] Imports/variables/functions made unused by my changes are removed
- [ ] Pre-existing dead code is **not** removed
- [ ] Total diff size is proportional to the ticket

---

## Phase 13 — Report Back

**Do not commit or push.** The human reviews the diff.

Print the report to the terminal:

```
✅ Built <ISSUE-ID>: <issue title>

Function changes:
  path/to/file1
    - functionName: <what changed, e.g. "added precheck before transaction", "changed error code to 422">
    - anotherFn:   <e.g. "added expiresAt parameter", "replaced fixture with DB query">
    - newFn:       <e.g. "new — aggregates voucher balance by type_voucher">

Acceptance criteria (all verified):
  ✓ AC1: <one-line summary> — <test file or command>
  ✓ AC2: <one-line summary> — <test file or command>

Notes:
  - <pre-existing issues spotted but not fixed, migration steps, anything the reviewer should know>

Next: review the diff with `git diff`, then commit when ready.
```

Function changes: one bullet per function touched, grouped by file. Describe the change in
plain language (e.g. "added X parameter", "changed logic to check Y before Z", "new — does Z").
Use `new —` prefix for newly created functions. No signatures, no line numbers. Skip test files.
"Notes" section is optional; omit entirely if empty.

Then **ask the user** whether to post this summary to Linear:
`Post this summary to Linear as a comment on <ISSUE-ID>? (yes/no)`

Only call `mcp__linear-server__save_comment` if the user confirms.

When posting to Linear, wrap the **Function changes** block in a markdown code block
so indentation renders correctly. The rest of the comment is plain markdown.

Linear comment body format:
```
✅ Built <ISSUE-ID>: <issue title>

**Function changes:**
\`\`\`
  path/to/file1
    - functionName: <description>
    - newFn:        new — <description>

  path/to/file2
    - functionName: <description>
\`\`\`

**Acceptance criteria (all verified):**

| Verified | Criterion | Verified by |
|---|-----------|-------------|
| ✓ | <one-line summary> | <test file or command> |
| ✓ | <one-line summary> | <test file or command> |

**Notes:**
  - <notes if any>
```

---

## When to stop and escalate

Stop and ask the human if:
- An acceptance criterion is ambiguous and you cannot pick an interpretation
- The plan calls for a file or function that does not exist in the codebase
- The plan conflicts with what you find in the code (seam moved since planning)
- A test you wrote to verify a criterion fails for reasons the spec did not predict
- You need to touch a file outside the plan's list
- The spec itself appears wrong given a constraint not surfaced earlier

Pushing through any of the above produces unreviewable diffs. Stopping is cheap.
