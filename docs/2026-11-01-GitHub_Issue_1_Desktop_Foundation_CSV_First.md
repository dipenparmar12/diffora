# Issue 1: Build the Desktop Application Foundation and Complete the CSV-First Vertical Slice

## Objective

Build the first usable release of the unified CSV and JSON comparison desktop application by delivering the shared desktop foundation and a complete, production-quality CSV comparison workflow.

This issue is intentionally CSV-first. The existing Python CSV comparison module remains the authoritative comparison engine. The application must wrap it with a reliable desktop workflow rather than rewrite its comparison logic.

The JSON source audit may proceed in parallel, but JSON implementation is tracked separately in Issue 2.

## Product and architecture baseline

- Desktop shell: Tauri v2.
- Frontend: Vue 3, preferably TypeScript, following the repository's established conventions.
- CSV engine: existing Python `compare_csv(...)` implementation.
- Local processing: managed Python worker/sidecar.
- Persistence: SQLite or another justified local disk-backed store for job metadata and canonical results.
- Source files are processed locally and opened read-only.
- The frontend receives metadata, progress, bounded result pages and selected record details only. It must never receive complete multi-gigabyte files or complete result collections.
- Do not introduce DuckDB, Polars or another replacement comparison engine without reproducible benchmark evidence and an explicit decision.

## Scope

### 1. Repository and existing CSV engine audit

- Inspect the current desktop repository, build system, dependencies and conventions.
- Inspect the complete CSV module and its tests.
- Document the public API, options, outputs, side effects, progress callback, cancellation behavior, storage strategies and export behavior.
- Capture baseline fixture outputs before integration.
- Record verified behavior, assumptions, limitations and any mismatch between the existing module and this specification.
- Preserve backward compatibility for existing callers.

### 2. Desktop shell and native file workflow

- Set up or adapt the Tauri desktop application.
- Provide native file selection for Left/Baseline and Right/Candidate.
- Support drag-and-drop path selection.
- Show file name, size, format, modification time, validation state and path on demand.
- Provide replace and clear actions.
- Detect same-file selection, missing files, access errors and changing files.
- Detect CSV/TSV format, encoding and delimiter where possible, with explicit overrides when detection is ambiguous.
- Pass validated filesystem paths to the local processing layer. Never read complete files into JavaScript strings, blobs or base64.
- Open source files read-only. Never modify, rename, move, delete or overwrite source files.
- Do not create permanent duplicate source copies.

### 3. Application service and adapter architecture

Implement clear boundaries for:

- Input inspection and validation.
- Comparison plan validation.
- Engine routing and adapter invocation.
- Job lifecycle and progress normalization.
- Cancellation.
- Canonical result persistence and querying.
- Filtering, sorting and pagination.
- Export coordination.
- Workspace cleanup and recovery.

The CSV adapter must call the existing `compare_csv(...)` API. Do not duplicate its matching, canonicalization, tolerance, storage-selection or export logic in the service layer.

The adapter must:
- Translate UI options into supported engine arguments.
- Pass explicit destinations for every requested export.
- Never rely on implicit current-directory output paths or default report generation.
- Disable or omit HTML output unless explicitly requested with a selected destination.
- Adapt the existing progress callback and cancellation behavior.
- Preserve the original engine error code in diagnostic metadata.
- Map verified CSV errors to stable application error codes.
- Preserve existing behavior for callers that do not use the new result sink.

### 4. Canonical result contract and disk-backed CSV result repository

The result repository is a Phase 1 P0 deliverable. It must be implemented during the CSV vertical slice, not postponed as a later optimization.

Extend the CSV module backward-compatibly with a result sink or equivalent callback that receives classified records during the existing comparison pass. Do not reconstruct authoritative results by parsing exported CSV files after comparison.

The repository must support the following logical data:

- Job metadata, engine/version, input metadata, options snapshot, timestamps, state and errors.
- Stable `record_ref` identifiers scoped to a job.
- Record status, serialized or typed key, duplicate group, match quality, ambiguity flag and left/right source row references.
- Field differences, field paths, statuses, values and type metadata.
- Source locations for both input sides.
- Summary counts, field counts, duplicate and ambiguity counts, warnings and column statistics.
- Export metadata, status, bytes written and commit state.

Preserve actual source values for detail inspection. Large values may use a bounded or blob-backed representation, but truncation must be explicit and must not silently change exported values.

Canonical matching concepts:
- `match_quality`: `exact`, `ambiguous` or `unmatched`.
- `duplicate_group_id`: identifies records sharing a duplicate comparison key.
- `ambiguous`: explicit boolean.
- Stable record references must not depend on the current UI page.

Preserve current CSV duplicate semantics:
- `match`: match identical records inside duplicate-key groups first, then pair remaining records in file order and mark residual pairs ambiguous.
- Residual duplicate pairing is order-dependent. Do not imply reorder invariance.
- `error`: reject ambiguous duplicate-key comparisons.
- Do not introduce Cartesian multiplication from duplicate keys.
- Keyless mode is a multiset comparison and must not claim modified-record detection.

Repository operations:
- Fetch job summary.
- Fetch cursor/keyset pages of records.
- Fetch a record and its field differences.
- Filter by status, key, field path, ambiguity and match quality.
- Search keys through an indexed strategy.
- Sort only by supported indexed or bounded fields.
- Stream selected results to export writers.
- Delete all artifacts for a job.
- Validate repository integrity after recovery.

Use stable keyset/cursor pagination. Avoid large OFFSET queries. Default UI page size should be 50 or 100. Keep frontend memory bounded.

### 5. Comparison configuration and mandatory preflight

Provide configuration for:
- Keyed and keyless comparison.
- One or more key columns, including composite keys where supported.
- Suggested keys with evidence and explicit user confirmation.
- Duplicate-key policy.
- Null-key policy.
- Column selection, ignored columns and renamed-column mappings.
- Trim, case normalization, numeric normalization, NA tokens, boolean and date normalization.
- Absolute, relative and per-column tolerance.
- CSV delimiter and encoding overrides.
- Strict CSV structure validation.
- Memory budget, temporary workspace and strategy selection.

Preserve exact comparison by default and preserve Decimal precision. Do not silently convert arbitrary numeric values to binary floating point.

CSV null-key policy defaults to `error`. Make this visible before comparison and identify null-key risks during preflight.

Preflight is mandatory after configuration and before the full comparison. It must report:
- File readability, format, encoding and delimiter.
- File sizes and fingerprinting status.
- Header/schema compatibility.
- Selected strategy and relevant engine options.
- Workspace free space, configured quotas and estimated temporary disk range.
- Memory limit.
- Suggested key and confirmation state.
- Null-key and duplicate-key risks where discoverable.
- Warnings requiring acknowledgment.

Clearly label estimates and sample-based statistics. Do not present estimates as guarantees or silently run an expensive full scan without identifying it.

Disk policy:
- Soft quota warns and requires acknowledgment where permitted.
- Hard quota prevents work that would exceed the configured limit.
- Reserve safety space for database journals/WAL and clean shutdown.
- If space falls below the safety margin, stop safely, preserve diagnostics, discard uncommitted artifacts and report `disk_space`.

### 6. Job lifecycle, worker protocol and progress

Implement a managed local worker. A spawned process is not considered healthy merely because it exists.

Version the worker protocol and include protocol version, worker version, request ID, job ID, message type, sequence/timestamp and payload.

Support:
- `hello`
- `health_check`
- `start_job`
- `cancel_job`
- `job_progress`
- `job_completed`
- `job_failed`
- `shutdown`

Large source contents and result sets must never travel through IPC. Pass validated paths and small configuration objects only.

Worker requirements:
- Versioned startup handshake with timeout.
- Periodic heartbeat during active work.
- Missed-heartbeat/stall detection.
- Health response containing worker state, active job, last progress sequence, heartbeat and protocol version.
- Record exit code/signal and filtered diagnostics after unexpected exit.
- Do not automatically replay an interrupted comparison.
- Mark worker-death jobs `interrupted`, never `completed`.
- Graceful shutdown followed by bounded wait and forced termination when necessary.

Job states must cover creation, inspection, validation, queueing, running, finalization, completion, cancellation, failure and interruption/recovery.

Progress must be truthful:
- Show phase and absolute counters when available.
- Show percentage only when the denominator is known and representative.
- Show ETA only when meaningful.
- Unknown values remain unknown, not guessed.
- Distinguish input scanning, matching, repository persistence, finalization and export.
- Avoid excessive IPC events or Vue rendering.

Cancellation must be cooperative across inspection, indexing, matching, result persistence, finalization and export. Do not present partial results as completed results.

### 7. Results and comparison workspace

Build a consistent, format-neutral application shell with a structured CSV table-oriented workspace.

Include:
- Summary metrics derived from canonical repository data.
- Added, removed, modified, unchanged, duplicate and ambiguous counts where supported.
- Virtualized result grid/list.
- Column, row, key and source-row context.
- Search, status filters and supported sorting.
- Previous/next difference navigation.
- Details drawer with field histogram and affected-item navigation.
- By key view.
- Per record view.
- Record detail with left/right values and field differences.
- Explicit ambiguity and match-quality indicators.
- Bounded value display with load-more/view-full behavior for large values.
- Clear loading, empty, invalid, warning, running, cancelling, cancelled, completed, failed and result-unavailable states.

All summary metrics, filters, details and exports must derive from the same canonical result model. Do not calculate authoritative counts from the currently visible page.

The frontend must not render a multi-gigabyte source file in CodeMirror or load all results into memory.

### 8. Export workflow

Support the existing CSV module's streamed outputs where applicable:
- Added records.
- Removed records.
- Modified records.
- Duplicate keys.
- Validation errors.
- Column statistics.
- Summary JSON.
- Optional HTML report.

Also provide unified delta/schema audit exports where practical.

Export requirements:
- Stream from the result repository to disk.
- Use a native save dialog.
- Support all differences or the current filtered subset.
- Show progress and cancellation.
- Stage output in a job-scoped temporary path.
- Flush, close and validate before commit.
- Commit atomically where supported.
- Preserve existing destination files if commit fails.
- Never overwrite either input.
- Confirm destination overwrite.
- Remove or mark partial output after failure.
- Do not publish partial exports after cancellation.
- Keep frontend memory bounded.

### 9. Sessions, profiles and retention

Distinguish:
- Job: one comparison execution.
- Session: retained user-facing workspace and results for a job.
- Profile: reusable comparison options.

Provide:
- Recent comparisons.
- Save/reopen sessions.
- Saved comparison profiles.
- Configurable retention period or quota-based cleanup with a documented default.
- Keep results and Delete after export choices.
- Per-job Delete all artifacts action.
- Separate retention behavior for metadata, result records and exports.
- Source fingerprint checks when reopening sessions.

If a source has changed or is unavailable, warn that the retained result represents the original source version, but allow inspection of the stored result. Never present it as a fresh comparison.

### 10. Security and privacy

- All processing remains local.
- Use narrowly scoped Tauri v2 capabilities.
- Do not grant broad filesystem access to the frontend.
- Apply a strict Content Security Policy.
- Set and enforce IPC payload limits.
- Validate paths at the native/service boundary.
- Prevent path traversal and relevant symlink/path confusion.
- Use restrictive workspace permissions.
- Do not log source values, full keys, tokens or sensitive paths by default.
- Protect user-defined regex patterns against ReDoS with limits, isolation/timeouts where supported and adversarial tests.
- Provide Delete all artifacts and report failures.
- Document that exported files are outside application control after saving.
- Redaction/masking is deferred and must never silently alter canonical results.

### 11. Network-mounted files

Support accessible network-mounted files to the extent the OS and engine can read them reliably.

An optional local staging mode may be provided, but it must be opt-in. Show its disk cost, verify copied size/fingerprint, never modify the network source and clean staged copies according to retention policy.

### 12. Testing and quality

Create versioned fixtures and reviewed expected outputs.

CSV regression coverage must include:
- Identical, empty, header-only and single-record files.
- Missing/malformed headers and differing schemas.
- Column reordering and very wide CSVs.
- Quoted delimiters, embedded newlines and encoding cases.
- Null keys under each supported policy.
- Duplicate keys with identical/conflicting records.
- Keyless multiset comparison.
- High add/remove/modify rates.
- Decimal precision and all tolerance modes.
- Ignored, renamed and selected columns.
- Input changes during comparison.
- Output collisions.
- Disk exhaustion and storage failure.
- Cancellation during indexing, matching, result persistence and export.

Add property-based tests only for guaranteed semantics, plus fuzz tests for corrupt/truncated input, quoting, escaping, Unicode and malformed options.

Test native dialogs, drag/drop, path validation, progress, cancellation, worker crash, recovery, pagination, filters, details, exports, retention, themes and accessibility.

## Acceptance criteria

### Desktop and source safety
- [ ] Application launches using the agreed development workflow.
- [ ] Native file selection and drag/drop work for both sides.
- [ ] Paths with spaces and Unicode are handled.
- [ ] Source files are read-only and remain unchanged.
- [ ] No complete source file is passed through frontend memory or IPC.

### CSV correctness and repository
- [ ] Existing `compare_csv(...)` remains the comparison authority.
- [ ] Existing callers remain backward-compatible without the result sink.
- [ ] Classified records are written to the canonical repository during the comparison pass.
- [ ] Summary counts reconcile exactly with engine summary counts.
- [ ] UI pages and details come from the repository, not parsed exports.
- [ ] Millions of results can be paginated and filtered with bounded frontend memory.
- [ ] Duplicate and ambiguous behavior is accurately represented.
- [ ] Keyless mode never claims modified-record detection.

### Preflight and resource safety
- [ ] Every comparison passes through preflight.
- [ ] Strategy, workspace estimate, free space and quotas are shown.
- [ ] Suggested keys require confirmation.
- [ ] CSV null-key default is visibly `error`.
- [ ] Disk exhaustion stops safely and records `disk_space`.
- [ ] Estimates disclose confidence and limitations.

### Jobs, cancellation and exports
- [ ] Worker handshake, heartbeat, health and crash handling are tested.
- [ ] Cancelled/interrupted jobs cannot appear completed.
- [ ] Cancellation during export publishes no partial final file.
- [ ] Existing destination files remain intact if commit fails.
- [ ] Temporary artifacts are cleaned or marked for recovery.

### Security and retention
- [ ] Tauri capabilities, CSP and IPC limits are documented and tested.
- [ ] Frontend cannot request unrestricted filesystem reads.
- [ ] Regex adversarial tests pass.
- [ ] Logs do not contain source values by default.
- [ ] Delete all artifacts works and reports failures.
- [ ] Retention and source fingerprint mismatch behavior are implemented.

### UI and QA
- [ ] Summary, details, filters and exports use the canonical result model.
- [ ] CSV workspace is table-oriented and does not require code-editor concepts.
- [ ] Keyboard accessibility and light/dark/system themes are supported.
- [ ] P0 CSV regression, cancellation, disk-failure and fuzz tests pass.

## Explicitly out of scope for this issue

- JSON engine integration and JSON parity work. Tracked in Issue 2.
- CSV-to-JSON or JSON-to-CSV semantic comparison.
- Cloud processing, hosted SaaS and multi-user collaboration.
- Full Git-like branch management.
- Redaction/masking of canonical results.
- Replacing the existing CSV engine without benchmark evidence.

## Implementation expectations

Before implementation, inspect the actual repository and report the source inventory, baseline behavior and gaps. Do not silently change existing engine semantics.

For each implementation task or PR, report:
- Requirement IDs implemented.
- Requirement IDs partially implemented or deferred.
- Files changed.
- Existing engine behavior reused and new behavior introduced.
- Tests and results.
- Performance impact.
- Known limitations and remaining decisions.

No task is complete solely because the code compiles. Relevant acceptance criteria must pass.
