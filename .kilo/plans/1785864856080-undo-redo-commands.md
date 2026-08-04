# Plan: Add `/undo` and `/redo` commands to nanocoder

## Goal
Give users a way to revert (and re-apply) the **file changes** the agent made,
by walking an automatic per-turn undo/redo stack. Conversation messages are NOT
reverted (undo affects the workspace only).

## Resolved design decisions
- **Scope:** file changes only (workspace files, not chat history).
- **History population:** automatic lightweight snapshot at the start of every agent turn.
- **Lifecycle:** in-memory per session — lost on `/exit`; `/resume` starts a fresh stack.
- **Reset triggers:** `/clear`, `/compact`, and `/checkpoint load` reset the stack.
- **Fidelity:** full — revert edits to existing files, recreate git-tracked files the agent
  deleted, and delete files the agent created during a turn. (Untracked-and-deleted files
  cannot be restored — documented limitation.)

## New files
### `source/services/undo-redo-manager.ts`
Singleton (mirror `CheckpointManager`'s `getDefaultCheckpointManager()`):
- Holds `history: TurnSnapshot[]` and `pointer: number`.
- `captureTurnSnapshot(): Promise<void>` — capture current file state via the
  enhanced `FileSnapshotService`; **dedupe** vs `history[pointer]` (only push when the
  snapshot actually differs); truncate any redo tail before pushing.
- `undo(): Promise<UndoResult>` — if `pointer > 0`: `pointer--`, `restoreSnapshot(history[pointer])`,
  return affected files. Else `{reverted: false, reason: 'Nothing to undo'}`.
- `redo(): Promise<UndoResult>` — if `pointer < history.length-1`: `pointer++`, restore, return files.
  Else `{reverted: false, reason: 'Nothing to redo'}`.
- `canUndo()`, `canRedo()`, `reset()` (used by /clear, /compact, /checkpoint load).

### `source/commands/undo.tsx` and `source/commands/redo.tsx`
Real `Command` modules (like `checkpoint.tsx`). Each handler calls the singleton and
returns a `successMsg`/`infoMsg`/`errorMsg` describing the files reverted/restored, or an
informative message at the boundary (e.g. "Nothing to undo"). Use `addToMessageQueue` for
multi-step feedback like `checkpoint.tsx` if needed.

## Modified files
### `source/services/file-snapshot.ts`
Extend `FileSnapshotService` to support full-fidelity snapshots:
- `captureState(): Promise<{contents: Map<string,string>; deleted: string[]}>` — for each
  file from `getModifiedFiles()`: if it still exists, read content (modify/create);
  if it is a **deleted** git-tracked file, capture a deletion marker and, when possible,
  read its prior content via `git show HEAD:<path>` (for recreation on undo). Honor
  `.gitignore` (already done in `getModifiedFiles`).
- `restoreState(snapshot)`:
  1. Write every `contents` entry (recreates deleted files, overwrites edits/creations).
  2. `fs.rm` every path in `deleted` if present.
  3. Remove any file currently present in the working tree that is **absent** from both
     `contents` and `deleted` (i.e. created after the target snapshot) — mirrors the
     "revert to this snapshot" semantics. Scope deletions to git-tracked/untracked-modified
     files (not arbitrary ignored files) to avoid wiping build artifacts.

### `source/commands/lazy-registry.ts`
Add `undo` and `redo` entries (name + description + `load()`) consistent with the module
exports (`undoCommand`, `redoCommand`). Keep description in sync with the command modules.

### `source/hooks/chat-handler/useChatHandler.tsx`
At the **start** of `handleChatMessage` (line ~302, before the model/tool loop runs), call
`undoRedoManager.captureTurnSnapshot()`. This covers normal turns and `/retry` (which routes
through `onHandleChatMessage`). Slash commands (`/undo`, `/redo`, `/clear`, etc.) go through
`handleSlashCommand` and therefore do NOT trigger a snapshot — correct.

### Reset hooks
- `source/commands/clear.tsx` — call `undoRedoManager.reset()` after clearing.
- `source/app/utils/handlers/compact-handler.ts` — call `reset()` when a compact completes.
- `source/commands/checkpoint.tsx` — call `reset()` after a checkpoint `load`/`restore`.

## Data model & algorithm
- A `TurnSnapshot` = `{ contents: Map<relPath,string>, deleted: string[] }` of the workspace
  file set at the start of a turn.
- Stack invariant: `history[pointer]` is the current on-disk state. New turn → push (deduped),
  `pointer = end`. `undo` moves `pointer` back and restores; `redo` moves forward. A new turn
  after an undo truncates the redo tail.
- Snapshots only contain git-modified/untracked files, so empty/clean turns push nothing.

## Edge cases & risks
- **Untracked-and-deleted files:** content is already gone; cannot be restored. Document as
  a known limitation (undo will still remove the file from disk but cannot bring it back).
- **Large diffs:** bounded by `MAX_CHECKPOINT_FILES` and `.gitignore` filtering (reuse existing
  guards in `FileSnapshotService`).
- **Non-git workspace:** `getModifiedFiles()` returns `[]`; undo still reverts edits to any
  files captured via the untracked path where git works, but deletion detection needs git.
- **Permission errors on restore:** surface via `errorMsg` and do not corrupt `pointer`.
- **No history / at boundary:** return informative message, leave state unchanged.

## Validation
- Unit tests for `undo-redo-manager` (capture dedup, undo/redo stepping, boundary no-ops,
  reset) and for `file-snapshot` `captureState`/`restoreState` (edit, create, delete,
  recreate-deleted). Use a temp git repo in tests.
- `lazy-registry` test asserting `undo`/`redo` entries exist (mirror the existing `retry` test
  in `app-util.spec.ts`).
- Manual: in a git repo, ask the agent to create+edit+delete a file, then run `/undo` (files
  revert), `/redo` (re-applied); confirm `/clear` and `/checkpoint load` reset the stack.
- Run `pnpm run test:all` (format, lint, types, AVA, knip) before considering done.
