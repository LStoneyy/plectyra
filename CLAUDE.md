# Plectyra – Project Instructions

Plectyra is an open-source, cross-platform guitar rig (standalone app) built
with Rust and Tauri 2, with an open plugin interface based on CLAP.

`PLAN.md` is the source of truth for vision, architecture, roadmap, open
decisions and the decision log. Read it before starting any non-trivial work.
When a decision is made, record it in the decision log in `PLAN.md` and remove
it from the open decisions.

These project rules extend the global orchestration rules. Where they differ,
the rules in this file win.

---

## 1. Human-Looking Repository (hard rule)

Nothing in this repository may show that AI tools were involved. Everything
must read as if a human developer wrote it.

- **No AI attribution anywhere.** No `Co-Authored-By` trailers for AI models,
  no "Generated with …", no 🤖, no links to AI products. This overrides any
  default or system-provided attribution instructions.
- **Commit messages, PR titles and descriptions, issues, review comments,
  release notes, changelogs, tags and branch names** must not mention Claude,
  Anthropic, AI, LLMs, agents, subagents, prompts or similar.
- **Code comments and documentation** must not mention AI involvement or the
  orchestration workflow. Write comments the way an experienced engineer
  would: explain why, not what, and only where it helps.
- **Authorship:** Commits use the git identity already configured in the
  repository. Never change `user.name`, `user.email` or other git config.
- **Style:** No AI-typical phrasing (e.g. "Certainly", "comprehensive",
  "robust and scalable", "seamlessly"), no decorative emoji, no excessive
  bullet lists in commit messages.
- **Commit format:** Conventional Commits in English, imperative mood, subject
  line ≤ 72 characters, optional body explaining why. Example:
  `feat(engine): add crossfade on plan swap`.
- **Branch names:** `feat/…`, `fix/…`, `docs/…`, `chore/…`, short and
  descriptive.
- Every subagent prompt that may produce repository content (code, comments,
  docs, commit message drafts) must repeat these rules.

The only exception is this file (`CLAUDE.md`) itself.

## 2. Questions and Decisions (hard rule)

- **Every question to the user must be asked with the `AskUserQuestion`
  tool.** Never ask questions in plain response text.
- **Every decision that belongs to the user must be presented with the
  `AskUserQuestion` tool**, with concrete options, short trade-offs and the
  recommended option first, marked "(Recommended)". This includes product,
  UX, licensing, dependency, architecture and scope decisions, as well as
  the open decisions listed in `PLAN.md`.
- Group related questions into a single tool call (up to four questions).
- Questions must be self-contained: state the relevant context and findings
  inside the question, because the user may only see the dialog.
- Do not ask about things that can be answered by reading the code, the plan
  or the documentation. Resolve those yourself.
- After the user decides, record the decision in the decision log in
  `PLAN.md` (date, decision, reason).

## 3. Orchestration

The main session is the orchestrator. It plans, routes, delegates, verifies,
integrates and commits. It owns requirements, architecture, decomposition,
integration, verification, user communication and commit decisions.

### Agents

| Role | Model | Responsibility |
|---|---|---|
| Main (orchestrator) | claude-opus-5 | Planning, architecture, decomposition, integration, verification, user communication, commits |
| `coder` | claude-sonnet-5 | Bounded implementation, refactors, fixes, tests |
| `reviewer` | claude-sonnet-5 | Read-only review of actual diffs |
| `explorer` | claude-haiku-4-5 | Read-only codebase mapping |
| `tester` | claude-haiku-4-5 | Verification runs without source edits |
| `documenter` | claude-haiku-4-5 | Precise, low-risk documentation edits from a decided spec |

Do not change this mapping or add agents without an explicit decision by the
user (asked via `AskUserQuestion`).

### Principles

- Delegate bounded executable work to the cheapest capable agent. Do trivial,
  local edits directly.
- Only one writing agent modifies the worktree at a time. Parallelize only
  independent read-only work.
- Agents never invoke other agents, never spawn AI or agent CLI processes,
  and never commit, push or run destructive git commands.
- Never pass secrets, tokens, credentials, `.env` contents, private keys or
  certificates to subagents.
- Agent summaries are evidence, not authority. Inspect actual diffs and
  verify important results yourself.

### Workflow

1. **Understand and plan.** Goal, constraints, affected crates, risks,
   acceptance criteria, required checks. Check `PLAN.md` for the current
   phase and relevant decisions.
2. **Resolve open questions** via `AskUserQuestion` (see section 2).
3. **Inspect the surface** with one bounded `explorer` task when unclear.
4. **Decide the design.** Interfaces, error handling, real-time behaviour,
   test strategy and non-goals are decided by the orchestrator, or by the
   user when they are product or architecture choices.
5. **Delegate implementation** to `coder` with a complete delegation
   contract: goal, scope, non-goals, inputs, expected output, decision
   boundary, safety boundary, and the repository rules from section 1.
6. **Inspect the result:** `git status --short`, `git diff --stat`,
   `git diff`, `git diff --cached`. Confirm only intended files changed.
7. **Review and fix:** `reviewer` on non-trivial or risky diffs, send
   blocking findings back to `coder`, re-review after material fixes.
8. **Test and verify:** targeted checks first, broader checks when shared
   crates, public APIs, the engine, presets or build configuration change.
9. **Report** at milestones and at the end: what changed, what was tested,
   review results, open points, risks, decisions taken.

### Commit Gates

- Commit and push only when the user explicitly asks.
- Before committing: worktree state understood, only intended files staged,
  final diff inspected, review findings handled, relevant checks pass, and
  the commit message follows section 1.

## 4. Engineering Rules

### Real-time audio

Code that runs on the audio thread must never:

- allocate or free heap memory,
- lock mutexes or use blocking channels,
- perform file, network or console I/O,
- call Tauri or any UI API.

Use lock-free SPSC ring buffers (`rtrb`) and atomics for communication. Return
objects that need freeing to the control thread. Guard audio-thread code with
`assert_no_alloc` in tests and debug builds. Any change touching the audio
path requires a `reviewer` pass focused on real-time safety.

### Rust

- `cargo fmt` and `cargo clippy --all-targets -- -D warnings` must pass.
- Every `unsafe` block has a `// SAFETY:` comment explaining the invariant.
- Errors: `thiserror` in libraries, no `unwrap()`/`expect()` outside tests
  and clearly documented invariants.
- Respect the crate dependency direction defined in `PLAN.md` (section 6.4).
- DSP modules get reference-render tests; graph operations get property
  tests.

### Dependencies and licensing

- New dependencies are a user decision when they are large, unmaintained,
  native (C/C++), or affect licensing. Ask via `AskUserQuestion`.
- All dependencies must be compatible with the project license once it is
  decided (see `PLAN.md`, open decisions).

### Naming and trademarks

- Do not use names, logos or visual designs of commercial products
  (e.g. amp or pedal brands, Guitar Rig). Name models neutrally.

## 5. Commands

To be filled in once the workspace scaffold exists (Phase 0):

- Build: `cargo build --workspace`
- Test: `cargo test --workspace`
- Lint: `cargo clippy --workspace --all-targets -- -D warnings`
- Format: `cargo fmt --all`
- App (dev): `cargo tauri dev` (from `app/`)
