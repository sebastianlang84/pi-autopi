# Design: pi-autopi MVP

## Status

Draft design for the first implementable MVP. This resolves the PRD open questions conservatively; later autonomy should be added only behind explicit flags.

## Product shape

Ship first as a single Pi Extension packaged with:

```text
package.json
src/index.ts
src/todo.ts
src/ledger.ts
src/git.ts
src/schema.ts
```

`package.json` should declare:

```json
{
  "pi": {
    "extensions": ["./src/index.ts"]
  }
}
```

A future CLI can reuse the parser, ledger, and Git helpers after the extension workflow is proven.

## Commands

User-facing MVP commands:

```text
/autopi TODO.md [--max N] [--commit off|ask] [--push off|ask] [--update-todo off|ask|auto]
/autopi status
/autopi stop
```

Defaults:

```text
max=1
commit=ask
push=ask
update-todo=off
compress=compact
stopOnFailure=true
stopOnDirtyUnrelatedChanges=true
```

`commit=auto`, `push=auto`, and `compress=new-session` are post-MVP.

## Runtime loop

1. `/autopi TODO.md` starts or resumes a run.
2. Parse TODO tasks and record file hash/mtime.
3. Load ledger, reconcile against the current TODO file, and skip tasks already completed, blocked, or failed in this run.
4. Check Git state before assigning work.
5. Send one task prompt to the agent via `pi.sendUserMessage(...)`.
6. Agent works normally.
7. Agent must call `autopi_finish` exactly once.
8. `autopi_finish` validates payload, records result, and returns `terminate: true`.
9. The tool schedules an internal follow-up command, e.g. `/autopi __continue <runId>`.
10. Continue handler performs deterministic closeout: Git gate, optional TODO update, compaction, next task or stop.

The agent never chooses the next TODO item; only autopi does.

## Task prompt contract

Keep the injected prompt short:

```text
Autopi task <taskId> from <todoFile>:
<task text>

Rules: work only on this task; do not start other TODO items; verify relevant changes; finish by calling autopi_finish once. If blocked, call autopi_finish with status=blocked.
```

Avoid large prompt snippets. Detailed policy lives in docs and deterministic extension code.

## `autopi_finish` schema

MVP tool name: `autopi_finish`.

Required fields:

```json
{
  "taskId": "string",
  "status": "done | blocked | failed",
  "summary": "string",
  "changedFiles": ["string"],
  "verification": [
    {
      "command": "string",
      "status": "passed | failed | not_run",
      "exitCode": 0,
      "notes": "string"
    }
  ],
  "blockers": ["string"],
  "openQuestions": ["string"]
}
```

Optional fields:

```json
{
  "proposedCommitMessage": "string",
  "semverImpact": "no bump | patch | minor | major",
  "changelogNeeded": true
}
```

Validation rules:

- `taskId` must equal the active ledger task.
- `done` requires non-empty `summary`.
- `blocked` requires at least one blocker or open question.
- `failed` requires a summary explaining the failure.
- `changedFiles` must be relative paths when possible.
- `done` with code changes requires at least one `verification` entry with `status=passed`, or commit is blocked.
- Any `verification.status=failed` blocks commit and next-task progress unless the user overrides.
- Unknown enum values are rejected.

The tool result should be terminating so the closeout does not require an extra LLM turn.

## Ledger

Store durable state outside LLM context in the Pi extension state directory:

```text
~/.pi/agent/state/pi-autopi/runs/<runId>.json
```

Implement this with Node filesystem APIs under Pi's extension-state convention. Also append compact custom session entries with `pi.appendEntry(...)` for branch-aware visibility, but the JSON ledger is authoritative for resume.

Minimum ledger shape:

```json
{
  "schemaVersion": 1,
  "runId": "string",
  "cwd": "string",
  "todoFile": "TODO.md",
  "todoFileHash": "sha256:string",
  "todoFileMtimeMs": 0,
  "todoParsedAt": 0,
  "startedAt": 0,
  "updatedAt": 0,
  "status": "running | stopped | completed | failed",
  "currentTaskId": "string | null",
  "tasks": {
    "task-id": {
      "text": "string",
      "sourceLine": 12,
      "status": "pending | running | done | blocked | failed",
      "result": null
    }
  },
  "options": {
    "max": 1,
    "commit": "ask",
    "push": "ask",
    "updateTodo": "off",
    "compress": "compact"
  },
  "git": {
    "branch": "string | null",
    "baseStatus": "string",
    "lastCommit": "string | null"
  }
}
```

Use an atomic write pattern: write temp file, then rename.

## TODO parsing and IDs

Supported MVP syntax:

```md
- [ ] Task text
- [x] Completed task
- [ ] <!-- id:stable-id --> Task with stable ID
```

ID rules:

1. Prefer `<!-- id:... -->`.
2. Else derive `path:line:normalized-text-hash`.
3. If duplicate IDs exist, stop before running.
4. Reparse the current TODO file before selecting each next task.
5. If the TODO file changed and an active or completed task cannot be reconciled by stable ID, stop and ask.
6. Do not attempt full Markdown task-list semantics in MVP.

## TODO update policy

Ledger is authoritative by default. `TODO.md` is not modified unless requested.

- `--update-todo off`: never edit `TODO.md`.
- `--update-todo ask`: ask before marking the item.
- `--update-todo auto`: mark `done` tasks as `[x]`; append an HTML comment for blocked/failed only if the original line has a stable ID.

If the TODO file changed since parsing, stop and ask instead of patching.

## Git policy

Before each task:

- run `git status --porcelain=v1 --branch` if inside a Git repo;
- stop on dirty state unless changes are already recorded as owned by the active task;
- record branch and upstream summary.

After `autopi_finish`:

- inspect final status;
- compare changed files to `changedFiles` and warn on mismatch;
- do not commit if verification is missing, failed, or ambiguous;
- `commit=ask` proposes the exact command/message and waits;
- `push=ask` proposes remote/branch and waits;
- no automatic push in MVP.

SemVer uses SemVer 2.0.0. Docs-only design work is usually `no bump` unless package policy says otherwise.

## Verification policy

MVP records structured verification reported by the agent: command, pass/fail/not-run status, optional exit code, and notes. Autopi must not blindly execute arbitrary proposed commands.

Later option:

```text
--verify ask|auto --verify-allow "npm test" --verify-allow "pnpm test"
```

Until then, failed or absent verification blocks auto-commit and is shown in status.

## Compaction

After deterministic closeout, call `ctx.compact(...)` with focused instructions preserving only:

- run id, TODO path, current/next task;
- latest result;
- changed files;
- verification;
- commit SHA if any;
- blockers/open questions.

If compaction fails, stop the run and preserve ledger state.

## Stop conditions

Stop on:

- TODO parse errors or duplicate IDs;
- no open tasks;
- dirty unrelated Git state;
- active task mismatch in `autopi_finish`;
- missing `autopi_finish` after agent end;
- blocked/failed result;
- commit or push needs approval;
- TODO patch conflict;
- compaction failure;
- max task count reached.

## MVP acceptance checklist

- `/autopi TODO.md --max N` selects one unchecked task at a time.
- Agent receives only that task.
- `autopi_finish` is required for progress.
- Ledger survives Pi restart.
- Duplicate TODO IDs stop safely.
- Dirty repo stops safely.
- Commit/push are never automatic by default.
- `ctx.compact()` is triggered after each completed closeout.
- `/autopi status` reports current run, task, last result, and stop reason.
