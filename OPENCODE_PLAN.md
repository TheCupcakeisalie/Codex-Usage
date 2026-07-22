# Implementation Plan: opencode usage reporting in `codex_usage.py`

Status: proposed, not yet implemented
Author: drafted 2026-07-22
Target file: `codex_usage.py` (single-file, stdlib-only — this constraint is preserved)

---

## 1. Goal and scope

Extend `codex_usage.py` so the local-usage side of the tool can report on
[opencode](https://opencode.ai) sessions in addition to Codex sessions, and can show a
combined view across both tools.

**In scope**

- A new local collector that reads opencode's SQLite database.
- A `--tool codex|opencode|all` selector on `local-usage`, `export`, and the `all` report.
- opencode rows in `doctor`.
- opencode totals in the menu quick summary.
- JSON and CSV export parity with the existing Codex local report.

**Out of scope (explicitly)**

- Any opencode network/online reporting. The entire online half of the script
  (`/wham/usage`, reset credits, rate-limit windows, `api-usage` via the OpenAI Admin API)
  is ChatGPT/OpenAI-account specific. opencode has no equivalent endpoint, and inventing
  one would mean shipping provider credentials handling. The combined report stays
  local-only for opencode.
- Pricing tables / cost estimation for models that report `cost: 0`. See §10.4.
- Reading opencode's legacy `storage/*.json` tree. See §3.4.
- Any write to the opencode database. Read-only, always.

---

## 2. Design principles

1. **Mirror the existing contract, don't invent a new one.** `scan_sessions_metadata()`
   returns a dict with a specific key set that `print_local_usage()`, `limit_local_usage_days()`,
   and `rows_for_csv()` all consume. The opencode collector returns the *same* key set so
   the existing rendering, day-trimming, and CSV plumbing work unchanged.
2. **Single file, stdlib only.** `sqlite3` (with the `json1` extension, built into every
   CPython SQLite since 3.38 and enabled by default well before) is all that is needed.
3. **Local-only and privacy-preserving.** Same guarantee as the Codex local report: counters,
   model IDs, dates and truncated paths only. Never message text, titles are optional and
   off by default (see §10.5).
4. **Degrade quietly.** If opencode isn't installed, `--tool codex` behaviour is unchanged
   and `--tool all` prints a one-line "not found" note rather than failing.
5. **Additive.** No existing command, flag, or JSON key changes meaning. `--tool` defaults
   to `codex`, so every current invocation produces byte-identical output.

---

## 3. Findings: the opencode data model

All figures below were measured on this machine (opencode 1.18.4, 2026-07-22) and are
recorded so the implementation can be checked against known-good numbers.

### 3.1 Location

```
$XDG_DATA_HOME/opencode/opencode.db     (default ~/.local/share/opencode/opencode.db)
```

Also present and *not* used: `auth.json`, `account.json`, `log/`, `snapshot/`, `tool-output/`,
`storage/`. Resolution order for the implementation:

1. `$OPENCODE_DATA` if set (mirrors how `resolve_codex_home()` honours `$CODEX_HOME`)
2. `$XDG_DATA_HOME/opencode`
3. `~/.local/share/opencode`

### 3.2 Relevant tables

`session` — one row per session, with pre-aggregated counters:

| column | type | notes |
|---|---|---|
| `id` | text | `ses_...` |
| `parent_id` | text NULL | non-NULL ⇒ subagent/child session |
| `project_id` | text | FK → `project` |
| `directory`, `path` | text | working directory |
| `title` | text | model-generated summary — **treat as sensitive**, see §10.5 |
| `agent` | text | `build`, `plan`, `explore`, … |
| `model` | text (JSON) | `{"id": "...", "providerID": "..."}` — **NULL on older rows** |
| `cost` | real | 0.0 for subscription-billed providers |
| `tokens_input`, `tokens_output`, `tokens_reasoning`, `tokens_cache_read`, `tokens_cache_write` | integer | |
| `time_created`, `time_updated` | integer | epoch **milliseconds** |

`message` — one row per message; `data` is a JSON blob. Assistant rows carry:

```json
{
  "role": "assistant", "agent": "build",
  "modelID": "gpt-5.6-terra", "providerID": "openai",
  "cost": 0,
  "tokens": {"total": 214498, "input": 401, "output": 81, "reasoning": 0,
             "cache": {"write": 0, "read": 214016}},
  "time": {"created": 1784722319203, "completed": 1784722323046},
  "path": {"cwd": "...", "root": "..."},
  "finish": "stop"
}
```

`project` — `id`, `worktree`, `name`, `vcs`. Useful for a per-project table.

`part`, `todo`, `permission`, `event`, `session_share`, … — not needed.

### 3.3 Verified measurements (baseline for tests)

```
sessions:                     761  (246 with parent_id NOT NULL)
sessions with model NULL:     372  (355 of those still have tokens > 0)
messages:                  48,101  (44,446 assistant rows; 0 with a NULL modelID)
date range:            2026-02-11 → 2026-07-22
SUM(session.tokens_input)                        = 243,098,583
SUM(message json_extract tokens.input)           = 243,098,583   ← identical
SUM(session.cost)  = SUM(message cost)           = 21.340209824
```

**Two consequences that drive the design:**

- **Child sessions are *not* rolled up into their parents.** A parent's counters equal the sum
  of its *own* messages only. Verified per-session: `session.tokens_input` exactly equals the
  message-level sum for that session id, and parent counters bear no relation to the sum of
  their children's. Therefore *all* sessions, including `parent_id IS NOT NULL` ones, must be
  included in totals. Filtering them out would undercount by ~30% of sessions. They should
  still be *labelled* as subagent sessions in per-session listings.
- **`session.model` is unreliable (49% NULL) but `message.data.modelID` is universal (0% NULL).**
  So the message table, not the session table, is the source of truth for model attribution.

### 3.4 Why not the legacy `storage/` tree

`~/.local/share/opencode/storage/{session,message,part}/` is the pre-SQLite layout. On this
machine it holds 1 session and 407 messages against 761 sessions and 48,101 messages in the DB
— it is stale residue from before the migration (the `migration` / `data_migration` tables in
the DB confirm the move). Reading it would add a second parser for <0.2% of the data. Skip it.
The `doctor` command should note if it exists and is non-trivially sized, in case a user on an
older opencode version needs to know why totals look empty.

---

## 4. Architecture

### 4.1 New module-level constants

Insert after the existing `CODEX_HOME` block (`codex_usage.py:45-61`):

```python
def resolve_opencode_data_dir() -> Path: ...

OPENCODE_DATA_DIR = resolve_opencode_data_dir()
OPENCODE_DB = OPENCODE_DATA_DIR / "opencode.db"
OPENCODE_LEGACY_STORAGE = OPENCODE_DATA_DIR / "storage"
TOOL_CHOICES = ("codex", "opencode", "all")
```

`USAGE_FIELDS` (`codex_usage.py:62`) is reused verbatim — see the mapping in §5.

### 4.2 New functions

Placed in a clearly delimited `# --- opencode local usage ---` block immediately after
`cmd_local_usage()` (`codex_usage.py:1206`), keeping the Codex collectors contiguous.

| function | responsibility |
|---|---|
| `resolve_opencode_data_dir() -> Path` | env/XDG resolution per §3.1 |
| `connect_opencode_readonly(path: Path) -> sqlite3.Connection` | `mode=ro`, fall back to `immutable=1` on `sqlite3.OperationalError`; see §10.1 |
| `opencode_available() -> bool` | `OPENCODE_DB.is_file()` |
| `scan_opencode_messages(con, top_n) -> dict[str, Any]` | the core aggregation; returns the `scan_sessions_metadata()` key set |
| `opencode_project_totals(con, top_n) -> list[dict]` | per-project/per-directory rollup (opencode-only extra) |
| `opencode_agent_totals(con, top_n) -> list[dict]` | per-agent rollup (opencode-only extra) |
| `collect_opencode_usage(top_n) -> dict[str, Any]` | wrapper mirroring `collect_local_usage()` |
| `opencode_hints(data) -> list[str]` | mirrors `local_hints()` (`codex_usage.py:953`) |
| `print_opencode_usage(data, top, days) -> None` | mirrors `print_local_usage()` (`codex_usage.py:1011`) |
| `opencode_health() -> dict[str, Any]` | for `doctor`; reuses `path_status()` (`codex_usage.py:1216`) |

### 4.3 Aggregation strategy

**Do all aggregation in SQL with `json_extract`, not in Python.** Measured: a full
`GROUP BY` with four `json_extract` calls over all 48,101 message rows completes in
**0.42 s**. Pulling 48k JSON blobs into Python to parse them would be ~50× slower and would
load message bodies (including text parts) into memory unnecessarily.

Canonical CTE, reused by every rollup:

```sql
WITH m AS (
  SELECT
    m.session_id                                            AS session_id,
    m.time_created                                          AS ts_ms,
    json_extract(m.data, '$.providerID')                    AS provider,
    json_extract(m.data, '$.modelID')                       AS model,
    json_extract(m.data, '$.agent')                         AS agent,
    COALESCE(json_extract(m.data, '$.tokens.input'), 0)     AS input_tokens,
    COALESCE(json_extract(m.data, '$.tokens.output'), 0)    AS output_tokens,
    COALESCE(json_extract(m.data, '$.tokens.reasoning'), 0) AS reasoning_tokens,
    COALESCE(json_extract(m.data, '$.tokens.cache.read'), 0)  AS cache_read,
    COALESCE(json_extract(m.data, '$.tokens.cache.write'), 0) AS cache_write,
    COALESCE(json_extract(m.data, '$.cost'), 0.0)           AS cost
  FROM message m
  WHERE json_extract(m.data, '$.role') = 'assistant'
)
```

Daily rollup uses `date(ts_ms/1000, 'unixepoch', 'localtime')` — note **milliseconds** and
**localtime**, to match how the Codex report groups by local session-file date.

`total_tokens` is computed as `input + output` and **not** by adding cache or reasoning
figures. Rationale in §10.2.

### 4.4 Return shape

`scan_opencode_messages()` returns exactly the keys `print_local_usage()` already reads, so
the renderer needs no defensive branching:

```python
{
  "session_files": <int>,                     # session count (label differs, see §6)
  "jsonl_lines_scanned": <int>,               # assistant message count
  "parse_or_read_errors": <int>,
  "files_with_final_token_totals": <int>,     # sessions with tokens > 0
  "file_mtime_start_local": <str|None>,       # min(time_created)
  "file_mtime_end_local": <str|None>,         # max(time_updated)
  "final_token_totals_sum": {<USAGE_FIELDS>: int},
  "models_by_session": [[model, sessions], ...],
  "model_token_totals": {model: {<USAGE_FIELDS>: int}},
  "providers_by_session": [[provider, sessions], ...],
  "context_windows_by_session": [],           # not exposed by opencode; always empty
  "daily_usage": [{"date", "sessions", <USAGE_FIELDS>}, ...],
  "top_sessions_by_total_tokens": [
      {"session_file", "date", "model", "project", "usage"}, ...
  ],
  # opencode-only extras, ignored by the shared renderer:
  "cost_total": <float>,
  "by_agent": [...],
  "by_project": [...],
  "subagent_session_count": <int>,
}
```

`collect_opencode_usage()` wraps it as:

```python
{
  "retrieved_at_local": local_now_text(),
  "tool": "opencode",
  "opencode_data_dir": str(OPENCODE_DATA_DIR),
  "database": str(OPENCODE_DB),
  "network_calls_made": 0,
  "privacy_note": "...",
  "sqlite_threads": {"selected": None, "all": []},   # no analogue; keeps shape stable
  "sessions": <scan_opencode_messages(...)>,
}
```

Keeping `sqlite_threads` present-but-empty means `limit_local_usage_days()`
(`codex_usage.py:2991`) and `rows_for_csv()` (`codex_usage.py:3034`) need no null-guards
beyond what they already have.

---

## 5. Field mapping

| `USAGE_FIELDS` key (Codex) | opencode source |
|---|---|
| `input_tokens` | `tokens.input` |
| `cached_input_tokens` | `tokens.cache.read` |
| `output_tokens` | `tokens.output` |
| `reasoning_output_tokens` | `tokens.reasoning` |
| `total_tokens` | `tokens.input + tokens.output` (computed; see §10.2) |
| *(no analogue)* | `tokens.cache.write` → surfaced only in the opencode-specific cache table |
| *(no analogue)* | `cost` → surfaced only in the opencode cost row |
| `model` | `message.data.modelID` |
| `provider` | `message.data.providerID` (Codex uses `model_provider`) |
| `project` | `session.directory`, rendered through the existing `short_path()` |
| `date` | `date(message.time_created/1000,'unixepoch','localtime')` |
| `context_window` | **unavailable** — opencode does not persist it |

---

## 6. CLI surface

### 6.1 New flag

```
--tool {codex,opencode,all}   Which coding tool's local data to report on. Default: codex.
```

Added to: `local-usage`, `all`, `export`, and `menu` (as a settings-menu entry).
A shared `add_tool_option(parser)` helper alongside `add_common()` (`codex_usage.py:3441`).

### 6.2 Dispatch changes

- `cmd_local_usage()` (`codex_usage.py:1206`) branches on `args.tool`:
  - `codex` → current behaviour, unchanged
  - `opencode` → `collect_opencode_usage()` / `print_opencode_usage()`
  - `all` → both, separated by the existing `"=" * min(terminal_width(), 100)` rule, then a
    combined totals table (§7.3)
- `collect_all()` (`codex_usage.py:2950`) gains `"opencode_usage": collect_opencode_usage(top_n)`
  when the tool selector includes opencode; `print_all()` renders it after the Codex local block.
- `export_json()` (`codex_usage.py:3002`) gains a `tool` parameter threaded through
  `export_report()` and `cmd_export()`.
- `export_path()` (`codex_usage.py:3095`) filename prefix becomes tool-aware:
  `codex_<report>_report_<ts>.<fmt>` stays for `--tool codex`; `opencode_...` and
  `combined_...` for the others. Existing filenames are unchanged for the default.

### 6.3 Backwards compatibility

`--tool` defaults to `codex` everywhere. Every existing command line, JSON key, CSV column,
and export filename is unchanged. This is a strict superset.

---

## 7. Report design

### 7.1 opencode local report sections

Rendered with the existing `section()`, `explain()`, `print_counter_table()` helpers so it
looks native:

1. **Local report overview** — retrieved-at, data dir, DB path + size, network calls (0), privacy note.
2. **Highlights** — from `opencode_hints()`: busiest day, dominant model, share of tokens that
   are cache reads, subagent session share, total spend if non-zero.
3. **Session and message counts** — sessions, subagent sessions, assistant messages, sessions
   with token totals, date range.
4. **Approximate token totals** — the five `USAGE_FIELDS`, plus a cache-write row and a
   `Cost (reported)` row.
5. **Daily token totals, last N days** — Date / Sessions / Total / Output / Cache read.
6. **Tokens by model** — Model / Sessions / Messages / Total tokens / Share.
7. **Tokens by provider** — same shape, coarser.
8. **Tokens by agent** — Agent / Messages / Total tokens / Share. opencode-specific and
   genuinely useful: `build` vs `plan` vs `explore` vs subagents.
9. **Tokens by project** — Project (via `short_path()`) / Sessions / Total tokens / Share.
10. **Top sessions by total tokens** — Date / Model / Agent / Total / Output / Project /
    Session id (truncated). A `↳` marker prefixes subagent sessions.
11. **Notes** — local-only, no network calls; counters are provider-reported and not a bill;
    cache reads are counted separately and not folded into `total_tokens`.

### 7.2 Cache-efficiency line

opencode reports cache reads at a scale that dwarfs everything else (8.1 M cache-read tokens
against 222 K input on a single recent session). A single derived line is worth showing:

```
Cache read ratio    cache_read / (cache_read + input)
```

Presented as an efficiency indicator, explicitly *not* as a cost saving, since the script has
no pricing table (§10.4).

### 7.3 Combined view (`--tool all`)

A single extra table after both per-tool blocks:

| Tool | Sessions | Total tokens | Output | Cost | First seen | Last seen |
|---|---|---|---|---|---|---|
| Codex | … | … | … | — | … | … |
| opencode | … | … | … | $… | … | … |
| **Combined** | … | … | … | … | … | … |

With a prominent caveat line: **these totals are not directly comparable.** Codex figures come
from the last `total_token_usage` counter per session file; opencode figures are summed per
assistant message. They also may bill against different accounts. The table answers "where is
my time going", not "what do I owe".

---

## 8. Export schema

### 8.1 JSON

`--report local-usage --tool opencode` emits `collect_opencode_usage()` verbatim.
`--tool all` emits `{"codex_local_usage": {...}, "opencode_local_usage": {...}}`.
`limit_local_usage_days()` is applied to each independently (it already operates on
`data["sessions"]["daily_usage"]`, which both shapes have).

### 8.2 CSV

New `section` values in `rows_for_csv()` (`codex_usage.py:3034`), keeping the existing
"section column + union of keys" convention:

- `opencode_daily_usage` — date, sessions, five token fields, cost
- `opencode_model_usage` — model, provider, sessions, messages, token fields
- `opencode_agent_usage` — agent, messages, token fields
- `opencode_project_usage` — project, sessions, token fields
- `opencode_top_session` — session id, date, model, agent, project, token fields

Existing `daily_local_usage` and `sqlite_model_usage` sections keep their exact meaning
(Codex only), so any downstream parsing of past exports still works.

---

## 9. `doctor` and quick-summary integration

### 9.1 `doctor`

`collect_doctor()` (`codex_usage.py:1307`) gains an `"opencode"` block:

- `path_status()` for the data dir, `opencode.db`, `-wal`, `-shm`
- installed flag + version if `~/.opencode/bin/opencode` or `$PATH` resolves (`--version`,
  guarded by a short timeout; skip entirely if that adds a subprocess call the maintainers
  don't want — the DB's presence is sufficient)
- table presence check (`session`, `message`) and row counts
- `json1` availability probe: `SELECT json_extract('{"a":1}', '$.a')` — fail loudly and
  early with a clear message rather than producing silently-zero totals
- WAL size warning: a `-wal` file in the hundreds of MB (251 MB here) means a long-running
  opencode process hasn't checkpointed; totals are still correct but the read is slower
- legacy `storage/` note per §3.4

`print_doctor()` (`codex_usage.py:1370`) renders it as its own table, only when opencode is
detected, so Codex-only users see no new noise.

### 9.2 Quick summary

`collect_quick_summary()` (`codex_usage.py:2387`) is currently network-only (resets + online).
Add a cheap local opencode line — two aggregate queries, ~0.4 s — guarded so a missing or
locked DB never blocks the menu:

```
opencode today: 12 sessions, 3.09 M tokens
```

`quick_summary_lines()` appends it only when opencode data exists.

---

## 10. Edge cases and correctness rules

### 10.1 Reading a live, WAL-mode, 7 GB database

- Open `file:{path}?mode=ro` via `connect_sqlite_readonly()`-style URI. This **works while
  opencode is running** (verified).
- If that raises `sqlite3.OperationalError` — which happens when the `-shm` is absent and the
  directory isn't writable, so SQLite can't set up shared-memory for WAL recovery — retry with
  `immutable=1`, which reads the main DB file and **ignores the WAL**. Record
  `"stale_read": true` in the output and surface a note, because uncheckpointed recent
  sessions will be missing.
- Never `PRAGMA journal_mode`, never `VACUUM`, never open read-write. No fallback that copies
  the file: it is 7.5 GB.
- Set a `busy_timeout` of a few seconds so a concurrent checkpoint doesn't produce a hard
  failure.

### 10.2 What counts as `total_tokens`

opencode's own `tokens.total` field is unreliable for summing: on the sampled message,
`total: 214498` while `input: 401, output: 81, cache.read: 214016` — i.e. `total` includes
cache reads. Summing that across messages triple-counts a cache-heavy session and produces a
figure that looks alarming and means nothing.

**Rule: `total_tokens = input + output`.** Cache read, cache write, and reasoning are reported
in their own columns. This is documented in the report's Notes section, and is also the closest
match to how the Codex side reports (`total_tokens` there comes from Codex's own final counter,
which is a different convention — hence the §7.3 caveat).

### 10.3 Subagent (child) sessions

Include in all totals — verified in §3.3 that parents do not absorb children. Mark them in
per-session tables and expose `subagent_session_count`. A future `--exclude-subagents` flag is
possible but is not part of this plan.

### 10.4 Zero-cost rows

Only 4 of 761 sessions report a non-zero cost; total reported spend is $21.34. Subscription
providers (the OpenAI `gpt-5.x` family here) report `cost: 0`. The Cost column will therefore
read `$0.00` for nearly everything.

Do **not** silently substitute an estimate. Show reported cost as-is, labelled
`Cost (reported)`, with a note that subscription-billed providers report zero and that this is
not a bill. A pricing-table overlay is a plausible follow-up but would require bundling and
maintaining per-model prices — a maintenance burden inappropriate for a single-file tool.

### 10.5 Privacy

The script's stated guarantee is that it never prints prompts, replies, transcripts, or
secrets. `session.title` is a model-generated summary of the conversation and can leak content
("Fix auth bypass in payments service"). **Do not print titles.** Identify sessions by
truncated id and `short_path()`-ed directory, exactly as the Codex report identifies them by
relative session-file path. Working directories are already treated as printable by the
existing report; keep them behind `short_path()`.

The `message.data` blob contains message text in sibling `part` rows, not in `data` itself, and
the SQL only ever selects specific `json_extract` paths — no message body is read into memory.

### 10.6 Anomalous provider rows

Anthropic-labelled messages on this machine show 2,307 messages but only 6,463 input tokens —
i.e. the provider returned no usage data. Rows like this will appear as high-message /
near-zero-token entries. Add a hint when a model's messages/tokens ratio is degenerate:
"provider reported no token usage for N messages on <model>", so the numbers aren't read as a
bug in this tool.

### 10.7 Timezone and epoch units

opencode timestamps are epoch **milliseconds**. The existing `fmt_local_timestamp()`
(`codex_usage.py:187`) already auto-detects this — it divides by 1000 when the value exceeds
10,000,000,000 — so it can be reused directly for display with no change. The division is only
needed in SQL, where `date(ts_ms/1000, 'unixepoch', 'localtime')` must do it explicitly. Daily
grouping uses `'localtime'` so days align with the Codex side.

### 10.8 Empty / missing installation

`--tool opencode` with no DB present: print a single clear line ("No opencode database found at
<path>") and exit 0, not via `die()`. `--tool all` with no opencode: render the Codex report
plus that one note. `die()` remains reserved for the Codex-home-missing case it already covers.

---

## 11. Phased implementation

Each phase leaves the script working and shippable.

**Phase 1 — read path (no UI).**
`resolve_opencode_data_dir()`, `connect_opencode_readonly()`, `scan_opencode_messages()`,
`collect_opencode_usage()`. Validate against the §3.3 baseline numbers via
`local-usage --tool opencode --json`.

**Phase 2 — rendering.**
`print_opencode_usage()`, `opencode_hints()`, the `--tool` flag on `local-usage`.
Deliverable: `./codex_usage.py local-usage --tool opencode` produces a full report.

**Phase 3 — combined view.**
`--tool all` on `local-usage`, the combined totals table (§7.3), `collect_all()` /
`print_all()` integration.

**Phase 4 — export.**
`tool` threaded through `export_json()` / `rows_for_csv()` / `export_path()` / `cmd_export()`,
plus the new CSV sections. Verify all three formats for each of the three tool values.

**Phase 5 — doctor and menu.**
`opencode_health()` in `collect_doctor()` / `print_doctor()`; quick-summary line; a tool
selector in the menu settings screen (§6.1).

**Phase 6 — docs.**
`README.md`: new `--tool` flag, an opencode section, the comparability caveat, the privacy
statement extension, and a note that opencode support is local-only by design.

---

## 12. Testing

No test framework exists in the repo today; keep verification as runnable commands plus a small
optional self-check.

**Baseline assertions** (values from §3.3, this machine):

```
sessions == 761, messages(assistant) == 44446
sum(input_tokens) == 243_098_583
sum(cost) ≈ 21.3402
message-level sum == SUM(session.tokens_input)     # cross-check invariant
```

That last one is the important one and should ship as a runtime cross-check: compute both, and
if they diverge by more than a rounding tolerance, emit a hint that the session counters and
message counters disagree (indicating an opencode schema change). It is cheap and it turns a
future silent-wrong-numbers regression into a visible warning.

**Manual matrix:**

| command | expectation |
|---|---|
| `local-usage` (no flag) | byte-identical to pre-change output |
| `local-usage --tool opencode` | full report, no network |
| `local-usage --tool all` | both blocks + combined table |
| `local-usage --tool opencode --json` | valid JSON, `--days` trimming applied |
| `export --report local-usage --tool opencode --format {txt,json,csv}` | three files written |
| `doctor` | opencode block present, no crash |
| `doctor` with `OPENCODE_DATA=/nonexistent` | graceful "not found" |
| `local-usage --tool opencode` while opencode is running | succeeds against the live WAL |
| `--no-colour`, narrow terminal (`COLUMNS=60`) | tables degrade like existing ones |

**Timing budget:** the opencode report should stay under ~2 s. Measured aggregate query cost is
0.42 s over 48 k messages; the full report runs perhaps 6–8 such queries, so ~1–3 s. If it
exceeds that, collapse the rollups into a single pass over the CTE with conditional aggregates
rather than adding caching.

---

## 13. Risks

| risk | mitigation |
|---|---|
| opencode schema changes between versions (it has already migrated once, from `storage/` JSON to SQLite) | probe `PRAGMA table_info` before querying, as `sqlite_threads_summary()` already does for Codex; degrade to "unsupported opencode schema" rather than crashing |
| `json1` unavailable in an exotic Python build | explicit probe in `doctor` and a guarded fallback message |
| This repo is a fork of MacSteini/Codex-Usage; a large opencode addition will conflict with upstream | keep all new code in one contiguous, clearly-delimited block and touch shared functions minimally (dispatch only) so rebases stay tractable |
| Scope creep into pricing/cost estimation | explicitly out of scope, §10.4 |
| The tool becomes "a multi-tool usage dashboard" and duplicates [tokscale](https://github.com/junhoyeo/tokscale) | stop at opencode; if more tools are wanted, use tokscale |

---

## 14. Prior art

Reviewed; none is directly reusable (all TypeScript/Node or separate Python packages — nothing
single-file stdlib Python), but useful as reference:

- [junhoyeo/tokscale](https://github.com/junhoyeo/tokscale) — closest in spirit: one CLI across
  opencode, Claude Code, Codex, Gemini, Cursor and more. **If multi-tool coverage rather than
  opencode specifically is the goal, this already exists — check it before building.**
- [ramtinJ95/opencode-tokenscope](https://github.com/ramtinJ95/opencode-tokenscope) (MIT) —
  aggregates `step-finish` parts; good reference for cache-efficiency metrics and recursive
  subagent attribution.
- [Cateds/opencode-stats](https://github.com/Cateds/opencode-stats) — terminal dashboard,
  heatmap; display ideas.
- [xiello/opencode-usage](https://github.com/xiello/opencode-usage),
  [gaboe/opencode-usage](https://github.com/gaboe/opencode-usage) — grouping by
  agent/model/provider/session, which informed §7.1 items 6–9.
- [opgginc/opencode-bar](https://github.com/opgginc/opencode-bar),
  [tongsh6/opencode-token-tracker](https://github.com/tongsh6/opencode-token-tracker),
  [Shlomob/ocmonitor-share](https://github.com/Shlomob/ocmonitor-share) — plugin/real-time
  approaches; different problem shape.
