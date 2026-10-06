# Plan: `fava-import-review`, a keyboard-first import review extension for Fava

> **Status:** Phase 0 done (see `NOTES.md`). The amendments from Phase 0 are
> listed first. The original plan follows, with `[VERIFY]` markers replaced by
> their outcomes.

## Amendments after Phase 0

1. **Beancount 3 / beangulp only.** Fava dropped Beancount 2 in 1.30.13. Pin
   `fava>=1.30.13,<1.32`. The "detect the version in the adapter" mitigation
   is removed.
2. **Duplicate detection always runs** (beangulp `Importer.deduplicate`) and
   marks `meta["__duplicate__"]`. `is_duplicate` reads that flag. It is not
   "best effort, only if the hook ran".
3. **Insertion re-serializes text.** `ledger.file.insert_entries` writes
   `to_string(entry)`. Comments inside an edited entry are dropped and amounts
   are re-aligned. Phase 2 uses `insert_entries`, and the README documents the
   limitation. Preserving raw text (via `find_insert_position` plus our own
   write) is a possible later option. It would mean reimplementing
   `insert-entry` line bookkeeping, so it is deferred.
4. **Validation is ours.** Fava has no snippet validator. Pipeline:
   - `parse_string`;
   - require exactly one Transaction, Balance, or Note;
   - `booking.book(…, ledger.options)`;
   - `validate_check_transaction_balances`;
   - check that every account is open on the entry date.

   Balance assertions are only checked after insertion.
5. **No inline-script fallback.** Fava inserts report HTML via `innerHTML`, so
   template `<script>` tags never run. Shipping `ImportReview.js` next to
   `extension.py` is tested and works. The JS mounts in
   `onExtensionPageLoad`, and must unmount (remove global listeners) when the
   page goes away.
6. **Endpoints catch everything and return JSON.** Fava turns exceptions into
   HTML 500 pages, and `ctx.api` always calls `.json()`. Handlers read
   `flask.request` because they take no arguments.
7. **Keyboard:** our `keydown` listener runs in the capture phase on
   `document`. It stops propagation only for keys it handles, so Fava's global
   `?` overlay doesn't double-fire. A bare `g` passes through, so `g <x>`
   navigation still works.
8. **Config is a Python dict literal** (`ast.literal_eval`), for example
   `"{'placeholder': 'Expenses:Uncategorized'}"`.
9. **No automatic file moving.** Fava's "Move" is a separate manual action.
   We may offer an explicit "move source file to documents" action in Phase 6,
   but never do it implicitly.
10. **Read-only mode** (`fava --read-only`) rejects every non-GET request with
    401. The UI must report this instead of failing silently.
11. Pending IDs: `"<filename>::<importer>::<index>"`. Pending state lives on the
    extension instance, which survives ledger reloads.
12. **Repository hosting (interim):** the package lives on an orphan branch
    `claude/fava-import-review` of `cvaske/fava`. That branch shares no history
    with Fava and contains only this package, so it can be pushed to a
    standalone repo unchanged. `vendor/fava` is a gitignored clone of
    `beancount/fava`.

---

## Goal

Build a standalone Fava extension (a separate pip package, with no changes to
Fava core). It replaces the painful import flow with a **keyboard-driven review
queue**:

- A list of pending imported entries, navigated with the keyboard.
- Each entry is edited as **plain Beancount text**, not form widgets. This
  removes the "add/remove posting" and "add metadata" mouse widgets.
- Validate, accept, and advance without touching the mouse.

## Ground rules

1. Do not modify Fava itself. `./vendor/fava` (gitignored) is for reading only.
2. Pin the supported Fava version range. Keep all Fava-specific calls in
   `fava_import_review/fava_adapter.py`.
3. Work phase by phase. Commit after each phase. Stop at each Checkpoint and
   summarize.
4. Prefer small, dependency-light code. Use vanilla TypeScript and esbuild. Our
   JS is self-contained.

## Phase 1: Scaffold and test fixtures

Layout:

```
pyproject.toml, README.md, NOTES.md, PLAN.md, Makefile
src/fava_import_review/
  __init__.py          # re-exports ImportReview (required for discovery)
  extension.py         # ImportReview(FavaExtensionBase); JS and templates sit next to it
  fava_adapter.py      # ALL Fava-internal calls
  pending.py, editing.py
  templates/ImportReview.html
  ImportReview.js      # built by esbuild, committed
frontend/ (package.json, src/*.ts, style.css, esbuild config)
tests/fixtures/ (ledger.beancount, import_config.py, inbox/sample.csv)
tests/test_pending.py, test_editing.py, test_endpoints.py
```

- Fixtures:
  - a plain 2-posting transaction;
  - a 3+-posting split;
  - a duplicate of an existing entry;
  - rows with `__source__`;
  - the `import-config`, `import-dirs`, and `insert-entry` options.
- `make dev | build | test`.

**Checkpoint 1:** the sidebar shows "Import review", and it renders "hello"
from our JS.

## Phase 2: Backend

`PendingEntry{id, filename, importer, text, source, is_duplicate, errors}`.

| Method | Path | Purpose |
|---|---|---|
| GET | `pending` | Return the queue. Runs importers on first use only. |
| POST | `refresh` | Re-run importers explicitly. |
| POST | `validate` | `{text}` → `{ok, errors:[{line, message}]}` |
| POST | `accept` | `{id, text}`: re-validate server-side, insert, drop from queue. |
| POST | `discard` | `{id}`: drop from the session queue. |
| GET | `completions` | `{accounts, payees, narrations, tags, links, currencies}` |

Rules:
- `accept` always re-validates.
- Any number of postings and metadata is allowed.
- Reject text with zero or several directives, and unsupported directive
  types.
- Use `insert_entries`, so the `insert-entry` and indent options are honored.
- Pending state lives in memory on the extension instance. Accepting an entry
  removes it from the queue. A ledger reload does not re-run importers, so
  only `refresh` does. On `refresh`, entries already accepted in this session
  are flagged as duplicates by beangulp's own dedup (they are now in the
  ledger).

Tests:
- `pending`
- `validate`: 2-posting, 4-posting, unbalanced, syntax error, multiple
  directives, empty, unknown account
- `accept`
- `completions`

**Checkpoint 2:** tests pass, plus a curl transcript.

## Phase 3: Frontend list and navigation (read-only)

Entry cards (monospace text, dimmed source, `duplicate` and `error` badges)
and progress in the header. The cursor is a state `id`, kept visible with
`scrollIntoView({block:"nearest"})`. One capture-phase `keydown` listener (see
amendment 7) ignores editable targets and modifier keys unless the key is
bound.

Keys:

| Key | Action |
|---|---|
| `j` / `↓`, `k` / `↑` | Next / previous entry |
| `g g` / `G` | First / last entry |
| `e` / `Enter` | Edit |
| `a` | Accept |
| `s` | Skip |
| `x` | Discard |
| `u` | Undo |
| `?` | Help overlay |

Styling uses Fava's theme variables (NOTES §7), scoped under `#import-review`.

**Checkpoint 3:** keyboard-only navigation works, and the help overlay
appears.

## Phase 4: Inline editing, validation, accept

`e` opens a textarea. `Ctrl/Cmd+Enter` validates, accepts, and advances (and
re-enters edit mode if "continuous edit" is on). `Esc` cancels.
`Ctrl/Cmd+Shift+Enter` accepts and stays. Validation is debounced (300 ms),
with inline errors and line numbers. On failure, the text is kept. The UI is
optimistic: a rejected card is restored with a banner.

**Checkpoint 4:** a 4-posting edit is accepted and appears in the ledger file
and in Fava's Journal.

## Phase 5: CodeMirror 6 and autocomplete

CodeMirror 6 is bundled. Highlighting uses our own `StreamLanguage` mode.
Completions:
- accounts on posting lines;
- payees and narrations in header strings;
- tags after `#`, links after `^`;
- currencies after a number.

`Ctrl/Cmd+Enter` and `Esc` keep working inside the editor. Optional: Tab from
the account moves to the amount.

**Checkpoint 5:** completions work in all positions. Report the bundle size.

## Phase 6: Productivity features and polish

1. Auto-balance helper (`Ctrl+B`).
2. Bulk accept (`A`) with a confirmation, skipping placeholder accounts.
3. Filter (`/`) and sort (errors and uncategorized first).
4. Undo: discards always. Accepts only if the inserted region can be verified
   unchanged. Otherwise, document the limit.
5. Payee memory (`ledger.attributes.payee_transaction`).
6. Config dict.
7. Accessibility (`listbox` / `aria-activedescendant`).
8. README.
9. Optional explicit "move source file to documents" action.

**Checkpoint 6:** full test run, a list of limitations, and the README.

## Acceptance criteria

- Zero mouse use for 50 entries.
- Any number of postings and metadata lines.
- Invalid entries cannot be saved, and errors show line numbers.
- Entries are inserted at the correct position and appear after reload.
- No changes to Fava core. All Fava calls go through `fava_adapter.py`.
- `pytest` passes, and the wheel contains the template and JS.

## Known risks

| Risk | Mitigation |
|---|---|
| The extension API is unstable | Pin the range, use the adapter, run CI against Fava `main`. |
| Re-serialization drops comments | Document it. Raw-text insert is a possible later option. |
| A reload invalidates pending state | The instance survives reloads, and the queue is decoupled from the ledger. Covered by tests. |
| Keyboard conflicts with Fava's global shortcuts | Capture-phase listener. A bare `g` passes through. |
| Read-only mode | Detect the 401 and show a banner. |
