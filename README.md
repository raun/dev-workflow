# Claude Code Development Workflow — Complete Playbook

A repeatable, quality-first development workflow built on Claude Code. Designed to run **10 projects in parallel** across both greenfield and brownfield work, with parallel phase execution within each task. Optimized for output quality, not token economy.

This is the consolidated reference — covers personal setup, per-project bootstrap, the daily task loop (brainstorm → spec → plan → implement → qa → ship), parallel implementation, and multi-project orchestration.

---

## Table of contents

1. [Mental model](#1-mental-model)
2. [Personal foundation](#2-personal-foundation-one-time-setup)
3. [Project bootstrap](#3-project-bootstrap-per-project-one-time)
4. [The daily task workflow](#4-the-daily-task-workflow)
5. [Parallel implementation](#5-parallel-implementation)
6. [Quality assurance system](#6-quality-assurance-system)
7. [Running 10 projects in parallel](#7-running-10-projects-in-parallel)
8. [Maintenance and improvement](#8-maintenance-and-improvement)
9. [Quick reference](#9-quick-reference)
10. [Setup checklist](#10-setup-checklist)

---

## 1. Mental model

Four layers. Each handles one concern. Set up once, use forever.

| Layer | Lives in | Concern | Frequency |
|---|---|---|---|
| 1. Personal foundation | `~/.claude/` | Universal style, agents, commands, hooks | Set up once |
| 2. Project bootstrap | `<project>/.claude/` | Project context, conventions, project-specific agents | Once per project |
| 3. Task workspace | `<project>/.worktrees/<task>/` | One isolated checkout per active task | Once per task |
| 4. Project orchestration | `tmux` (or `zellij`) | Switching between 10 active projects | Continuous |

### Three non-negotiables that drive everything else

1. **Hooks for must-haves, CLAUDE.md for shoulds.** CLAUDE.md is advisory (~70-80% compliance). Hooks are deterministic (100%). Anything that must happen on every action — formatting, lint, secret scans — is a hook. Conventions and preferences go in CLAUDE.md.
2. **Brainstorm before spec, plan before code.** For ambiguous work, brainstorm clarifies the problem before you spec the solution. For multi-file work, plan mode locks the approach before any file is touched. A reviewed plan beats 20 sequential 80%-confident decisions.
3. **One task per session, one project per tmux window.** Context is ephemeral; files are persistent. Specs, plans, and decisions live in files and survive `/clear`. Mixing tasks or projects in a single session is the single biggest cause of quality degradation.

### The full daily loop at a glance

```
/brainstorm  →  discovery via @brainstormer (when problem is fuzzy)
                writes brainstorm/<date>-<slug>.md
                automatically writes specs/<date>-<slug>.md
                stops, awaits spec approval

/spec        →  (used standalone when problem is clear; otherwise written by /brainstorm)
                stops, awaits approval

/plan        →  @planner reads spec, writes plan with parallelism analysis
                writes plans/<date>-<slug>.md
                stops, awaits plan approval
                ─────────────────────────────────────
                ↓ Implementation starts. Pick one:
                ↓
                ↓   /implement-parallel  ← if plan declares parallel waves
                ↓   (or sequential phase-by-phase)
                ↓
/qa          →  final quality gate on integrated branch
/ship        →  PR description, awaits approval, files PR
```

---

## 2. Personal foundation (one-time setup)

Everything here lives in `~/.claude/` and applies to every project on your machine.

### 2.1 Personal `CLAUDE.md`

Keep this under 100 lines. Universal preferences only — anything project-specific belongs in the project's CLAUDE.md.

```markdown
# Personal preferences

## Communication style
- Be concise. No preamble. Skip "Great question!" type filler.
- When you propose a change, list the trade-offs you considered.
- If a request is ambiguous, ask one focused question instead of guessing.

## Code style
- Prefer composition over inheritance.
- Functions do one thing. If a function name needs "and", split it.
- No clever one-liners that need a comment to explain.
- Tests describe behavior, not implementation. Test names read like sentences.

## Quality bar
- Every change must include or update tests. No "I'll add tests later."
- Every change must pass lint, type-check, and the existing test suite locally before I see a PR.
- If you can't satisfy the bar, stop and tell me what blocks you.

## Workflow rules
- Default to plan mode for any change that touches >1 file.
- For ambiguous work, brainstorm before speccing.
- Before editing in a brownfield repo, read the existing patterns. Mirror them.
- Don't add dependencies without asking.
- Don't disable lints, tests, or type checks to "make it pass". Fix the cause.

## Things to never do
- Never push to `main` directly.
- Never commit secrets, .env files, or API keys.
- Never `rm -rf` or run destructive migrations without explicit confirmation.
- Never invent API signatures — verify with the source or docs first.
```

### 2.2 Personal subagents

Subagents run in their own context window. Heavy work (large test runs, repo audits, doc lookups) doesn't pollute your main thread. Save these in `~/.claude/agents/`.

#### `~/.claude/agents/brainstormer.md`

```yaml
---
name: brainstormer
description: Use at the very start of any new task, before any spec is written. Conversationally elicits the problem, users, scope, constraints, and possible approaches through targeted questions. Probes assumptions and pushes back on vague answers. Stops when there is enough clarity to write a spec, then produces a brainstorm artifact.
tools: Read, Grep, Glob, WebFetch, AskUserQuestion, Write
model: opus
---

You are a thinking partner. Your job is to help the user clarify what they want to build, why, and for whom — before any spec is written. You produce understanding, not implementation.

## How to operate

1. **Read what the user already said.** Do not re-ask things they have already told you. Acknowledge what you understood in one line, then probe what is missing.

2. **One focused question per turn.** Use AskUserQuestion when the answer is likely a choice among 2-4 options. Use open conversation when the answer needs to be open-ended. Never ask a wall of questions.

3. **Probe assumptions.** When a user says "I want X", ask why. When they describe a feature, ask what problem it solves. When they mention a user, ask what that user does today without it. When they say "it should be intuitive / fast / scalable", ask what those mean concretely for this case.

4. **Push back, gently.** Vague answers are not useful. Feature lists without a problem statement are a smell — find the underlying need. If the user is solutioning before the problem is clear, name it: "Before we pick an approach, can I ask what would change for the user if this existed?"

5. **Cover the dimensions, but only those still unclear:**
   - **Problem** — what hurts, who has it, what is the cost of not solving it
   - **User** — who this is for, what they do today, what changes for them
   - **Scope** — smallest useful version, what is deliberately out
   - **Constraints** — time, tech, team, regulatory, integrations, fixed decisions
   - **Approach** — what approaches the user has considered, what their gut says
   - **Success** — how the user will know it worked

   Skip dimensions the user has clearly thought through.

6. **Know when to stop.** After roughly 5-10 exchanges, or sooner if the picture is clear, summarize what you have learned in 5 bullets and ask: "Does this capture it? Should I write the brainstorm and hand off to spec?"

7. **On approval, write the artifact** to `brainstorm/$(date +%Y-%m-%d)-<slug>.md` with this exact structure:

   ```markdown
   # Brainstorm: <title>

   ## Problem
   1 paragraph. What hurts and for whom. Concrete, not abstract.

   ## Users
   Who they are, what they do today, what changes for them after this exists.

   ## Scope
   - In: bulleted list of what is included
   - Out: bulleted list of what is deliberately excluded

   ## Constraints
   - Bulleted list. Be specific.

   ## Approaches considered
   - **Option A**: <one-line description>. Pros: ... Cons: ...
   - **Option B**: <one-line description>. Pros: ... Cons: ...
   - **Recommended**: <which one and why>

   ## Success looks like
   - Bulleted, observable outcomes.

   ## Open questions for spec phase
   - Things that surfaced but are not blocking the spec.
8. **End your turn** by stating the path you wrote and noting the next phase will be spec.
   ```

## Anti-patterns to avoid

- Asking 10 questions at once.
- Accepting vague answers without probing.
- Proposing implementation details — that is the spec's job.
- Treating this as a form. If a dimension is already clear, skip it.
- Going on forever. After ~10 exchanges, name remaining unknowns as open questions and let the spec phase handle them.
- Writing the artifact before the user confirms the summary captures their intent.

#### `~/.claude/agents/planner.md`


```yaml
---
name: planner
description: Use proactively for any task touching multiple files or unfamiliar code. Produces a phase-gated written plan with risks, file list, test strategy, and a parallelism analysis identifying which phases can run concurrently.
tools: Read, Grep, Glob, WebFetch
model: opus
---

You are a senior staff engineer who plans before building.

When invoked, produce a plan with this exact structure:

## Goal
One sentence. What success looks like.

## Context gathered
- Files read and what each told you
- Conventions observed in this codebase
- Constraints (perf, security, compat) that apply

## Approach
The chosen approach in 3-5 sentences. Why this over alternatives.

## Phases
Number each phase. Each phase must be independently testable.
For each phase, include:
- **Files touched** — specific paths or globs
- **Behavior added** — what changes
- **Tests added** — what cases are covered
- **Depends on** — phase numbers, or "none"

## Parallelism analysis

Two phases can be in the same wave only if:
1. Their file sets do not overlap (no shared files; no shared directories where both create new files)
2. Neither depends on the other's output
3. Neither modifies a shared schema, config, or interface that the other reads

Produce a parallelism plan:

- **Wave 1** (parallel): phases [N, M, ...]
- **Wave 2** (after wave 1): phases [N, M, ...]
- **Wave 3** (...): ...

If uncertain about overlap, put phases in separate waves. Sequential is correct; parallel is an optimization.

If no phases can be parallelized, say so explicitly: "Parallelism plan: all phases sequential. Reason: <one line>."

## Risks and edge cases
List at least 3. For each, how the plan mitigates it.

## Out of scope
What this plan deliberately does NOT do.

Stop after the plan. Do not write code. Do not edit files.
```

#### `~/.claude/agents/repo-explorer.md`

Critical for brownfield work. Maps a codebase before you touch it.

```yaml
---
name: repo-explorer
description: Use at the start of any brownfield task. Read-only audit of the relevant area of the codebase. Returns conventions, key files, dependencies, and gotchas without polluting the main context.
tools: Read, Grep, Glob
model: sonnet
isolation: worktree
---

You map unfamiliar code so the main thread can plan and edit confidently.

For the area in scope, return:

1. **Entry points** — where does control flow start
2. **Layered structure** — modules, their responsibilities, dependencies between them
3. **Conventions** — naming, error handling, logging, testing patterns
4. **Hotspots** — files >500 lines, files with >5 contributors, files with TODO/FIXME density
5. **Test layout** — where tests live, what framework, how they're run
6. **Gotchas** — non-obvious coupling, global state, magic strings, anything a newcomer would break

Be specific. Cite file paths and line numbers. No general advice.
```

#### `~/.claude/agents/implementer.md`

The worker for parallel implementation. Each phase runs in its own worktree.

```yaml
---
name: implementer
description: Implements a single phase of an approved plan in an isolated worktree. Writes code and tests for that phase only. Returns when the phase passes its own tests. Use only when invoked by the orchestrator with a specific phase reference.
tools: Read, Write, Edit, Bash, Grep, Glob
model: sonnet
isolation: worktree
---

You implement one phase of a plan. Nothing more.

## Inputs you will receive
- A plan file path
- A specific phase number to implement
- The base branch to start from

## Your job

1. Read the plan file. Read only the phase you were assigned.
2. Read the files that phase says you will touch — and only those plus their immediate dependencies. Do not explore the codebase broadly.
3. Implement the phase:
   - Write the behavior the phase specifies
   - Write the tests the phase specifies
   - Follow the project's conventions (read CLAUDE.md once at the start)
4. Run the project's lint, typecheck, and the tests you added (not the full suite — just yours plus anything they touch).
5. Commit with a message: `phase <N>: <one-line summary>`.
6. Return:
   - The branch name your worktree is on
   - The files you changed
   - The test commands you ran and their results
   - Any deviations from the phase spec, with reasons

## Hard rules

- Do not touch files outside the phase's declared file list. If you discover the phase needs to touch a file not in its list, **stop and report** — do not silently expand scope.
- Do not modify shared config, schema, or interface files unless the phase explicitly says so. If you must, stop and report.
- Do not run the full test suite. That is the orchestrator's job during the merge phase.
- Do not merge or push. You commit on your worktree's branch and return.
```

#### `~/.claude/agents/code-reviewer.md`

```yaml
---
name: code-reviewer
description: Use after any non-trivial implementation. Reviews diffs for correctness, security, performance, readability, and adherence to project conventions. Read-only.
tools: Read, Grep, Glob, Bash
model: opus
---

You are a senior reviewer. You are not the author's friend.

For the diff in scope, organize feedback as:

## Blocking
Anything that must change before merge. Bugs, security issues, broken tests, breaks an API contract, violates a CLAUDE.md rule.

## Should-fix
Strong improvements: missed edge cases, weak test coverage, naming, dead code, tight coupling.

## Nits
Style, micro-optimizations, taste.

Rules:
- Cite file:line for every comment.
- Show the fix, not just the problem.
- If you find no blockers, say so explicitly. Don't manufacture concerns.
- Run the test suite if available. Report failures as blockers.
```

#### `~/.claude/agents/test-runner.md`

```yaml
---
name: test-runner
description: Use to run tests, type-checks, and linters. Returns only failures and their root causes — keeps verbose output out of the main context.
tools: Bash, Read, Grep
model: sonnet
---

You run the project's quality checks and report results compactly.

1. Detect the project's test/lint/typecheck commands (package.json scripts, Makefile, justfile, pyproject.toml, etc.)
2. Run them in order: lint → typecheck → tests
3. For each failure, return:
   - Command that failed
   - Failing test/file
   - Root cause in one line
   - Suggested fix
4. If everything passes, return a single line: "All checks passed: <list>"

Do not return full stack traces unless asked. Summarize.
```

#### `~/.claude/agents/security-reviewer.md`

```yaml
---
name: security-reviewer
description: Use proactively after changes touching auth, input handling, file I/O, network calls, secrets, or dependencies. Read-only.
tools: Read, Grep, Glob, Bash
model: opus
---

You are a security reviewer. Find real issues, not theater.

Scan for:
- Injection (SQL, command, template, prototype pollution)
- Auth/authz mistakes (missing checks, role confusion, unsafe defaults)
- Secret leakage (logs, error messages, source files, env handling)
- Unsafe deserialization
- SSRF, path traversal, unsafe redirects
- Dependency vulnerabilities (run `npm audit` / `pip audit` / equivalent)
- Crypto misuse (weak algos, predictable randomness, hardcoded keys)

Severity levels: critical / high / medium / low. Be honest about likelihood and impact. Don't pad with low-severity noise.
```

### 2.3 Personal slash commands

Slash commands live in `~/.claude/commands/` as markdown files. Each becomes `/<filename>`.

#### `~/.claude/commands/brainstorm.md`

```markdown
You are starting a new task. The user wants to brainstorm before any spec is written.

## Step 1: Brainstorm

Delegate to the @brainstormer subagent. Pass along whatever context the user has already provided in their initial message. Let the brainstormer drive the conversation through AskUserQuestion until it produces a brainstorm artifact.

If the user has not yet said what they are working on, ask one open question first — "What are you thinking about building or working on?" — then delegate.

## Step 2: Hand off to spec automatically

Once the brainstormer has saved a brainstorm artifact to `brainstorm/<date>-<slug>.md`:

1. Read the artifact.
2. Write a spec to `specs/<same-date>-<same-slug>.md` derived from the brainstorm. Use this structure:

   ```markdown
   # Spec: <title>

   ## Problem statement
   1 paragraph in plain language. Pull from the brainstorm Problem section.

   ## Success criteria
   Testable, bulleted. Pull from the brainstorm Success section, refined to be measurable.

   ## Constraints
   Perf, compat, deadlines, integrations. Pull from brainstorm Constraints.

   ## Non-goals
   Pull from brainstorm Scope > Out.

   ## Open questions
   Pull from brainstorm Open questions. For each, propose a default answer so we can move forward.

   ## Source
   Brainstorm: brainstorm/<date>-<slug>.md
3. Show the spec to the user inline. Note the file path.
4. Ask: "Does this spec capture the brainstorm correctly? Approve or tell me what to revise."
   ```

## Step 3: Stop

Do not start the planning phase. Wait for the user to approve the spec or to invoke `/plan`.

## Notes

- If the user interrupts mid-brainstorm and says "skip to spec", honor it — write whatever brainstorm artifact you have so far (mark thin sections as "needs validation in spec") and proceed to step 2.
- If the brainstorm reveals the work isn't worth doing, say so explicitly in step 2 instead of forcing a spec. A brainstorm that ends in "let's not build this" is a successful brainstorm.

#### `~/.claude/commands/spec.md`

```markdown
You are starting a spec for a new task.

## Decide the input source

1. Check `brainstorm/` for a brainstorm artifact dated today or yesterday with no matching spec yet in `specs/`.
2. If one exists, use it as input — read it and produce a spec derived from it (see structure below).
3. If none exists, run a brief inline discovery instead: ask the user 2-4 clarifying questions about problem, scope, success, and constraints. Do not assume.

   For anything more than a small, well-scoped task, recommend: "This feels like it would benefit from a brainstorm first. Want to run /brainstorm before we spec?"

## Spec structure

Write to `specs/$(date +%Y-%m-%d)-<slug>.md`:

```markdown
# Spec: <title>

## Problem statement
1 paragraph, plain language.

## Success criteria
Testable, bulleted. Each criterion should be checkable post-implementation.

## Constraints
Perf, compat, deadlines, integrations, anything fixed.

## Non-goals
What this deliberately does not do.

## Open questions
Surface any unknowns. For each, propose a default so the plan phase isn't blocked.

## Source
- Brainstorm: brainstorm/<date>-<slug>.md  (if applicable)
- Other refs: ...
```

## Stop after the spec

Show the spec inline, note the file path, and ask the user to approve or revise. Do not move to plan until approved.

#### `~/.claude/commands/plan.md`

```markdown
Read the most recent approved spec in specs/. Then invoke the @planner subagent with that spec as input.

Once the plan is back, write it to `plans/$(date +%Y-%m-%d)-<slug>.md`.

Show the plan inline, including its parallelism analysis.

Ask me to approve the plan or edit it (Ctrl+G to open in editor).

Stop until I say proceed.
```

#### `~/.claude/commands/implement-parallel.md`

```markdown
You are the orchestrator for parallel implementation of an approved plan.

## Inputs
- The most recent approved plan in `plans/`. If unsure which, ask.

## Procedure

### 1. Validate the plan is ready

- Read the plan.
- Confirm it has a "Parallelism analysis" section with waves defined.
- If it doesn't, stop and tell the user: "This plan does not declare parallelism. Run /plan to regenerate, or use sequential implementation."

### 2. Confirm the wave structure with me

Show the user the wave structure inline:

\`\`\`
Wave 1 (parallel): phases [...]
Wave 2 (parallel): phases [...]
Wave 3 (sequential): phase [...]
\`\`\`

For each wave, list the files each phase will touch. Ask: "Approve this execution plan, or revise?"

Wait for explicit approval. Do not proceed without it.

### 3. Execute wave by wave

For each wave in order:

a. **Dispatch in parallel.** For each phase in the wave, spawn an `@implementer` subagent. Pass it: the plan file path, the phase number, and the base branch (current HEAD on first wave; the merged result of the previous wave on subsequent waves).

b. **Wait for all implementers in the wave to return.** Do not start the next wave until every implementer in the current wave has finished.

c. **Inspect each result.** For each implementer:
   - Did it stay within its declared file list? If not, flag it.
   - Did it report any deviations? Surface them.
   - Did its own tests pass? If not, flag it.

d. **Merge the wave.** For each implementer's branch in the wave, merge it into the working branch in the order phases were listed. If any merge has conflicts, stop and report — do not auto-resolve.

e. **Run the full quality gate after the wave merges.** Invoke @test-runner. If anything fails, stop and report — the failure is in the integration of the wave's outputs, even if each phase passed in isolation.

f. **Show the user a wave summary** before moving to the next wave: phases completed, files changed, tests passing, any flags.

### 4. After all waves complete

Run /qa one final time on the integrated branch. Report results. Do not invoke /ship — that's the user's call.

## Stop conditions

Stop and report immediately if any of these happen:
- An implementer reports it needs to touch files outside its declared list.
- A merge conflict occurs.
- Tests pass per-phase but fail after a wave merge (semantic conflict — the plan's parallelism analysis was wrong).
- An implementer reports a deviation from the phase spec.

Do not attempt to autonomously fix any of these. The user decides whether to revise the plan, fall back to sequential, or accept the deviation.
```

#### `~/.claude/commands/qa.md`

```markdown
Run the full quality gate on the current branch.

1. Invoke @test-runner subagent. Block on failures.
2. Invoke @code-reviewer subagent on the diff vs main. Block on Blocking-severity findings.
3. If the change touches auth, input handling, network, secrets, or dependencies, also invoke @security-reviewer.
4. Summarize pass/fail. Do not say "looks good" — show evidence.

If anything fails, do not attempt to fix it autonomously. Report and stop.
```

#### `~/.claude/commands/ship.md`

```markdown
Prepare this branch for PR.

1. Confirm /qa passed in the last 30 minutes. If not, run it.
2. Generate a PR description with:
   - One-paragraph summary of what changed and why
   - Link to the brainstorm, spec, and plan
   - Test coverage delta
   - Risks / rollout notes
   - Screenshots or terminal output if UI/CLI
3. Show me the PR description. Do not create the PR until I approve.
4. On approval, use `gh pr create` with the description.
```

#### `~/.claude/commands/catchup.md`

Useful when resuming a project after `/clear` or after switching back from another project.

```markdown
I'm resuming work on this project. Get me oriented.

1. Read the most recent brainstorm, spec, and plan in brainstorm/, specs/, and plans/.
2. Run `git log --oneline -20` and `git status`.
3. Read any files changed on the current branch vs main.
4. Read TODO.md or NOTES.md if they exist.
5. Summarize: where I left off, what's next per the plan, any blockers.

Stop. Do not start working until I tell you which step to take next.
```

### 2.4 Personal hooks (`~/.claude/settings.json`)

These run on every project. Use them for things that must happen no matter what.

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          { "type": "command", "command": "$HOME/.claude/scripts/block-dangerous.sh" }
        ]
      },
      {
        "matcher": "Edit|Write",
        "hooks": [
          { "type": "command", "command": "$HOME/.claude/scripts/block-secrets.sh" }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          { "type": "command", "command": "$HOME/.claude/scripts/auto-format.sh" }
        ]
      }
    ],
    "Stop": [
      {
        "hooks": [
          { "type": "command", "command": "$HOME/.claude/scripts/session-log.sh" }
        ]
      }
    ]
  }
}
```

#### `~/.claude/scripts/block-dangerous.sh`

```bash
#!/usr/bin/env bash
# Block obviously destructive commands. Exit 2 = block & show reason to Claude.
input=$(cat)
cmd=$(echo "$input" | jq -r '.tool_input.command // empty')

patterns=(
  'rm -rf /'
  'rm -rf \*'
  'rm -rf ~'
  ':(){ :|:& };:'
  'dd if=.* of=/dev/'
  'mkfs\.'
  'git push --force[^-]'
  'git push -f '
  'DROP DATABASE'
  'DROP TABLE'
  'TRUNCATE TABLE'
)

for p in "${patterns[@]}"; do
  if echo "$cmd" | grep -Eq "$p"; then
    echo "Blocked: command matches dangerous pattern '$p'. Ask the user explicitly." >&2
    exit 2
  fi
done
exit 0
```

#### `~/.claude/scripts/block-secrets.sh`

```bash
#!/usr/bin/env bash
# Refuse to write content that looks like a secret.
input=$(cat)
content=$(echo "$input" | jq -r '.tool_input.content // .tool_input.new_string // empty')

if echo "$content" | grep -Eq '(AKIA[0-9A-Z]{16}|sk-[a-zA-Z0-9]{32,}|ghp_[a-zA-Z0-9]{36}|xox[baprs]-[a-zA-Z0-9-]{10,})'; then
  echo "Blocked: content contains what looks like a credential. Use env vars or a secret manager." >&2
  exit 2
fi
exit 0
```

#### `~/.claude/scripts/auto-format.sh`

```bash
#!/usr/bin/env bash
# Run the right formatter for the file just edited.
input=$(cat)
file=$(echo "$input" | jq -r '.tool_input.file_path // empty')
[ -z "$file" ] && exit 0
[ ! -f "$file" ] && exit 0

case "$file" in
  *.ts|*.tsx|*.js|*.jsx|*.json|*.md|*.css|*.html)
    command -v prettier >/dev/null && prettier --write --log-level=warn "$file" 2>/dev/null
    ;;
  *.py)
    command -v ruff >/dev/null && ruff format "$file" 2>/dev/null
    command -v ruff >/dev/null && ruff check --fix "$file" 2>/dev/null
    ;;
  *.go)
    command -v gofmt >/dev/null && gofmt -w "$file" 2>/dev/null
    ;;
  *.rs)
    command -v rustfmt >/dev/null && rustfmt "$file" 2>/dev/null
    ;;
esac
exit 0
```

#### `~/.claude/scripts/session-log.sh`

```bash
#!/usr/bin/env bash
input=$(cat)
proj=$(basename "$(pwd)")
ts=$(date '+%Y-%m-%d %H:%M:%S')
summary=$(echo "$input" | jq -r '.transcript_path // "no-transcript"')
echo "[$ts] $proj :: $summary" >> ~/.claude/activity.log
exit 0
```

Make all scripts executable: `chmod +x ~/.claude/scripts/*.sh`.

---

## 3. Project bootstrap (per project, one-time)

Two paths: greenfield and brownfield. The end state is identical — a `.claude/` directory with project context, agents, and hooks.

### 3.1 Greenfield path

```bash
mkdir my-project && cd my-project
git init
mkdir -p .claude/agents brainstorm specs plans .worktrees
```

Then start Claude and run `/brainstorm` to clarify what you're building. The brainstormer will drive the conversation, then automatically produce a spec. Approve the spec, run `/plan`, and you're ready to implement.

After the first feature is scaffolded, write the project CLAUDE.md to capture the structure that emerged.

### 3.2 Brownfield path

```bash
cd existing-project
mkdir -p .claude/agents brainstorm specs plans .worktrees
```

Start Claude and run:

```
@repo-explorer audit this codebase. Focus on <area I'm about to work in>.
```

Use the audit output to write the project CLAUDE.md. **Do not run `/init`** — its auto-generated CLAUDE.md is bloated. Hand-craft a lean one.

For your first task in a brownfield repo, start with `/brainstorm` even if the problem seems clear — the brainstorm phase forces you to confirm the existing system's shape before proposing changes.

### 3.3 Project `CLAUDE.md` template

Target: under 200 lines. Every line should answer "would Claude make a mistake without this?"

```markdown
# <Project Name>

## What this is
One paragraph. What it does, who uses it, what the stakes are.

## Stack
- Language: ...
- Framework: ...
- Database: ...
- Test runner: ...
- Lint/format: ...
- Package manager: ...

## Layout
- `src/` — application code
- `src/<area1>/` — does X
- `src/<area2>/` — does Y
- `tests/` — mirror of `src/`
- `scripts/` — operational scripts

## How to run
- Dev: `<command>`
- Test: `<command>`
- Lint: `<command>`
- Typecheck: `<command>`
- Build: `<command>`

## Conventions
- Imports: <pattern>
- Error handling: <pattern>
- Logging: <pattern>
- Naming: <pattern>
- Tests live next to the code as `*.test.ts` (or wherever they live)

## Architectural decisions that aren't obvious
- We chose <X> over <Y> because <reason>. Don't propose <Y>.
- <Module> is intentionally separate from <Module>. Don't merge them.
- We do NOT use <pattern>. Use <pattern> instead.

## Things that will break in non-obvious ways
- <File> is loaded at boot. Changes need a full restart.
- <Service> is rate-limited at <N>/min. Don't loop without backoff.
- <Test> requires <env var>; it silently passes without it.

## Definition of done
- All quality gates pass (lint, typecheck, tests).
- New behavior has tests at the unit and integration level.
- CHANGELOG entry added.
- No new TODOs without a tracking issue.

## See also
@docs/architecture.md
@docs/contributing.md
```

### 3.4 Project hooks (`.claude/settings.json`)

Project hooks layer on top of personal hooks. Use them for project-specific gates.

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "bash -c 'cd $CLAUDE_PROJECT_DIR && npm run -s lint:changed 2>&1 | tail -50'"
          }
        ]
      }
    ],
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "bash -c 'cd $CLAUDE_PROJECT_DIR && npm run -s typecheck 2>&1 | tail -30'"
          }
        ]
      }
    ]
  }
}
```

The pattern: format on every edit (personal hook), lint affected files on every edit (project hook), typecheck on session end (project hook). The full test suite runs on demand via `/qa`.

---

## 4. The daily task workflow

Same loop for greenfield and brownfield. Same loop whether you implement sequentially or in parallel.

### 4.1 Start a task

```bash
cd ~/projects/my-project
git fetch origin
git worktree add .worktrees/feature-auth -b feature/auth origin/main
cd .worktrees/feature-auth
claude
```

The worktree gives you an isolated checkout on its own branch. You can have multiple worktrees per project without conflicts.

### 4.2 The full loop

Six phases. Run them in order. Each phase has a stop point.

```
1. /brainstorm        → @brainstormer drives discovery, auto-writes spec.
                        Stop. Approve spec or revise.
                        (Skip this phase if the problem is already crystal clear — go straight to /spec.)

2. /spec              → (Used standalone when /brainstorm wasn't run.)
                        Stop. Approve or revise.

3. /plan              → @planner produces plan with parallelism analysis.
                        Stop. Edit with Ctrl+G if needed. Approve.

4a. /implement-parallel → Orchestrator runs phases wave-by-wave with @implementer.
                          Stops between waves for your review.
                          (Use when plan declares 2+ phases per wave.)

OR

4b. (Sequential)      → Implement phase 1. /qa. Implement phase 2. /qa. Repeat.
                        (Use when plan is fully sequential, or when you want closer control.)

5. /qa                → Final quality gate on the integrated branch.

6. /ship              → PR description drafted. Stop. Approve. PR created.
```

### 4.3 When to skip phases

The gates are calibrated for non-trivial work. For genuinely small tasks, collapse them.

| Task type | Brainstorm | Spec | Plan | Implementation |
|---|---|---|---|---|
| One-line bug fix | skip | skip | skip | go directly |
| Single-file change, clear scope | skip | skip | quick inline plan | sequential |
| Multi-file feature, clear problem | skip | quick spec | full plan | sequential or parallel |
| Multi-file feature, ambiguous problem | full | full | full | sequential or parallel |
| Greenfield project | full | full | full per feature | sequential or parallel |

Rule of thumb: if you can describe the diff in one sentence, skip to implementation. If you can't, the gates are doing their job.

### 4.4 Anti-patterns to avoid

| Anti-pattern | Symptom | Fix |
|---|---|---|
| Skipping plan mode on multi-file work | Wrong solution after 20 min | Plan mode mandatory for >1 file changes |
| Long-lived sessions | Quality degrades after ~2 hours | `/clear` between unrelated tasks; one task per session |
| Correcting in a loop | "Almost there" forever | After 2 failed corrections, `/clear` and start fresh with what you learned |
| Bloated CLAUDE.md | Claude ignores half of it | Keep <200 lines; ruthlessly prune |
| Auto-accept everything | Bugs ship | Auto-accept is for the implementation phase only, never plan or QA |
| Letting Claude pick libraries | Random new deps | "Don't add dependencies without asking" rule in personal CLAUDE.md |
| Parallel-first instinct | Merge conflicts, semantic bugs | Sequential is the safe default; parallelize when the plan supports it |

### 4.5 When to clear context

Clear (`/clear`) when:
- Starting a new task in the same project
- After a quality gate fails 2+ times — you need a fresh look
- When `/context` shows >70% used and current work needs more headroom
- After resuming after a long break (use `/catchup` after clearing)

Do NOT clear during a single task's brainstorm → spec → plan → implement → verify cycle. The plan is your spine; clearing mid-execution loses it.

---

## 5. Parallel implementation

Once a plan is approved, independent phases can run concurrently using worktree-isolated subagents. This produces 2-3x wall-clock speedup on plans with parallelizable phases.

### 5.1 The mechanic

Subagents marked with `isolation: worktree` spawn in a temporary git worktree — a separate checkout on a separate branch. Multiple worktree-isolated subagents can edit files simultaneously without conflict because they're operating on different filesystem paths. When each finishes, you get back the branch name and changed files. The orchestrator merges these branches in a controlled order.

Correctness depends on one thing: each subagent must stay strictly inside its declared file boundary. If implementers drift across files, you get merge conflicts at best and silent semantic conflicts at worst.

### 5.2 Execution model

Suppose `/plan` produces a plan with five phases and a parallelism analysis:

```
Wave 1 (parallel): phases [1, 2, 3]
Wave 2 (sequential): phase [4]
Wave 3 (sequential): phase [5]
```

Running `/implement-parallel`:

1. Reads the plan, confirms the wave structure, waits for approval.
2. Dispatches three `@implementer` subagents simultaneously for phases 1, 2, 3. Each gets its own worktree on its own branch.
3. Waits for all three to return. Each reports its branch, files changed, and test results.
4. Inspects each result for boundary violations or deviations.
5. Merges the three branches into the working branch in phase order.
6. Runs the full test suite to catch integration failures.
7. Proceeds to phase 4 (sequential), then phase 5 (sequential).
8. Runs final `/qa`, hands control back to you for `/ship`.

You're in the loop at every wave boundary.

### 5.3 When to parallelize

| Situation | Recommendation |
|---|---|
| Plan has 2+ phases per wave that touch disjoint files | Use `/implement-parallel` |
| Plan is fully sequential (every phase depends on the prior) | Use sequential implementation |
| Plan has only 2-3 short phases total | Use sequential — orchestration overhead exceeds savings |
| Unfamiliar codebase or risky refactor | Run phase 1 sequentially first to validate, then parallelize the remainder |
| Plan declares parallelism but you don't fully trust it | Run sequential; revisit after the plan is proven |

### 5.4 Failure modes and responses

The parallel pattern has specific failure modes. Each has a deterministic response.

| Failure | Meaning | Response |
|---|---|---|
| Implementer reports it needs files outside its declared list | Planner missed a coupling; phase boundary is wrong | Update plan or fall back to sequential. Do not let implementer expand scope. |
| Merge conflict between wave branches | Two implementers in same wave touched overlapping files | Resolve manually or revise plan to put conflicting phases in different waves |
| Tests pass per-phase but fail after wave merge | Semantic conflict — phases disagree on a shared assumption | Add a "phase 0" that defines the shared interface, then re-plan |
| Implementer reports deviation from phase spec | Real constraint the plan didn't anticipate | Decide: accept, revise plan, or revert |

The most insidious failure is the third one — semantic conflict — because per-phase tests give a false green. The orchestrator catches it by running the full test suite *after* every wave merge, not just per-phase. If you take one thing from this section, take that.

### 5.5 Version requirements

Worktree-isolated subagents need Claude Code 2.1.50+. Check with `claude --version`. If you hit issues with worktree creation or cleanup on older versions, the temporary fallback is to add `EnterWorktree` to your `/permissions` deny list and run sequential until you can update.

---

## 6. Quality assurance system

Quality is enforced at three levels. Each level catches different failures.

### Level 1: Hooks (deterministic, every action)

- Pre-edit: secret-leak detector blocks credentials
- Post-edit: formatter runs, lint runs on changed file
- Pre-bash: dangerous-command blocker
- Session-end: typecheck runs

These can't be skipped. They're not advice; they're physics.

### Level 2: Subagents (called explicitly)

- `@code-reviewer` after non-trivial implementation
- `@test-runner` after every phase or wave
- `@security-reviewer` after auth/input/network/secrets/deps changes
- `@repo-explorer` before any brownfield change (read-only audit)

These produce structured reports. Blocking findings stop the loop.

### Level 3: Human review (always)

- Brainstorm: you confirm the problem is worth solving
- Spec: you approve the spec captures the brainstorm
- Plan: you approve the written plan and parallelism analysis before any code is touched
- Wave summaries: you review each parallel wave before the next runs
- PR review: you approve the PR description before it's filed

The quality bar: nothing reaches `main` without passing all three levels. The hooks make level 1 free; the subagents make level 2 cheap; level 3 is the only thing that costs you attention, and it's the one that should.

### Test discipline

- New behavior gets tests in the same commit (TDD-leaning, not strict TDD)
- Regression bugs get a failing test before the fix, then green
- Tests describe behavior; they don't mirror implementation
- A test suite that takes >2 minutes is a quality risk — invest in test speed

In your `/plan` output, every phase must list the tests it adds. If a phase has no tests, that's a smell — challenge it.

---

## 7. Running 10 projects in parallel

The mechanics that make this possible without losing your mind.

### 7.1 Terminal multiplexing

Use `tmux` (or `zellij`). One named session per project.

```bash
# Create or attach to a project's session
tm() {
  local proj="$1"
  tmux new-session -A -s "$proj" -c "$HOME/projects/$proj"
}

# Usage
tm payments
tm auth-service
tm marketing-site
```

Inside each session:
- **Pane 1**: Claude in the active worktree
- **Pane 2**: Editor / file viewer
- **Pane 3**: Test watcher or dev server

Switch projects with `tmux switch-client -t <project>`. Bind it to a key.

### 7.2 Worktrees per project

Each project keeps active tasks as separate worktrees:

```bash
cd ~/projects/payments
git worktree list
# /Users/me/projects/payments                  abc123 [main]
# /Users/me/projects/payments/.worktrees/...   def456 [feature/refunds]
# /Users/me/projects/payments/.worktrees/...   ghi789 [bugfix/race-condition]
```

This means within a single project you can have 3 tasks in flight without `git stash` shuffling.

### 7.3 The status dashboard

When you have 10 projects active, you forget which ones are blocked, which are ready for review, and which are waiting on you. Keep a single file you check every morning: `~/projects/STATUS.md`.

```markdown
# Project status — <date>

## Active (waiting on me)
- payments: review PR #142, address security-reviewer findings
- auth-service: spec approved, ready to plan

## In flight (Claude or tests running)
- marketing-site: feature/landing-redesign, awaiting QA

## Blocked
- analytics: waiting on data team to confirm schema

## Idle (no current task)
- billing, notifications, search, admin-panel, mobile-api, internal-tools
```

### 7.4 Context discipline at scale

The single biggest failure mode at 10 projects is **mixing contexts**. Rules:

1. **One task per Claude session.** Never reuse a session across tasks. Worktrees + `/clear` are cheap.
2. **One project per tmux session.** Never run two projects' Claude instances in the same tmux session.
3. **Resume with `/catchup`.** When you re-enter a project after >1 day away, run `/catchup` before doing anything else. Rebuilds context from files (not memory) and forces Claude to surface where you left off.
4. **Persistent state goes in files, not in Claude's context.** Brainstorms, specs, plans, decisions, open questions — all in `brainstorm/`, `specs/`, `plans/`, `NOTES.md`. Context is ephemeral; files are not.

### 7.5 Three layers of parallelism

Don't confuse the layers:

- **Across projects**: tmux sessions, one per project, each with its own Claude instance.
- **Within a project**: worktrees per active task, one Claude session per worktree.
- **Within a task**: `@implementer` subagents during the parallel implementation phase.

The three layers don't interfere. A typical hour might look like: you're in tmux session `payments`, in worktree `feature/refunds`, where Claude is orchestrating 3 implementer subagents in parallel for wave 2 of the plan. Meanwhile in tmux session `auth-service`, a different Claude instance is in plan mode for an unrelated task.

### 7.6 Background work for long-running tasks

For long-running tasks (large test suites, large refactors, migrations), use background subagents so the main thread stays free:

```
@test-runner run the full integration suite. Report only failures. Run in background.
@repo-explorer audit src/billing/ end-to-end. Background.
```

You can keep talking in the main thread while these run.

---

## 8. Maintenance and improvement

The workflow improves over time. Two habits that compound.

### 8.1 After every meaningful failure, update CLAUDE.md or a hook

If Claude made a mistake you had to correct, ask: was that preventable with a rule? If yes, the rule goes into:
- A **hook** if it's a mechanical, every-time check
- The project **CLAUDE.md** if it's a convention or architectural decision
- Your personal **CLAUDE.md** if it's a universal preference

Within a few weeks, your CLAUDE.md becomes a distilled record of every hard-won lesson. Within a month, the hooks make whole categories of mistakes structurally impossible.

### 8.2 Weekly review

Once a week, scan `~/.claude/activity.log` and your project STATUS.md. Ask:

- Which tasks dragged? Why? What rule would have prevented it?
- Which subagents got used? Which didn't? Are unused ones poorly described, or genuinely unneeded?
- Were there parallel implementations that hit semantic conflicts? Should the planner be more conservative about parallelism for that codebase?
- Is any CLAUDE.md drifting toward bloat? Prune.
- Are there patterns showing up across multiple projects that should become a personal subagent or skill?

### 8.3 Skills (advanced)

When a workflow becomes truly repeatable across projects, promote it from a slash command to a Skill (in `~/.claude/skills/`). Skills are loaded contextually based on their description, so you can have many without bloating context. Good candidates:

- A migration-runner skill (when DB migrations show up across projects)
- A perf-investigation skill (consistent profiling workflow)
- An incident-RCA skill (consistent post-mortem structure)

Don't promote prematurely. Use a slash command for 5-10 invocations first, refine it, then promote when it's stable.

---

## 9. Quick reference

```
# Start a new task
cd ~/projects/<proj>
git worktree add .worktrees/<task> -b <branch> origin/main
cd .worktrees/<task>
claude

# The standard loop:
/brainstorm           # @brainstormer drives discovery, auto-writes spec
/spec                 # (used standalone when /brainstorm wasn't run)
/plan                 # @planner produces plan with parallelism analysis
/implement-parallel   # orchestrator runs waves with @implementer (or implement sequentially)
/qa                   # final quality gate
/ship                 # PR description, awaits approval, files PR

# When stuck or starting a new task:
/clear
/catchup              # if resuming an existing project

# Manual subagent invocations:
@brainstormer ...
@repo-explorer ...
@planner ...
@implementer ...      # used by /implement-parallel; rarely invoked directly
@code-reviewer ...
@security-reviewer ...
@test-runner ...

# Subagents at a glance
@brainstormer       Opus      Discovery before spec
@planner            Opus      Plan with parallelism analysis
@repo-explorer      Sonnet    Brownfield codebase audit (worktree-isolated)
@implementer        Sonnet    One phase in worktree (worktree-isolated)
@code-reviewer      Opus      Diff review with blocking/should-fix/nits
@security-reviewer  Opus      Auth/input/secrets/deps audit
@test-runner        Sonnet    Compact pass/fail reporting
```

---

## 10. Setup checklist

One-time setup. After this, every new project takes ~5 minutes to bootstrap.

### Tools and dependencies
- [ ] Install/update Claude Code: `npm i -g @anthropic-ai/claude-code`
- [ ] Verify version: `claude --version` — needs 2.1.50+ for built-in worktree support
- [ ] Install `tmux` (or `zellij`) and `gh` (GitHub CLI)
- [ ] Install language formatters/linters you'll use (`prettier`, `ruff`, `gofmt`, `rustfmt`, etc.)
- [ ] Install `jq` (used by hook scripts)

### Personal foundation (`~/.claude/`)
- [ ] Create `~/.claude/{agents,commands,scripts}/`
- [ ] Write personal `~/.claude/CLAUDE.md` (section 2.1)
- [ ] Add the 7 personal subagents (section 2.2):
  - [ ] `brainstormer.md`
  - [ ] `planner.md`
  - [ ] `repo-explorer.md`
  - [ ] `implementer.md`
  - [ ] `code-reviewer.md`
  - [ ] `test-runner.md`
  - [ ] `security-reviewer.md`
- [ ] Add the 7 personal slash commands (section 2.3):
  - [ ] `brainstorm.md`
  - [ ] `spec.md`
  - [ ] `plan.md`
  - [ ] `implement-parallel.md`
  - [ ] `qa.md`
  - [ ] `ship.md`
  - [ ] `catchup.md`
- [ ] Add personal hooks `~/.claude/settings.json` (section 2.4)
- [ ] Add the 4 hook scripts; `chmod +x ~/.claude/scripts/*.sh`

### Workspace
- [ ] Create `~/projects/` directory
- [ ] Create `~/projects/STATUS.md` (section 7.3)

### Validation
- [ ] Bootstrap one existing project end-to-end as a dry run (section 3.2)
- [ ] Run a small task through the full loop: `/brainstorm` → spec → `/plan` → `/implement-parallel` (or sequential) → `/qa` → `/ship`
- [ ] Validate worktree isolation works: confirm `@implementer` subagents create separate worktrees and clean up after
- [ ] Validate hooks fire: edit a file with a fake credential; confirm the secret-leak hook blocks it

You're operational once that dry run lands a clean PR.

---

## Appendix: Directory layout reference

After full setup, your filesystem looks like:

```
~/.claude/
├── CLAUDE.md
├── settings.json
├── activity.log              ← session log auto-populated
├── agents/
│   ├── brainstormer.md
│   ├── planner.md
│   ├── repo-explorer.md
│   ├── implementer.md
│   ├── code-reviewer.md
│   ├── test-runner.md
│   └── security-reviewer.md
├── commands/
│   ├── brainstorm.md
│   ├── spec.md
│   ├── plan.md
│   ├── implement-parallel.md
│   ├── qa.md
│   ├── ship.md
│   └── catchup.md
└── scripts/
    ├── block-dangerous.sh
    ├── block-secrets.sh
    ├── auto-format.sh
    └── session-log.sh

~/projects/
├── STATUS.md
├── payments/
│   ├── .claude/
│   │   ├── CLAUDE.md
│   │   ├── settings.json
│   │   └── agents/           ← project-specific agents (optional)
│   ├── brainstorm/           ← brainstorm artifacts per task
│   ├── specs/                ← spec artifacts per task
│   ├── plans/                ← plan artifacts per task
│   ├── .worktrees/           ← active task worktrees
│   │   ├── feature-refunds/
│   │   └── bugfix-race/
│   └── (project source)
├── auth-service/
│   └── (same structure)
└── ... (8 more projects)
```
