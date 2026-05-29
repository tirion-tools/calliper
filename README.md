# Calliper

> A fast, cross-platform SQL Server query plan analyzer in the spirit of SQL Sentry Plan Explorer, but native, lighter, and built on a fully open data format.

By **Tirion** · [tirion.tools](https://tirion.tools)

Calliper reads `.sqlplan`, `.queryplan`, `.pesession` (SentryOne / SolarWinds) and its own `.osession` container, then renders the plan tree, statement breakdown, wait stats, captured rowsets, and runtime profile in a single desktop app. It can also drive ambient XEvent captures and replay them side-by-side with prior runs to spot regressions.

The desktop binary is closed-source; the on-disk format and every parser it depends on are open and unencumbered. See [Open-source libraries](#open-source-libraries) below.

---

## Status

1.0. Format is stable. Subsequent releases will carry forward-compat readers for any breaking schema changes.

Tested platforms:

- Windows 10/11. Primary target; Inno Setup `.exe` installer.
- Linux (X11; Wayland under XWayland). GLFW 3.3+; `.deb` for Debian / Ubuntu.
- macOS. Native build planned.

---

## Features

### File formats

- **`.sqlplan` / `.queryplan`.** Raw ShowPlanXML, read direct.
- **`.pesession`.** SentryOne / SolarWinds Plan Explorer session container. Parsed clean-room from public Microsoft NRBF / ShowPlanXML specs; no decompilation of vendor binaries. Inner streams supported:
  - `<n>.queryanalysis` (NRBF: plans, TraceRowEx, QueryStats, waits)
  - `<n>.runtime` (per-statement aggregates)
  - ConnectionParameters / batch text / Index Analyzer JSON
- **`.osession`.** Calliper's native SQLite container. Same data shape as a pesession but indexed, compressed (deflate), and trivially diff-able. See [`libosession`](#libosession).

### Plan diagram

- Operator tree rendered with SQL Sentry-style stacked layout (cost %, rows, icon, op name, sub-type, object, index).
- Operator icons sourced from Microsoft's `vscode-mssql` repo (MIT-licensed), visually consistent with SSMS.
- Heat coloring by Total / CPU / IO / Rows.
- Pan (right-/middle-drag), zoom (wheel), fit-to-view, click-to-select.
- Hover tooltips: Seek Predicates (bulleted, one per column), Predicate, Warnings, Output List, Information.
- Long IN-list compression in Seek Predicates (`RangePartitionNew([…], (0), …[64 values]…, (2000000000))`).

### Statements grid

- One row per `sp_statement_completed` / `TraceRowEx` event, ordered in **call-tree pre-order** (parent EXEC dispatcher above its L+1 descendants, matching Plan Explorer's display order).
- Auto-deduped against `plan_id` for live captures (a sproc fired in a loop produces one row + N per-execution trace events, not N rows of the same plan).
- Hideable columns: Statement, Object, Est Cost %, Compile, Duration, UDF Duration, CPU, UDF CPU, Est CPU %, Reads, Writes, Est IO %, Est Rows, Actual Rows, Key Lookups, RID Lookups, Flags, Start, End. Sort by any column.
- Tree-mode collapse/expand on parent rows.
- **Scope to this subtree** (Ctrl+E or right-click): drills into one EXEC and renormalises Cost % so the scoped sproc reads 100%.
- Breadcrumb above the grid: `All › sproc1 › sproc2 (scoped, costs renormalised)`. Click any segment to pop back.
- Keyboard navigation: ↑/↓, PgUp/PgDn, Home/End.
- Mouse side-buttons (X1/X2) navigate the scope stack like a browser back/forward. Also responds to `ImGuiKey_AppBack` / `AppForward` for drivers that route those as keystrokes.
- Hover tooltip per row: full call chain `at: schema.name, Line: N, Nest Level: M, Start Offset: X, End Offset: Y`.

### Version compare

- Load any two captures into the same tab.
- Per-statement diff classification: Unchanged, CostShift (≥20% delta), ShapeChange (RelOp tree differs), Both, OnlyInA, **ResultDiff**.
- ResultDiff (saturated red) escalates the other states when the captured **output rows** differ. Semantic divergence is a stronger signal than the plan that produced it.
- Result-row equality uses an FNV-1a 64-bit hash computed at load time, so the diff stays sub-millisecond on thousand-row rowsets.
- Per-(item × compare) diff cache invalidates on item reload or compare swap, so the diff doesn't recompute every frame.

### Live capture

- Server-side XEvent session (auto-named, ring-buffer target): `sp_statement_completed`, `sql_batch_completed`, `rpc_completed`, `query_post_execution_showplan`, `wait_info`.
- Streams directly into a `.osession` tab. The Statements grid auto-refreshes at 750 ms; the status row carries the live counters (`● LIVE   snapshots: N   events: M   waits: W`).
- Toolbar Stop button doubles as the live-capture stop. After stop: Ctrl+S to save, or Discard from the same toolbar row.
- Captured statements deduped by plan_id; per-execution metrics aggregate into one statement row via `trace_events`.

### Script-tab capture

- Editable Command Text tab, bound to a SQL Server connection.
- **Get Estimated Plan** (SHOWPLAN_XML). Server-side restriction worked around by sending the SET as its own batch.
- **Get Actual Plan** (STATISTICS XML). Every run creates a new History version. Rowset capture is opt-in.
- Optional add-ons (server-version gated):
  - **Wait Stats**: diffs `sys.dm_exec_session_wait_stats` around the run (SQL Server 2016+).
  - **Live Query Statistics**: sibling connection polls `sys.dm_exec_query_profiles` every 250 ms during execution (SQL Server 2014+).
- F5 binds to Get Actual Plan.

### Wait stats panel

- Batch-level diff from `sys.dm_exec_session_wait_stats`.
- Sorted by total duration; signal-wait percent column.

### Live Query Statistics panel

- Operator-level row/percent-complete heatmap from `sys.dm_exec_query_profiles`.
- Updates in-place from the sibling poller's last snapshot.

### Captured result rows

- "Include query results" option on Get Actual Plan or live capture stores every emitted row-set alongside the plan.
- Result rows are diff-able across versions (see ResultDiff above).
- Displayed in the Results tab per statement.

### Plan XML & Index Analyzer

- Plan XML tab shows the original ShowPlanXML for the selected statement, no normalisation applied.
- Index Analyzer reads the gzipped JSON pesession carries and renders missing-index recommendations alongside SQL Server's own `<MissingIndexes>` block.

### Other

- **HiDPI auto-scale** via `glfwGetMonitorContentScale`.
- **Font size control** (Ctrl+= / Ctrl+- / Ctrl+0; View → Font size).
- **Dark / Light themes**.
- **Connection manager** with recent-connection history. Opt-in "Remember password" stores credentials in the OS-native keychain (Windows Credential Manager, Linux libsecret / GNOME Keyring).
- **Plan-shape dedup** in `.osession`. Re-emissions of the same plan share a single row in the `plans` table keyed by FNV-1a of the normalised XML (RuntimeInformation / QueryTimeStats / WaitStats / MemoryGrantInfo stripped before hashing). Original XML kept verbatim in `plan_snapshots` for lossless round-trip.

---

## Quick start

```bash
# Open a saved plan
calliper path/to/plan.sqlplan

# Open a SentryOne capture
calliper path/to/capture.pesession

# Open a Calliper osession
calliper path/to/run.osession

# CLI conversion: .pesession → .osession (lossless verified)
pesession2osession in.pesession out.osession
verify_lossless    in.pesession out.osession
```

Live capture and script execution are GUI-only. Start `calliper` with no argument, then click **Live Capture…** or **+ New Script**.

---

## Open-source libraries

The format and every parser Calliper uses are open source under permissive licenses (MIT), maintained by Tirion at [github.com/tirion-tools](https://github.com/tirion-tools). Linking from a separate project is explicitly supported.

| Library | Purpose | License |
|---|---|---|
| [`libshowplan`](https://github.com/tirion-tools/libshowplan) | SQL Server ShowPlanXML parser. RelOp tree, runtime stats, missing indexes, warnings, parameters, statistics. pugixml-based. | MIT |
| [`libnrbf`](https://github.com/tirion-tools/libnrbf) | .NET Binary Remoting Format (NRBF / MS-NRBP) reader. Visitor API, forward-ref string dictionary, handles the lpstr varint length convention. | MIT |
| [`libpesession`](https://github.com/tirion-tools/libpesession) | SentryOne / SolarWinds `.pesession` reader. Walks the NRBF tree, decodes queryanalysis / runtime / TraceRowEx streams, and converts to osession. | MIT |
| [`libxesession`](https://github.com/tirion-tools/libxesession) | SQL Server Extended Events session reader (`.xel` / ring-buffer XML). Used by live capture to ingest server-side events. | MIT |
| [`libosession`](https://github.com/tirion-tools/libosession) | SQLite container format: plans, plan_snapshots, statements, runtime, trace_streams, waits, objects, statement_results, items. Includes deflate-compressed trace-event batches and the schema docs. | MIT |
| [`libliveconnect`](https://github.com/tirion-tools/libliveconnect) | ODBC connector wrapping nanodbc for the live-capture + script-execution paths. Archived: folded into the Calliper app once it shrank to a thin nanodbc wrapper. | MIT (archived) |

Library READMEs cover the on-disk layout in detail. The osession schema is stable enough that any consumer (Python, Go, .NET) can read a Calliper capture by just opening it as SQLite.

The Calliper desktop binary is closed source. Source-license inquiries: <contact@tirion.tools>.

---

## Roadmap

Tracked in [Issues](https://github.com/tirion-tools/calliper/issues). Notable in-flight work:

- **macOS native build**: app bundle, Keychain integration, signed + notarized .dmg.
- **Curated `clang-tidy` adoption** across all repos.

---

## Building from source

Calliper's source isn't public, but the open-source libraries (all under Tirion at [github.com/tirion-tools](https://github.com/tirion-tools)) build standalone:

```bash
git clone https://github.com/tirion-tools/libshowplan
cmake -B build libshowplan && cmake --build build
ctest --test-dir build
```

Each library is dependency-light. pugixml is vendored, `libosession` vendors miniz and depends only on `sqlite3` from the system.

---

## Reporting issues

File against the relevant repo:

- Plan-parsing weirdness → [`libshowplan`](https://github.com/tirion-tools/libshowplan)
- pesession parsing → [`libnrbf`](https://github.com/tirion-tools/libnrbf) + the Calliper repo
- osession content / schema → [`libosession`](https://github.com/tirion-tools/libosession)
- UI / capture / live features → [`calliper`](https://github.com/tirion-tools/calliper)

Attach the source `.sqlplan` / `.osession` when filing. Calliper writes everything needed for a repro into the osession.

---

## Acknowledgments

- **Microsoft.** Public ShowPlanXML schema, the `vscode-mssql` operator-icon set, and the NRBP / NRBF format docs.
- **SolarWinds / SentryOne.** Plan Explorer set the UX bar. Calliper's pesession reader is built clean-room from public Microsoft specs; no SentryOne binary is decompiled or referenced. Plan Explorer's EULA restricts reverse engineering of its redistributables and Calliper respects that.
- **Dear ImGui**, **GLFW**, **pugixml**, **miniz**, **SQLite**, **nlohmann/json.** The libraries that do the heavy lifting.

---

## License

The closed-source desktop binary ships under a commercial license. Open-source libraries are MIT (see each repo's `LICENSE`).
