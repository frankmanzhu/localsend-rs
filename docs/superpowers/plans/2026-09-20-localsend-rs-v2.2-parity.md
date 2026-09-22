# LocalSend-RS v2.2 Parity Implementation Plan

**Design:**
`docs/superpowers/specs/2026-09-20-localsend-rs-v2.2-parity-design.md`  
**Method:** test-driven, vertical slices, one behavior change per reviewable commit  
**Target release:** `0.2.0`, after a `v0.1.3` v2.1 baseline checkpoint

## Global constraints

- Write the failing unit/conformance/interop test before changing behavior.
- Do not hold `ServerState`'s write guard across network I/O, file I/O, or accept decisions.
- Upload remains one whole-file POST; do not add range/chunk protocol extensions.
- Tokens remain random and per file.
- Receive through a held safe destination and publish atomically only after validation.
- TLS code stays behind `https`; `cargo build --no-default-features --lib` must remain green.
- Preserve unrelated user changes and existing CrossCopy protected-upload integration.
- The pinned official oracle is an executable reference, not copied production code.
- End each task with focused tests; end each phase with the full local gate.

Full local gate:

```bash
cargo test
cargo clippy --all-targets --all-features -- -D warnings
cargo fmt --check
cargo build --no-default-features --lib
cargo build --features https
cargo build --features tui
```

---

## Phase 0 — Freeze the baseline and make e2e standalone

### Task 0.1: Record the oracle and current behavior

**Create:**

- `e2e/oracle.lock`
- `tests/oracle_contract.rs`

**Modify:**

- `README.md`
- `AGENTS.md`

- [ ] Add the official repository URL, commit
  `230fb692962668ca22ce0e61a8f53ce1cfd32102`, app/CLI version `1.18.2`, protocol `2.2`,
  architecture, and artifact checksum fields to `oracle.lock`.
- [ ] Add ignored characterization tests that can compare captured official DTO fixtures without
  requiring Docker. Store small hand-authored JSON fixtures, not binaries or copied source.
- [ ] Correct the status docs: reverse-download hosting exists; reverse-download client/CLI do not.
- [ ] Record the baseline command output in the release notes or PR, not as generated files.

**Verify:** full local gate.

### Task 0.2: Replace the stale standalone Docker build

**Modify:**

- `e2e/docker/localsend-rs.Dockerfile`
- `e2e/docker-compose.yml` (rename to `compose.yaml` if done in one atomic change)
- `e2e/run.sh`
- `e2e/scripts/receiver.sh`
- `e2e/scripts/sender.sh`
- `.dockerignore`

- [ ] First add a shell assertion showing the current Docker build fails when this repository is the
  build context; it currently expects `vendors/localsend-rs` and CrossCopy sibling packages.
- [ ] Build the public CLI from this repository root using cached dependency layers.
- [ ] Keep the runtime image minimal and non-root where file permissions allow.
- [ ] Replace random payload generation with a deterministic fixture manifest and SHA-256.
- [ ] Add hard readiness and scenario timeouts; always collect logs and clean volumes.
- [ ] Make `./e2e/run.sh smoke` pass without a CrossCopy checkout.

**Verify:**

```bash
./e2e/run.sh smoke
```

### Task 0.3: Cut the v2.1 checkpoint

This is a release operation, not an automatic test action.

- [ ] Inspect `cargo package --list` and exclude `.agents`, `.claude`, `.cursor`, `.journal`,
  `.DS_Store`, Docker outputs, and other private development material.
- [ ] Run `cargo package --allow-dirty` only for inspection; run the final package from a clean tree.
- [ ] Confirm downstream `zmanager-localsend` and `zmanager-ffi` against the exact commit.
- [ ] Update package version from `0.1.2` to `0.1.3` if this checkpoint will be published.
- [ ] Create annotated tag `v0.1.3` only after review and explicit release approval.

**Exit criterion:** a reproducible v2.1 rollback point exists before public API changes.

---

## Phase 1 — Land the narrow v2.2 wire delta

### Task 1.1: Checksum mismatch is 422 and retryable

**Modify:**

- `tests/conformance_upload.rs`
- `src/server/handlers.rs`
- `src/error.rs`
- `src/client/client.rs`

- [ ] Change the existing hash-mismatch conformance expectation from `500` to a failing `422`
  assertion.
- [ ] Extend the test: assert no final or temporary file exists and the session remains open.
- [ ] Retry the same `sessionId`, `fileId`, and token with the declared bytes; assert `200` and exact
  SHA-256.
- [ ] Return `UNPROCESSABLE_ENTITY` only for checksum mismatch. Keep size/I/O failures separately
  classified.
- [ ] Map `422` to a typed `ChecksumMismatch` client error.

**Verify:**

```bash
cargo test --test conformance_upload sha256_mismatch
cargo test --test interop_upload
```

### Task 1.2: Advertise protocol 2.2

**Modify:**

- `src/protocol/constants.rs`
- `src/protocol/validation.rs`
- `tests/unit_schema.rs`
- discovery/conformance tests that assert the version

- [ ] Add failing tests for `2.2` in info, register, prepare, and announcements.
- [ ] Change the constant only after Task 1.1 is green.
- [ ] Keep major-version-compatible input behavior and add `2.1` peer interop coverage.

**Verify:** full local gate.

### Task 1.3: Correct prepare-upload edge behavior

**Modify:**

- `tests/conformance_prepare_upload.rs`
- `tests/interop_message.rs`
- `src/server/handlers.rs`

- [ ] Change empty file-map behavior to official v2.2 `400`.
- [ ] Preserve text-preview `204` and its `TextReceived` event.
- [ ] Test mixed preview/file offers, partial acceptance, timeout, and aborted HTTP request slot
  cleanup.
- [ ] Compare the cases against the pinned official-core container when Phase 7 exists.

---

## Phase 2 — Separate wire DTOs from runtime peers

### Task 2.1: Introduce exact endpoint DTOs

**Create:**

- `src/protocol/dto_v2.rs`

**Modify:**

- `src/protocol/mod.rs`
- `src/protocol/types.rs`
- `tests/unit_schema.rs`
- `tests/conformance_prepare_upload.rs`

- [ ] Add golden serialization tests for register request, register response, info response,
  prepare-upload request/response, and prepare-download response.
- [ ] Prove response DTOs omit `port`, `protocol`, and `ip`; prove request DTOs include required
  endpoint fields.
- [ ] Preserve lenient deserialization for missing optional fields and unknown device types.
- [ ] Introduce explicit conversions between DTOs and domain types.
- [ ] Stop serializing the broad runtime `DeviceInfo` directly on endpoint responses.

### Task 2.2: Introduce endpoint/channel state

**Modify/Create:**

- `src/core/device.rs`
- `src/protocol/types.rs`
- `src/client/client.rs`
- `src/discovery/*.rs`
- public re-exports in `src/lib.rs` and `src/prelude.rs`

- [ ] Write unit tests proving a peer can own multiple HTTP channels and no wire response invents an
  address.
- [ ] Add `Peer`/`HttpEndpoint` domain types or equivalent.
- [ ] Provide migration conversions for callers that still construct `DeviceInfo` targets.
- [ ] Document the `0.2.0` source break before removing any old constructor.

**Exit criterion:** raw response JSON matches official fixtures while existing upload interop remains
green.

---

## Phase 3 — Harden and complete the client transport

### Task 3.1: Central URL builder

**Create:**

- `src/client/url.rs`

**Modify:**

- `src/client/client.rs`

- [ ] Test IPv4, DNS, IPv6, and scoped IPv6 host rendering.
- [ ] Test percent-encoding for PIN, session ID, file ID, and token query values.
- [ ] Replace string formatting in every client endpoint with the builder.
- [ ] Disable redirects and ambient proxies for every LocalSend transport client.

### Task 3.2: Per-target TLS pinning and client identity

**Modify:**

- `src/client/client.rs`
- `src/client/trust_policy.rs`
- `tests/interop_tls.rs`

- [ ] Add a failing two-server test: reuse one client for peer A then peer B and prove B is checked
  against B, not A's cached TLS client.
- [ ] Key cached pinned clients by normalized endpoint plus expected fingerprint, or make a client
  single-peer by construction.
- [ ] Assert a mismatch sends zero application payload bytes.
- [ ] Assert client certificate presentation works against a listener that requests it.
- [ ] Add request/connect/decision timeouts with typed errors.

### Task 3.3: Cancellation-aware low-level methods

**Modify:**

- `src/client/client.rs`
- `src/error.rs`
- `tests/interop_upload.rs`
- `tests/conformance_reservation.rs`

- [ ] Add cancellation tests for waiting prepare-upload and streaming upload.
- [ ] Accept a `CancellationToken` (or options containing one) without removing simple convenience
  methods.
- [ ] On cancellation after session creation, best-effort POST `/cancel` and return `Cancelled`.
- [ ] Assert the receiver releases its pending/active slot and deletes partial bytes.

### Task 3.4: High-level send transaction

**Create:**

- `src/client/transfer.rs`
- `tests/interop_send_transaction.rs`

**Modify:**

- `src/core/file.rs`
- `src/cli/commands/send.rs`

- [ ] Test registration, partial acceptance, bounded parallel upload, aggregate progress, one-file
  failure, cancel propagation, and final summary.
- [ ] Add metadata building options for SHA-256 and timestamps. Reliable high-level send enables
  checksum calculation by default.
- [ ] Make text send call `upload_bytes` only when a receiver actually requests bytes; never create
  a temporary plaintext file.
- [ ] Refactor CLI send to use the high-level operation and fail with a non-zero exit code on any
  unhandled file failure.

---

## Phase 4 — Finish reverse-download as a library and CLI feature

### Task 4.1: Add reverse-download DTO and client methods

**Modify:**

- `src/protocol/dto_v2.rs`
- `src/client/client.rs`
- `src/lib.rs`
- `src/prelude.rs`

**Create:**

- `tests/interop_download.rs`

- [ ] Against the existing `start_web_share`, write failing tests for `prepare_download`, streamed
  `download`, and `download_to_writer`.
- [ ] Cover PIN `401`/`429`, decline `403`, expired/unknown session, unknown file, interrupted body,
  and parallel downloads.
- [ ] Implement low-level methods through the central URL/TLS/cancellation layer.

### Task 4.2: Safe download-to-directory transaction

**Modify:**

- `src/core/sink.rs`
- `src/core/file.rs`
- `src/client/transfer.rs`
- `tests/interop_download.rs`
- `tests/receive_path_safety.rs`

- [ ] Reuse the held pending receive abstraction for downloads.
- [ ] Test traversal, absolute paths, symlink races, duplicate names, existing files, size mismatch,
  checksum mismatch, and cancellation.
- [ ] Stream, validate, atomically publish, and return the actual collision-resolved path.
- [ ] Apply timestamps best-effort after publication and surface warnings.

### Task 4.3: Make browser-share lifecycle explicit

**Modify:**

- `src/server/server.rs`
- `src/server/web_share.rs`
- `src/server/routes.rs`
- `src/server/events.rs`
- `tests/interop_web_share.rs`

- [ ] Add a test proving an HTTPS-only listener does not advertise an unusable browser URL.
- [ ] Add a `WebShareHandle`/equivalent with URLs, events, active state, and async stop.
- [ ] Prefer a dedicated plain-HTTP listener so the HTTPS LocalSend receiver remains available.
- [ ] If same-port restart is required, test and expose downtime, protocol change, reannouncement,
  and restoration.
- [ ] Test browser disconnect and all-files-complete session cleanup.

### Task 4.4: Add CLI `share` and `download`

**Modify:**

- `src/cli/cli.rs`
- `src/cli/mod.rs`
- `src/cli/commands/mod.rs`
- `src/main.rs`

**Create:**

- `src/cli/commands/share.rs`
- `src/cli/commands/download.rs`
- CLI parser/command integration tests

- [ ] Add parser tests first, including multiple paths, destination, PIN, auto-accept, JSON output,
  and concurrency.
- [ ] `share` prints reachable plain-HTTP URLs and waits until Ctrl-C or requested completion.
- [ ] `download` lists/selects offered files or accepts `--all`, saves safely, and prints final paths.
- [ ] Both commands use only public library APIs and return stable non-zero exit codes on failure.
- [ ] On Ctrl-C, cancel sessions and await listener/file cleanup before exit.

**Exit criterion:** rs-to-rs reverse mode passes in-process and Docker tests without TUI code.

---

## Phase 5 — Server identity, lifecycle, and dual stack

### Task 5.1: Version the event model safely

**Modify/Create:**

- `src/server/events.rs`
- `src/server/server.rs`
- all CLI/TUI event matches
- server event tests

- [ ] Inventory downstream exhaustive matches before changing the enum.
- [ ] Add compile tests/examples for the migration path.
- [ ] Introduce lifecycle outcomes for abort, timeout, cancellation, file failure, and listener failure.
- [ ] Ensure terminal events are not silently lost when the progress channel is full.
- [ ] Mark the future-facing event type non-exhaustive before its first stable release.

### Task 5.2: Verify inbound HTTPS peer identity

**Modify:**

- `src/server/server.rs`
- TLS accept/configuration code under `src/crypto/`
- `src/server/handlers.rs`
- `tests/interop_tls.rs`

- [ ] Start with tests that present one certificate while claiming another JSON fingerprint.
- [ ] Request/accept client certificates and expose the presented leaf fingerprint to handlers.
- [ ] For HTTPS, bind peer/session identity to that verified fingerprint; for HTTP, use the payload
  identity with its documented weaker trust.
- [ ] Test official-core register and upload with client certificates in Phase 7.

### Task 5.3: Dual-stack listener and address reporting

**Modify:**

- `src/server/server.rs`
- `src/core/device.rs`
- CLI receive/share output
- listener integration tests

- [ ] Test independent IPv4 and IPv6 wildcard binding and ephemeral-port agreement.
- [ ] Tolerate systems without IPv6 while exposing the listeners that did bind.
- [ ] Add `local_addresses()` and make URL output bracket IPv6.
- [ ] Prove stop waits for both listeners and releases ports.

---

## Phase 6 — Stateful discovery parity

### Task 6.1: Device/channel store

**Create:**

- `src/discovery/store.rs`

**Modify:**

- `src/discovery/traits.rs`
- `src/discovery/multicast.rs`
- `src/discovery/http.rs`
- discovery tests

- [ ] Use paused Tokio time to test discovered, updated, duplicate, channel-added, and expired
  transitions.
- [ ] Key identity by verified fingerprint and retain multiple channels/addresses.
- [ ] Make multicast, HTTP registration, and subnet scan write the same store.
- [ ] Make `get_known_devices()` return snapshots from the store.

### Task 6.2: IPv6 multicast and periodic lifecycle

**Modify:**

- `src/discovery/multicast.rs`
- `src/discovery/multicast/interfaces.rs`
- `src/protocol/constants.rs`
- discovery tests

- [ ] Add official IPv6 group constants from the pinned oracle.
- [ ] Unit-test packet shapes and interface selection without needing a real LAN.
- [ ] Add IPv4/IPv6 listeners, periodic announce, reannounce on share state change, and stop cleanup.
- [ ] Keep failures on one family from disabling the other; emit a diagnostic event.

---

## Phase 7 — Build the official Docker oracle

### Task 7.1: Create a headless official-core harness

**Create:**

- `tools/oracle/Cargo.toml`
- `tools/oracle/src/main.rs`
- `e2e/docker/official-core.Dockerfile`
- `e2e/scenarios/oracle_self_test.sh`

- [ ] Build against exactly the commit in `e2e/oracle.lock`; fail the build if HEAD differs.
- [ ] Implement deterministic subcommands: `serve-upload`, `send`, `serve-download`, `download`,
  `info`, and `discover`.
- [ ] Keep policy in scenario scripts; the harness should be a thin official-core adapter.
- [ ] Add a self-test where official core sends to/receives from itself and asserts SHA-256.
- [ ] Cache the build image by lock-file hash.

### Task 7.2: Package the official CLI

**Create:**

- `e2e/docker/official-cli.Dockerfile`
- `e2e/scenarios/official_cli_to_rs.sh`

- [ ] Build `localsend-cli` from the locked commit.
- [ ] Use its documented headless `send --to <IP>` path.
- [ ] Preseed isolated config/identity storage; never read the host user's LocalSend configuration.
- [ ] Assert the official process exits successfully and the rs receiver's final hash matches.

### Task 7.3: Add the optional official Linux GUI image

**Create:**

- `e2e/docker/official-gui.Dockerfile`
- `e2e/scenarios/official_gui_smoke.sh`
- `e2e/OFFICIAL_GUI.md`

- [ ] Build the Linux bundle from the lock commit or download the matching AppImage with an enforced
  SHA-256.
- [ ] Install only documented runtime dependencies, Xvfb, D-Bus, and optional noVNC.
- [ ] Add a health probe that confirms the process stays alive and its v2 info endpoint becomes
  reachable.
- [ ] Automate only stable flows. Document manual VNC steps for any UI-only acceptance.
- [ ] Put this service behind the `official-gui` profile and keep it out of the normal merge gate.

---

## Phase 8 — Expand the scenario matrix

### Task 8.1: Compose topology and scenario runner

**Modify:**

- `e2e/compose.yaml`
- `e2e/run.sh`

**Create:**

- `e2e/scenarios/run.sh`
- `e2e/fixtures/manifest.json`
- scenario scripts named in the design

- [ ] Add profiles `smoke`, `secure`, `reverse`, `oracle`, `official-cli`, `discovery`, and
  `official-gui`.
- [ ] Use isolated dual-stack networks and named volumes per scenario.
- [ ] Put readiness, deadlines, result JSON, logs, and cleanup in one runner.
- [ ] Make scenarios order-independent and safe to run concurrently with unique Compose project
  names.

### Task 8.2: Direct-transfer and failure scenarios

- [ ] E01 rs-to-rs HTTP single/multi-file.
- [ ] E02 rs-to-rs HTTPS with certificate fingerprint assertion.
- [ ] E03 wrong PIN/fingerprint and zero payload sent.
- [ ] E04 partial acceptance and text preview.
- [ ] E05 checksum `422` followed by same-token retry.
- [ ] E06 cancel while waiting and mid-stream; no partials and slot reusable.

### Task 8.3: Reverse and browser scenarios

- [ ] E07 rs share to rs download over HTTP and HTTPS-capable client context.
- [ ] E08 real browser-page asset/prepare/download/PIN/refresh flow using an HTTP driver.
- [ ] Assert browser URLs are plain HTTP and normal HTTPS receiving remains available where the
  chosen listener architecture supports it.

### Task 8.4: Official interoperability scenarios

- [ ] E09 rs sender to official-core receiver.
- [ ] E10 official-core sender to rs receiver.
- [ ] E11 rs share to official-core downloader.
- [ ] E12 official-core share to rs downloader.
- [ ] E13 official CLI headless sender to rs receiver.
- [ ] Capture exact DTO/status/log artifacts on failure while redacting PINs/tokens.

### Task 8.5: Discovery scenarios

- [ ] E14 IPv4 multicast discovery/update/expiry on Linux.
- [ ] E14 IPv6 multicast discovery/update/expiry on Linux.
- [ ] Verify HTTP-register fallback feeds the same device store.
- [ ] Detect Docker Desktop and skip with a clear diagnostic; never convert an unsupported network
  into a false green result.

**Verify:**

```bash
./e2e/run.sh smoke
./e2e/run.sh oracle
./e2e/run.sh all
```

---

## Phase 9 — Packaging, documentation, and downstream migration

### Task 9.1: Feature and package hygiene

**Modify:**

- `Cargo.toml`
- `.gitignore`
- `.dockerignore`
- `README.md`

- [ ] Add a package-content assertion around `cargo package --list`.
- [ ] Decide and document minimal default library features versus CLI defaults.
- [ ] Test every supported feature combination in CI.
- [ ] Ensure examples compile against the public API and rustdoc has no broken links.

### Task 9.2: Update durable documentation

**Modify:**

- `README.md`
- `AGENTS.md`
- existing v2.1 docs with a short superseded notice, without deleting their history

- [ ] Replace stale “missing download” statements with an exact support matrix.
- [ ] Document security/trust modes, browser plain-HTTP warning, CLI examples, Docker profiles, and
  the pinned oracle update procedure.
- [ ] State what “v2.2 parity” excludes.

### Task 9.3: Compile and migrate downstreams

**In the CrossCopy superproject:**

- [ ] Point `zmanager-localsend` at the candidate localsend-rs commit/tag.
- [ ] Update exhaustive event matches, especially currently ignored Web Share events.
- [ ] Run `cargo test -p zmanager-localsend`.
- [ ] Run `cargo check -p zmanager-ffi` and the relevant `apps/xc` checks.
- [ ] Keep the dependency bump isolated from unrelated CrossCopy changes.

### Task 9.4: Release candidate and tag

- [ ] Run the full local gate.
- [ ] Run `./e2e/run.sh all` on Linux Docker.
- [ ] Build/boot the optional `official-gui` profile and attach evidence to the release.
- [ ] Run `cargo package` and inspect its contents.
- [ ] Produce a migration note from `0.1.3` to `0.2.0`.
- [ ] Tag/publish only with explicit release approval.

---

## Definition of done

- All `P22-*` requirements in the design have a test or a documented platform limitation.
- The library completes all four roles: send, receive, share, download.
- The CLI exposes those roles without duplicating protocol logic.
- Required Docker scenarios pass against the locked official core in both directions.
- The official CLI sends successfully to our receiver; the official GUI profile boots and is
  documented as optional smoke coverage.
- No mismatch, cancellation, timeout, or unsafe path leaves a published partial file.
- Package, feature matrix, lint, formatting, unit, conformance, interop, and downstream checks are
  green.
- The current support matrix in README/AGENTS matches reality.

