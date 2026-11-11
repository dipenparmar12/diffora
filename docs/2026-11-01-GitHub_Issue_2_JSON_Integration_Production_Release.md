# Issue 2: Integrate the Existing JSON Comparison Engine and Qualify the Application for Production

## Objective

Integrate the existing JSON comparison tool into the unified desktop application through an evidence-based adapter, then complete large-file hardening, cross-engine validation and Windows production release qualification.

This issue depends on the shared desktop architecture, worker lifecycle, canonical result contract and application services established in Issue 1. The JSON engine must preserve its existing comparison semantics. Do not rewrite or port it blindly.

The first usable release is a complete CSV workflow. JSON joins that workflow only when source audit, integration and parity gates demonstrate that the claimed capabilities work.

## Scope

### 1. Complete JSON source audit

Inspect the complete JSON comparison repository, not only screenshots or functional analysis.

Audit:
- Repository structure, version, entry points and dependencies.
- Runtime requirements and licenses.
- `json_align.js`, `key_compare.js` or equivalent modules.
- Collection discovery and identity-key detection.
- Matching algorithms, thresholds and move detection.
- Nested field comparison, tolerance and ignore rules.
- Repair, formatting, cleanup and sorting behavior.
- Result data structures and rendering dependencies.
- Tests, fixtures and known limitations.
- Browser globals, CodeMirror, DOM and UI-state dependencies.
- Parsing strategy, memory behavior, streaming support and largest tested inputs.
- Worker/sidecar feasibility, startup and shutdown behavior.

Produce:
- Repository and dependency inventory.
- Separation map for parsing, comparison, alignment, rendering and UI state.
- Result model and error contract.
- Capability matrix: existing, reusable, adaptable, unsupported or not verified.
- Versioned parity fixture corpus and known limitations.
- Explicit integration decision for each capability: extract, wrap, port, defer or exclude.

### 2. JSON integration decision gate

Choose the integration route based on source evidence:

1. If the headless comparison core is separable, extract or invoke it without CodeMirror, DOM, browser globals or UI lifecycle dependencies.
2. If it is coupled but can be separated with bounded work, isolate pure comparison functions and create parity fixtures first.
3. If it requires Node, keep the JavaScript engine in a headless Node sidecar or worker behind the shared adapter contract.
4. Port only if necessary, and only after fixtures capture existing behavior and automated parity tests compare old and new outputs.
5. If safe integration or parity cannot be demonstrated, defer the affected capability or JSON support from v1.

Do not port the engine merely to make the technology stack uniform.

### 3. JSON adapter and canonical result mapping

Implement the JSON adapter against the stable application contract delivered in Issue 1.

Preserve documented behavior only where confirmed by source audit and tests:
- Root arrays and supported collection envelopes such as `data`, `results` and `items`.
- Identity-key detection using uniqueness and overlap evidence.
- User-visible confirmation of detected identity keys.
- ID-based alignment.
- Exact LCS for small collections without an identity key, where supported.
- Hash-anchor/similarity fallback for larger collections, where supported.
- Record states such as same, changed, added, removed and moved.
- Nested object and array inspection.
- Field states including equal, changed, type mismatch, tolerance-equal, left-only, right-only and ignored.
- Field-level navigation and highlighting.
- By key and Per record views where supported.
- Search and status filtering.
- Ignore patterns and numeric tolerance.
- Formatting, minification, repair preview and cleanup only where source audit confirms them.

The adapter must:
- Declare supported statuses and capabilities.
- Map JSON records and field differences into the versioned canonical result contract.
- Preserve stable source locations only when the parser can provide them reliably.
- Distinguish missing properties from explicit `null`.
- Preserve original JSON values and types.
- Represent unsupported counts as unsupported/null, not zero.
- Never fabricate a match, move or modification.
- Keep comparison logic independent of Vue rendering and CodeMirror.

### 4. JSON large-input strategy

Do not assume the current browser-oriented implementation can process multi-gigabyte JSON files.

Evaluate and implement an appropriate bounded local strategy:
- NDJSON/JSON Lines: line-by-line parsing if supported.
- Large JSON arrays: incremental parsing using a verified parser such as `ijson` or another suitable implementation.
- Root objects: bounded parsing where possible.
- Nested arrays: streaming or disk-backed indexing by stable paths/identities.
- Embedded JSON strings: expand only when explicitly enabled and supported.
- Large values: preserve exact source or lossless parsed representation as required.

Document which JSON structures are genuinely streaming and which require materialization. Do not claim constant-memory behavior without measurements.

JSON large-file support is gated by:
- Source audit.
- Feasible parser and matching strategy.
- Parity fixtures.
- Memory and temporary-disk measurements.
- Repeatable benchmarks.

If a structure cannot be processed safely within supported resource limits, fail clearly or mark it unsupported. Do not silently fall back to whole-document frontend parsing.

### 5. Unified JSON user experience

JSON must be a first-class format under the shared application shell, but the interface must not be JSON/CodeMirror-centric.

Provide a structured tree/record-oriented workspace for verified capabilities:
- Format-specific controls only when relevant.
- Root/envelope and collection information.
- Identity-key suggestion with uniqueness evidence, overlap estimate, sample coverage and limitations.
- Explicit user confirmation of detected keys.
- Record alignment and move/reorder indicators where supported.
- Nested field differences.
- Details drawer and field histogram where supported.
- By key and Per record views where supported.
- Search, filters and difference navigation.
- Clear ambiguity and match-quality indicators.
- Bounded expansion and on-demand loading of large values.

Unsupported JSON capabilities must not appear as functioning controls. Keep shared toolbar actions format-neutral where possible. Do not expose internal engine names in ordinary UI.

### 6. Cross-engine correctness and fixture governance

Create a versioned JSON fixture corpus with reviewed expected outputs.

The JSON fixture matrix must cover, where supported:
- Root arrays and supported envelopes.
- Identity-key matching and identity detection.
- Reordering and moves.
- No-ID alignment and large-array fallback.
- Nested objects and arrays.
- Added/removed fields.
- Type mismatches.
- Missing versus explicit null.
- Tolerance and ignore rules.
- Malformed JSON and repair preview.
- Large-input parsing and resource behavior.

Compare old and integrated JSON outputs on the same fixtures. Every capability advertised in the UI must have source-backed status and fixture coverage. If parity is incomplete, hide the unsupported feature or mark it unavailable instead of approximating silently.

Use property-based tests only for guaranteed semantics. Fuzz corrupt/truncated JSON, escaping, unusual Unicode, deeply nested structures where supported and malformed options/patterns.

### 7. Large-file performance and resource qualification

The design workload target is approximately 5 GB per input and 10 GB combined. This is a design target, not a performance guarantee.

Benchmark representative workloads:
- 100 MB, 500 MB, 1 GB and 5 GB per side where practical.
- CSV unique-key, keyless, duplicate-heavy and wide-schema datasets.
- High and low difference rates.
- JSON arrays, nested structures and NDJSON only where supported by the audited strategy.
- Local SSD and network storage where relevant.
- Result-store growth under high difference rates.
- Cancellation and cleanup under resource pressure.

Record:
- CPU model and core count.
- Physical and available RAM.
- Storage type, free space and measured read/write throughput.
- OS build and application/engine versions.
- Input sizes, row/record counts, column counts and record widths.
- Difference and duplicate-key rates.
- Selected strategy and memory limit.
- Peak process memory.
- Peak temporary disk usage.
- Elapsed time, throughput and result repository growth.
- Correctness, cancellation time and cleanup outcome.

The proposed minimum qualification baseline is Windows 10/11 64-bit, 8 GB RAM and at least 20 GB free workspace, with local SSD recommended. This baseline must be ratified or revised using benchmark evidence before release. Do not publish performance claims without reproducible benchmark reports.

### 8. Temporary workspace, recovery and network storage hardening

Complete production hardening of the workspace and artifact lifecycle:
- Job-scoped directories for indexes, spill files, result stores, staged exports and manifests.
- Success cleanup of disposable intermediates.
- Cancellation cleanup of partial results unless resumable checkpoints are explicitly designed.
- Failure diagnostics with unsafe partial outputs removed.
- Startup detection of interrupted jobs.
- Validation of committed results before recovery.
- Cleanup or recovery of incomplete artifacts.
- Disk quotas and safety margin enforcement.
- Visible workspace path and disk consumption.
- Safe handling of network disconnects and read errors.

Network-mounted sources must never be modified. Optional local staging must be opt-in, show its disk cost, verify size/fingerprint before comparison and follow retention cleanup policy.

### 9. Security and privacy verification

Verify the complete application against the security requirements established in Issue 1:
- Narrow Tauri v2 capabilities and strict CSP.
- Bounded IPC payloads.
- No unrestricted frontend filesystem access.
- Path traversal and symlink/path confusion protection.
- Restrictive temporary workspace permissions.
- No source values or sensitive keys in logs by default.
- Regex ReDoS protection and adversarial tests.
- Delete all artifacts behavior.
- Retention and export sensitivity documentation.
- No cloud processing or telemetry containing source contents.

Redaction/masking remains a future optional export feature. It must never modify canonical comparison results.

### 10. Windows packaging and release qualification

Version 1 is Windows-only unless separate resourcing approves additional platforms. Do not imply macOS/Linux support merely because Tauri supports them.

Package the application so users do not need to manually install development tooling or runtimes. The final packaging must include the required Python runtime/sidecar, Python dependencies, JSON runtime/sidecar where selected, native dependencies and license notices.

Validate:
- Clean Windows machine installation and first launch.
- No developer environment required.
- No manual Python or Java installation unless the approved architecture explicitly documents it.
- Sidecar discovery from packaged locations.
- Worker protocol and version compatibility.
- Worker startup, health, crash, restart and shutdown.
- Native file dialogs and drag/drop.
- Paths containing spaces, Unicode and long path components.
- Restricted user permissions and non-admin operation.
- Code signing plan and SmartScreen/antivirus reputation considerations.
- Upgrade and uninstall behavior.
- Retained-session behavior across upgrades.
- Workspace permissions and cleanup.
- Installer and production build metadata.

### 11. Final end-to-end release validation

The production release must demonstrate:
- Both adapters run through documented application services.
- CSV and verified JSON workflows exist under one UI.
- Source files remain unchanged.
- Large inputs are not loaded into Vue memory.
- Key/identity selection and validation are explicit.
- Duplicate and ambiguous matching is visible.
- Progress and cancellation work for supported engines.
- Canonical results power summaries, browsing and exports.
- Pagination, filtering and detail retrieval are backend-driven.
- Supported By key and Per record workflows work.
- Exports stream to disk and are not published partially.
- Recovery and temporary cleanup are tested.
- Existing CSV and JSON fixtures pass.
- Performance benchmarks and limitations are documented.
- The Windows installer works on a clean supported machine.

## Acceptance criteria

### JSON audit and integration
- [ ] Complete JSON source tree and runtime dependencies are audited.
- [ ] Every advertised JSON capability has a source-backed status.
- [ ] A documented decision selects extract, wrap, port, defer or exclude for each capability.
- [ ] A versioned JSON parity fixture corpus exists.
- [ ] Integrated JSON output matches the existing tool for every claimed supported behavior.
- [ ] Unsupported capabilities are hidden or clearly marked unavailable.
- [ ] No large-file JSON claim is made without repeatable benchmark and memory evidence.

### JSON workflow
- [ ] JSON runs through the shared adapter and canonical result contract.
- [ ] Missing properties and explicit null values remain distinguishable.
- [ ] Identity suggestions require user confirmation.
- [ ] Source locations are shown only when reliable.
- [ ] JSON UI uses bounded result pages and on-demand details.
- [ ] No complete JSON document is passed through frontend IPC or rendered in full for large inputs.

### Performance and resource safety
- [ ] Benchmark reports include complete machine and workload metadata.
- [ ] High-difference-rate large-file workloads are included.
- [ ] Result repository growth and temporary disk peak are measured.
- [ ] Supported hardware baseline is ratified from evidence.
- [ ] No unsupported constant-memory or speed claims are published.
- [ ] Disk exhaustion, cancellation and cleanup are tested.

### Production readiness
- [ ] Interrupted jobs cannot be mistaken for completed jobs.
- [ ] Recovery validates committed results and handles partial artifacts safely.
- [ ] Network staging is opt-in and verified.
- [ ] Security, privacy and regex adversarial tests pass.
- [ ] Retention and deletion policies are documented and tested.
- [ ] Clean Windows installation and first launch succeed without developer tooling.
- [ ] Packaged sidecars are discovered and version-validated.
- [ ] Worker health and recovery tests pass in the packaged build.
- [ ] Code signing, SmartScreen and supported Windows versions are documented.
- [ ] Every P0 requirement has traceable acceptance evidence.

## Dependencies

This issue depends on Issue 1 for:
- Tauri/Vue application shell and native file workflow.
- Versioned worker protocol and managed worker lifecycle.
- Canonical application result contract.
- Job manager, progress and cancellation interfaces.
- Disk-backed result repository and bounded query interface.
- Shared export, session, retention and security services.

Shared contracts must be reviewed and stable before JSON integration relies on them. The JSON source audit can proceed in parallel with Issue 1 where repository access permits.

## Explicitly out of scope

- CSV-to-JSON or JSON-to-CSV semantic comparison.
- Cloud processing, hosted SaaS and multi-user collaboration.
- Full Git-like branch management.
- Redaction/masking of canonical results.
- Cross-platform release beyond Windows without separate approval.
- Guaranteeing completion time for every possible 5 GB dataset.

## Implementation expectations

Do not change JSON matching semantics without documenting the difference and updating the parity decision. Do not port before capturing baseline fixtures.

For every implementation task or PR, report:
- Requirement IDs implemented.
- Requirement IDs partially implemented or deferred.
- Decision IDs followed or changed.
- Files changed.
- Tests and results.
- Benchmark machine/workload and performance impact.
- Known limitations and evidence.
- Any behavior that differs from the existing engines.

No task is complete solely because the code compiles. Relevant acceptance criteria must pass.
