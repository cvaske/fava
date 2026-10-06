# Phase 0 reconnaissance notes

Reference checkout: `vendor/fava` at `e427597` ("json_api: use RFC 9457 error
format", 2026-10-02, unreleased `main`). Installed for running and testing:
**Fava 1.30.16** (latest PyPI release), **Beancount 3.2.3**, **beangulp 0.2.0**,
Flask 3.1.3, Python 3.13. Line numbers refer to `vendor/fava` (main). Where the
1.30.16 release differs, that is called out.

Facts marked **(tested)** were confirmed by running code against Fava 1.30.16,
not only by reading it. The smoke test used a throwaway extension package on
`sys.path` and Flask's test client.

---

## 1. Importer execution

**Where:** `src/fava/core/ingest.py`, class `IngestModule` (line 299), exposed
as `ledger.ingest`.

- `ledger.ingest.import_data() -> list[FileImporters]` (line 344) walks every
  `import-dirs` directory (`walk_dir`, line 133; skips `.git`, `node_modules`
  and similar, and files over 8 MB). For each file it lists the importers whose
  `identify()` matched, as `FileImportInfo(importer_name, account, date, name)`.
- `ledger.ingest.extract(filename: str, importer_name: str) -> list[Directive]`
  (line 377) does the following:
  1. Calls `extract_from_file` (line 235), which runs `importer.extract(path,
     existing=ledger.all_entries)`, then `importer.sort(entries)`, then
     `importer.deduplicate(entries, existing=...)`.
  2. Runs each function in `HOOKS` from the import config.
- `ledger.ingest.load_file()` (re)loads the config by `runpy`. It runs on every
  ledger load and is cached by the config file's mtime.
- Errors raise `FavaAPIError` subclasses: `MissingImporterConfigError`,
  `ImporterExtractError`, and `ImporterHookError` (main only).

**[VERIFY] Can an extension call this via `self.ledger`?** Yes. `self.ledger`
is the `FavaLedger`, so `self.ledger.ingest.import_data()` and
`self.ledger.ingest.extract(...)` are plain method calls. The JSON API does the
same thing (`json_api.get_extract`, line 353, which calls `ledger.changed()`
first).

**Duplicate detection:** handled by beangulp, not Fava.
`Importer.deduplicate` (beangulp `importer.py:126`) uses `Importer.cmp`
(default: `similar.heuristic_comparator()`) within a ±2-day window. It sets
`entry.meta["__duplicate__"]` to the matched existing entry, or `True` when set
through the `mark_duplicate_entries` helper. Fava's frontend treats any truthy
`__duplicate__` as a duplicate (`frontend/src/entries/index.ts:103`). Hooks may
add or remove the flag too. Detection therefore always runs, unless an
importer overrides `deduplicate`. **The badge can rely on `__duplicate__`.**

**Hook signature differences:**
- 1.30.16: hooks take either `[(filename, entries)]` (old v2 style) or
  `[(filename, entries, account, importer)]`, chosen by inspecting the hook's
  annotations.
- main (unreleased): only the 4-tuple form. Output is validated with msgspec.
  Errors are wrapped in `ImporterHookError`.

This does not affect us, because we never call hooks ourselves. We call
`ledger.ingest.extract`.

**Beancount 2 vs 3:** Fava 1.30.13 dropped Beancount 2 (see `CHANGES`) and
requires `beancount>=3.2,<4` and `beangulp>=0.2`. **Decision: support Beancount
3 and beangulp only.** That removes the v2/v3 branch the plan anticipated.

## 2. Inserting entries

**Where:** `src/fava/core/file.py`.

- `ledger.file.insert_entries(entries: Sequence[Directive]) -> None` (line 238)
  does the following:
  - Takes the file lock (`threading.Lock`, line 115).
  - Calls `ledger.changed()`.
  - Sorts the entries.
  - For each entry, calls `insert_entry(entry, default_file or main file,
    insert_options=fava_options.insert_entry,
    currency_column=fava_options.currency_column, indent=fava_options.indent)`
    (line 449).
  - Updates the in-memory `insert-entry` line numbers.
  - Calls `watcher.notify(path)` and fires `after_insert_entry`.
- `find_insert_position(entry, insert_options, default_filename)` (line 506)
  implements the `insert-entry` option. It takes the newest option dated
  strictly before the entry whose regex matches one of the entry's accounts
  (postings checked last-first). If none matches, it appends to the default
  file (`lineno=None`).

**[VERIFY] Reusable, honors `insert-entry` and `indent`?** Yes.
`insert_entries` is exactly what the "Add entry" form and the importer's Save
button use (`json_api.put_add_entries`, line 541).

**Important caveat: it re-serializes.** `insert_entry` writes
`to_string(entry, currency_column, indent)`, not the user's text. As a result:

- Comments inside the entry are **dropped**.
- Amounts are re-aligned to `currency-column`.
- Metadata keys starting with `_` are dropped.

Everything Beancount parses (all postings, posting and entry metadata, tags,
links, costs, prices, and auto-postings without amounts) round-trips
**(tested)**.

Options for Phase 2:
- **(a)** Accept the loss of comments and use `insert_entries` (simplest, and
  keeps the adapter thin). **Recommended for now.**
- **(b)** Call `find_insert_position` and write the raw text ourselves. This
  keeps comments and formatting, but means reimplementing the line-number
  bookkeeping for `insert-entry` options (`insert_entry`, lines 449–503).

**Reload:** `insert_entries` does **not** reload synchronously. It notifies the
watcher, and the next `ledger.changed()` reloads. Fava calls `ledger.changed()`
in a `before_request` hook for every non-`json_api` request
(`application.py:260–269`). **Extension endpoints are included**, so the
request *after* an accept sees the reloaded ledger. To see the new state within
the same request, call `self.ledger.changed()` ourselves.

**Extension instances survive reloads.** `ExtensionModule.load_file`
(`core/extensions.py:73`) instantiates each class only once
(`if cls not in self._loaded_extensions`). In-memory pending state on `self`
persists across ledger reloads, so no special handling is needed.

## 3. Entry → text

`fava.beans.str.to_string(entry, currency_column=61, indent=2)`
(`beans/str.py:96`) wraps `beancount.parser.printer.format_entry`, strips
`_`-prefixed metadata (line 102, which removes `__source__` and
`__duplicate__`), and aligns amounts. Use `ledger.fava_options.currency_column`
and `.indent`. `filename` and `lineno` metadata are not printed. **(tested)**

Gotcha: `import fava.beans.str` before `fava.core` raises a circular
`ImportError` (`fava.beans.str` → `fava.core.misc` → `fava.core` →
`fava.beans.str`). The adapter must `import fava.core` first. Inside a running
Fava this is already loaded. It matters for standalone tests.

## 4. Text → entry (validation)

Fava has **no** snippet validator. The source-slice editor
(`file.save_entry_slice`) writes text blindly and reports errors after reload.
We build our own from Beancount primitives **(tested)**:

1. `beancount.parser.parser.parse_string(text, report_filename="<entry>")` →
   `(entries, errors, options)`. Syntax errors carry `source["lineno"]`
   relative to the snippet (1-based).
2. Require exactly one directive, of type Transaction, Balance, or Note.
3. `beancount.parser.booking.book(entries, ledger.options)` interpolates
   missing amounts. Errors here include "more than one auto-posting per
   currency" with the posting's line number. `ledger.options` is the raw
   Beancount options map (`core/__init__.py:408`).
4. `beancount.ops.validation.validate_check_transaction_balances(booked,
   ledger.options)` returns "Transaction does not balance: (-1 USD)" with a
   line number.
5. Own check: every account must exist in `ledger.accounts` and be open on the
   entry's date. Beancount's `validate_open_close` needs the whole ledger, and
   steps 1–4 do **not** flag unknown accounts.

Balance assertions cannot be checked in isolation. They are verified after
insertion by the normal ledger load, and new load errors could be surfaced
then. Insert the **unbooked** parsed entry, so that auto-postings stay without
amounts in the file.

## 5. Extension JS loading

- Discovery (`core/extensions.py:46–80`, `ext/__init__.py:137`): for `custom
  "fava-extension" "mod"`, Fava runs `importlib.import_module("mod")` with the
  ledger's directory temporarily on `sys.path`. It collects every
  `FavaExtensionBase` subclass from `inspect.getmembers(module)` (line 166), so
  the class must be importable **from the top-level module**
  (`fava_import_review/__init__.py` must re-export it). A pip-installed package
  works.
- `ext.name` is the class `__qualname__` (`ImportReview`).
  `ext.extension_dir` is `Path(inspect.getfile(cls)).parent` (line 97), the
  directory of the **module defining the class**. `ImportReview.js` and
  `templates/` must sit next to `extension.py`.
- JS is served by `GET /<bfile>/extension_js_module/<Name>.js`
  (`application.py:365`) via `send_file`, with `Content-Type:
  text/javascript` **(tested)**.

**[VERIFY] A third-party package can ship a prebuilt JS file:** **yes
(tested)**.

- **Module format:** ESM loaded with dynamic `import(url)`. It must
  `export default {init?, onPageLoad?, onExtensionPageLoad?}`
  (`frontend/src/extensions.ts`; types in `frontend/src/extension-api.d.ts`).
- **Context:** all three callbacks receive `ctx: ExtensionContext = {api}`.
  `api.request(endpoint, method, params?, body?, output?)`, `get`, `put`,
  `post`, and `delete` build the URL `extension/<Name>/<endpoint>` (keeping
  `bfile`, and filters as query params) and send JSON bodies.
- **Lifecycle:** `init` runs once. `onPageLoad` runs after every page load.
  `onExtensionPageLoad` runs after each navigation to `extension/<Name>...`.
- **SPA caveat:** Fava's router fetches the page and sets
  `article.innerHTML = content` (`frontend/src/reports/route.ts:73`), so
  **inline `<script>` in our template would not run** on in-app navigation.
  The plan's "inline JS fallback" is therefore not viable, and is not needed.
  Our JS must:
  - mount from `onExtensionPageLoad`;
  - find `#import-review` in the DOM;
  - clean up global listeners when the user navigates away. Detect this when
    the root element is no longer `isConnected`, or on the next `onPageLoad`
    for a non-extension page.
- **CSP:** none. No `Content-Security-Policy` header is set anywhere in Fava
  **(tested)**.

## 6. Endpoints

- `@extension_endpoint` uses the function name and GET. The other forms are
  `@extension_endpoint("name", ["POST"])`, or `None` with a methods list
  (`ext/__init__.py:189`).
- URL: `/<bfile>/extension/<Name>/<endpoint>`, with GET, POST, PUT, and DELETE
  allowed (`application.py:347`). The handler is called with **no arguments
  other than `self`**. Read input from `flask.request` (`request.args`,
  `request.get_json()`).
- Response: anything Flask's `make_response` accepts. A **`dict` or `list`
  becomes JSON** **(tested)**. `None` → 404.
- Errors: an uncaught exception → a 500 HTML page. A `FavaAPIError` → a 500
  HTML `_layout` page (`application.py:291`). **Our endpoints must catch errors
  and return JSON** `{ok: false, errors: [...]}` with an explicit status, since
  `ctx.api.request` calls `response.json()` unconditionally.
- **Read-only mode:** `--read-only` aborts every non-GET request with 401.
  Accept and discard will fail, so surface this in the UI.
- `before_request` reloads the ledger if files changed (see §2).

## 7. Template

- `templates/ImportReview.html` is rendered with `ledger` and `extension` (plus
  Fava's globals: `url_for`, `static_url`, `_`, and so on) and inserted into
  `_layout.html`'s `<article>` (`application.py:373–385`).
- A bare `<div id="import-review"></div>` plus a `<style>` block or class-based
  CSS works. `<style>` inside innerHTML-inserted content **does** apply, while
  `<script>` does not. Initial data can be embedded as `<script
  type="application/json">` and read by our JS (the same pattern Fava uses for
  `<svelte-component>`).
- **Theme variables (confirmed in 1.30.16's bundled CSS and in
  `frontend/css/style.css`):**
  - `--background`, `--background-darker`, `--background-darkest`
  - `--border`, `--border-lighter`, `--border-darker`
  - `--text-color`, `--text-color-lighter`, `--text-color-lightest`
  - `--link-color`, `--error`, `--warning`, `--green`
  - `--code-background`
  - `--font-family-monospaced`, `--font-family-editor`
  - `--button-color`, `--button-background`, `--box-shadow-kbd`
  - editor colours: `--editor-account`, `--editor-date`, `--editor-string`,
    `--editor-number`, `--editor-currencies`, `--editor-comment`,
    `--editor-directive`, `--editor-label-name`, `--editor-activeline`

  These use CSS `light-dark()`. Dark mode follows `:root[data-color-scheme]` or
  the OS preference automatically, so using the variables is enough.

## 8. Existing import UI (`frontend/src/reports/import/*`)

Flow:

1. The Import page lists files from `import_data()`: importable and
   non-importable files, an upload form, and a document preview.
2. Per file and importer, "Extract" opens a **modal** (`Extract.svelte`) showing
   one entry at a time in the form widgets (`entry-forms/Entry.svelte`,
   `Transaction.svelte`, `Posting.svelte`, `AddMetadataButton.svelte`).
   - Next/Previous buttons and ⏮/⏭ move between entries. Submitting the form
     (Enter) moves to the next entry.
   - An "ignore duplicate" checkbox toggles `__duplicate__`.
   - `__source__` and `meta.lineno` are shown under the form.
3. **Nothing is written until "Save" on the last entry.** Then all
   non-duplicate entries go to `put_add_entries` together.
4. An in-memory `extract_cache` per `file:importer` keeps unfinished work while
   on the page. A router interrupt handler warns before leaving with
   unfinished imports.
5. **Moving files is a separate manual action, not automatic.** Each
   file/importer row has an account input, a filename (from
   `importer.filename()` / `.date()`), and a "Move" button calling `put_move`
   (`json_api.py:391`). `put_move` moves the file to `<documents>/<account
   path>/<name>` with `shutil.move`. A "×" button deletes the file.

**Behaviour to preserve or offer:**
- Duplicates are skipped by default.
- `__source__` is displayed.
- Warn before leaving with pending edits.
- An optional "move processed file to documents" action. Fava never does this
  implicitly, so it stays an explicit, configurable action (Phase 6, not
  automatic).

**Difference by design:** the plan's queue accepts entries **one at a time**,
writing each immediately. Fava's flow is all-or-nothing per file.

## 9. Versions and minimum support

| Feature we use | Since |
|---|---|
| `has_js_module` and JS lifecycle | 1.25 |
| `extension_endpoint` | 1.26 |
| `ctx.api` helpers | ≥1.26 (present in 1.30.16) |
| Beancount 3 / beangulp ingest | 1.30 |
| Beancount 2 dropped | 1.30.13 |

**Proposed pin:** `fava>=1.30.13,<1.32`, Beancount 3 only. Test in CI against
1.30.16 and `main`. Fava has no 1.31 release yet. `main` changes nothing we
use, apart from the hook validation, which does not affect us.

## Global keyboard shortcuts in Fava (affects Phase 3)

Fava registers one `document` `keydown` listener in the bubble phase
(`frontend/src/keyboard-shortcuts.ts:258`). It ignores editable targets
(line 84). Bindings (from `frontend/src/sidebar/*.svelte`, journal filters,
and chart switcher):

| Key | Action |
|---|---|
| `?` | Show shortcut tooltips |
| `n` | New entry |
| `r` | Reload |
| `g i` `g b` `g t` `g j` `g q` `g h` `g c` `g d` `g E` `g s` `g e` `g n` `g o` `g H` `g a` | Go to report |
| `f …`, `d …`, `s …`, `t …` | Journal filter sequences (journal page only) |
| `Ctrl+s` | Editor save |
| `Ctrl+Enter` | Query submit (query page only) |

**Conflicts with our map:**
- `?` would trigger both overlays.
- `g g` is free, but a bare `g` is Fava's prefix.
- `s` is a two-key prefix only where journal filters are mounted (not on our
  page).

**Plan:** register our listener on `document` in the **capture** phase while
our page is mounted. Call `stopImmediatePropagation()` + `preventDefault()`
only for keys we handle. Let a bare `g` pass through, so Fava's `g <x>`
navigation keeps working, and handle the second `g` of `g g` ourselves.

## Config string

The `fava-extension` config is parsed with `ast.literal_eval`
(`ext/__init__.py:87`). Use a **dict literal**, for example:

```
2020-01-01 custom "fava-extension" "fava_import_review" "{'placeholder': 'Expenses:Uncategorized'}"
```

A bare word is a `SyntaxError`, which Fava reports as an extension error. Other
malformed literals raise `ValueError`, which Fava does **not** catch.

## [VERIFY] outcomes

| Item | Outcome |
|---|---|
| Can an extension run importers via `self.ledger`? | ✅ `ledger.ingest.import_data()` / `.extract()` |
| Reusable insert that honors `insert-entry` and indent? | ✅ `ledger.file.insert_entries()`, but it re-serializes (comments lost) |
| Frontend is Svelte, not SvelteKit | ✅ Svelte 5 with its own router. We don't touch its bundle. |
| A third-party package can ship and serve JS | ✅ tested |
| CSP restrictions | ✅ none |
| Theme CSS variable names | ✅ listed in §7 |
| Inline-script fallback | ❌ won't run under SPA navigation. Not needed. |
| Beancount 2 support | ➖ dropped upstream in 1.30.13. Out of scope. |
