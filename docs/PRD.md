# PRD: pi-autopi

## 1. Summary

**pi-autopi** is a Pi extension and future CLI supervisor for long-running, task-list-driven coding workflows. A user points autopi at a `TODO.md`; autopi selects one actionable item at a time, prompts the Pi agent to complete it, enforces a structured task closeout, optionally commits/pushes, compresses or resets context, and continues with the next item.

Core principle: **the LLM is the worker for one task; autopi is the deterministic orchestrator, ledger, and context boundary.**

## 2. Problem

Long Pi sessions that attempt to process many tasks in one conversation are fragile:

- context grows until quality degrades;
- task boundaries become blurry;
- intermediate exploration pollutes later work;
- commits and pushes can become mixed or unsafe;
- recovery after interruption is hard;
- manual `/compact` is the opposite of unattended automation.

Users need a safe way to run many small-to-medium tasks from a TODO file without relying on the agent to remember the workflow.

## 3. Goals

- Run through a `TODO.md` task list one item at a time.
- Keep each task scoped, reviewable, and independently committable.
- Automatically trigger context compression or session reset after each task.
- Persist run state outside the LLM context.
- Require structured task completion via an autopi tool.
- Support conservative Git gates for status, commit, and push.
- Stop safely on ambiguity, failed verification, dirty unrelated changes, or repeated failures.

## 4. Non-goals for MVP

- Parallel execution of multiple TODO items in the same worktree.
- Blind automatic pushes by default.
- Full project-management replacement for GitHub Issues/Jira/etc.
- General-purpose autonomous “do anything forever” behavior.
- Perfect parsing of every Markdown TODO style.

## 5. Primary UX

### Pi extension commands

```text
/autopi TODO.md
/autopi TODO.md --max 5
/autopi TODO.md --commit ask --push ask
/autopi status
/autopi stop
```

### Future CLI

```bash
autopi run TODO.md --max 20 --commit auto --push ask --compress new-session
```

## 6. Default safety policy

MVP defaults should be conservative:

```text
maxTasks: 1
commit: ask
push: ask
compress: compact
stopOnFailure: true
stopOnDirtyUnrelatedChanges: true
```

More autonomous operation must be explicit:

```text
/autopi TODO.md --max 20 --commit auto --push ask
```

`--push auto` should require explicit opt-in and a clear warning.

## 7. Supported TODO format

Minimum MVP support:

```md
# TODO

- [ ] Refactor auth validation
- [ ] Add integration tests
- [x] Update README
```

Recommended stable IDs:

```md
- [ ] <!-- id:auth-validation --> Refactor auth validation
- [ ] <!-- id:integration-tests --> Add integration tests
```

If no ID exists, autopi may generate an internal ID from file path + line number + normalized text, but stable HTML comment IDs are preferred.

## 8. Core workflow

```text
LOAD_TODO
  -> SELECT_NEXT_TASK
  -> CHECK_GIT_STATE
  -> START_AGENT_FOR_SINGLE_TASK
  -> WAIT_FOR_STRUCTURED_FINISH
  -> VERIFY_RESULT
  -> UPDATE_TODO_AND_LEDGER
  -> COMMIT_GATE
  -> PUSH_GATE
  -> COMPRESS_OR_RESET_CONTEXT
  -> NEXT_TASK_OR_STOP
```

The agent receives exactly one selected task and must not start unrelated TODO items.

## 9. Agent task prompt contract

For each selected task, autopi sends a prompt like:

```text
You are running under autopi.

Task ID: <id>
Task: <text>
TODO file: <path>

Rules:
- Work only on this task.
- Use subagents for reconnaissance, implementation, or review when useful.
- Do not start other TODO items.
- Run relevant verification.
- At the end, call autopi_finish exactly once with structured result.
- If blocked, call autopi_finish with status="blocked" and explain the blocker.
```

## 10. Required agent completion tool

Autopi registers a tool such as `autopi_finish`.

Expected payload:

```json
{
  "taskId": "auth-validation",
  "status": "done",
  "summary": "Refactored auth validation and added coverage.",
  "changedFiles": ["src/auth.ts", "test/auth.test.ts"],
  "verification": ["npm test"],
  "proposedCommitMessage": "refactor auth validation",
  "semverImpact": "no bump",
  "blockers": [],
  "openQuestions": []
}
```

Allowed statuses:

- `done`
- `blocked`
- `failed`

Autopi must not proceed to commit/push/next task without this structured completion, unless the user manually overrides.

## 11. Context compression requirements

Autopi must never depend on the user manually running `/compact`.

After each task boundary, autopi should trigger one of two modes:

### Mode A: `compact`

Use Pi extension API `ctx.compact(...)` with focused instructions:

```text
Autopi task boundary compaction.

Keep only:
- run state and current TODO file path
- completed task ID and result
- files changed
- verification run and result
- commit SHA if any
- blockers/open questions
- next candidate task

Drop:
- raw exploration
- failed intermediate attempts
- long command output
- subagent chatter
```

### Mode B: `new-session` / future preferred long-run mode

After each task:

1. persist ledger;
2. create a new Pi session;
3. inject a compact handoff containing only durable run state;
4. start the next task from a clean context.

This mode is preferred for truly long unattended runs.

## 12. Git requirements

Before starting a task:

- inspect `git status` when inside a Git repository;
- stop if unrelated dirty changes are present unless configured otherwise;
- record current branch and remote tracking status.

Before commit:

- inspect final `git status`;
- ensure changed files are expected for the task;
- run or confirm verification;
- apply SemVer/changelog decision where relevant.

Commit modes:

- `off`: never commit;
- `ask`: propose commit and wait for approval;
- `auto`: commit if checks pass and no ambiguity exists.

Push modes:

- `off`: never push;
- `ask`: propose exact remote/branch and wait for approval;
- `auto`: push only after explicit configuration.

## 13. State persistence

Autopi stores durable run state outside LLM context, e.g. as session custom entries or extension state.

Minimum ledger fields:

```json
{
  "todoFile": "TODO.md",
  "startedAt": 0,
  "updatedAt": 0,
  "currentTaskId": "auth-validation",
  "completed": [],
  "blocked": [],
  "failed": [],
  "lastCommit": null,
  "options": {
    "maxTasks": 5,
    "commit": "ask",
    "push": "ask",
    "compress": "compact"
  }
}
```

The ledger must be sufficient to resume after interruption without reading the full previous conversation.

## 14. Subagent policy

Autopi should encourage subagents inside a single task, not across unrelated tasks.

Recommended per-task pattern:

```text
scout -> worker -> reviewer
```

Parallel TODO items are out of scope for MVP unless each task has its own branch/worktree and integration path.

## 15. Stop conditions

Autopi must stop and notify the user when:

- TODO file cannot be parsed safely;
- no open tasks remain;
- selected task is ambiguous;
- agent does not call `autopi_finish`;
- verification fails;
- Git status contains unrelated changes;
- merge conflicts occur;
- commit/push requires approval;
- compaction/session reset fails;
- max task count or error budget is reached.

## 16. MVP acceptance criteria

MVP is complete when:

- `/autopi TODO.md --max N` selects open checkbox tasks sequentially;
- the agent receives only one task at a time;
- `autopi_finish` captures structured completion;
- completed/blocked status is reflected in autopi ledger;
- autopi triggers `ctx.compact()` after each task;
- autopi stops safely on failure or missing structured finish;
- commit/push are gated and not automatic by default.

## 17. Future enhancements

- `--compress new-session` mode.
- Worktree-per-task parallelism.
- GitHub Issues integration.
- PR creation after a task batch.
- Web/TUI dashboard widget for current run status.
- Retry policy for transient test failures.
- Config file, e.g. `.autopi.json`.
- Rich TODO metadata: priority, labels, dependencies, estimated risk.

## 18. Open questions

- Should TODO status updates be written directly to `TODO.md` or only to autopi ledger by default?
- Should `--commit auto` be allowed before `new-session` mode exists?
- What exact schema should `autopi_finish` expose for SemVer/changelog decisions?
- Should autopi ship as extension-only first, or as Pi Package with extension + skill + future CLI?
