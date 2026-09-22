# LocalSend-RS: v2.2 Protocol, Library, CLI, and Oracle Parity — Design

**Date:** 2026-09-20  
**Status:** Draft for review  
**Repository baseline:** `localsend-rs` `0ff8eb30428e` (`Cargo.toml` version `0.1.2`)  
**Target protocol:** LocalSend v2.2  
**Official oracle baseline:** `localsend/localsend` `230fb692962668ca22ce0e61a8f53ce1cfd32102`
(official app/CLI version `1.18.2`, protocol constant `2.2`)  
**Supersedes for new work:**
`docs/superpowers/specs/2026-07-12-localsend-rs-v2.1-alignment-design.md`

---

## 1. Decision summary

`localsend-rs` will target three distinct kinds of parity. Keeping them separate avoids calling
the port “v2.2 complete” merely because the version string changed.

1. **Wire parity (required):** exact v2.2 routes, payloads, status codes, retry behavior, TLS
   identity, and discovery semantics needed to interoperate with current LocalSend peers.
2. **Official-core capability parity (required):** the Rust library can act as sender, receiver,
   reverse-download host, and reverse-download client without requiring the TUI. The CLI is a
   thin, scriptable consumer of those library operations.
3. **Official-app product parity (bounded):** Docker smoke tests prove interoperability with the
   official CLI and Linux GUI. Reproducing every Flutter screen, setting, platform integration,
   and WebRTC/internet-transfer feature is not part of this crate.

The protocol change from v2.1 to v2.2 is narrow: a checksum mismatch during upload is reported as
HTTP `422 Unprocessable Entity` and the session remains usable for a retry. Most work in this
document is not caused by that status-code change; it closes capability and robustness gaps between
the current port and the official v2.2 core.

The library remains the product. CLI and TUI behavior must be implemented through public library
APIs rather than private protocol logic.

---

## 2. Goals

- Advertise protocol `2.2` only after the v2.2 conformance tests pass.
- Match official v2.2 request/response DTOs and endpoint status behavior.
- Provide complete low-level client methods for `info`, upload, cancel, prepare-download, and
  download, plus high-level send/download transactions with cancellation and progress.
- Finish reverse mode as an exposed library and CLI feature. Reuse the existing Web Share server
  implementation rather than replacing it.
- Make HTTPS identity usable with current official clients: certificate-derived fingerprints,
  client certificates, strict peer pinning, and no cross-target pin-cache reuse.
- Support IPv4 and IPv6 URLs/listeners and maintain a real discovery device store.
- Preserve safe receive behavior: bounded streaming, held destination handles, atomic publication,
  traversal/symlink protection, and collision-safe names.
- Replace the stale CrossCopy-superproject-only Docker build with a standalone test harness.
- Run deterministic rs-to-rs and rs-to-official interoperability scenarios in containers.
- Publish a clean library crate and a useful CLI binary with documented feature flags.
- Make downstream impact explicit and compile-check CrossCopy consumers before release.

## 3. Non-goals

- Protocol v3, which remains experimental in the official repository.
- WebRTC or internet transfer; that remains an `apps/xc` responsibility.
- Pixel-for-pixel parity with the official Flutter application.
- Mobile platform integrations, tray behavior, notifications, clipboard integration, auto-start,
  app-store packaging, or operating-system share sheets.
- Resumable or ranged uploads. A LocalSend v2 upload is still the entire file in one POST.
- Making a GUI-in-Xvfb test a required pull-request gate. It is valuable smoke coverage but too
  sensitive to window-system and Flutter changes to be the protocol oracle.
- Supporting v1 routes unless a real downstream requirement is found.

---

## 4. Compatibility and release policy

### 4.1 Preserve the v2.1 baseline first

The current tree is materially newer than the published `0.1.2` crate and has no Git tags. Before
the public API changes, create a release commit and tag **`v0.1.3`** as the LocalSend v2.1 parity
baseline. Do not use a `v2.1` Git tag: that confuses the LocalSend protocol version with this
crate's package version.

The checkpoint is valid only after the existing Rust gates, package inspection, and the standalone
rs-to-rs Docker smoke test pass. If release policy does not permit publishing it to crates.io, the
Git tag is still useful as a downstream rollback point.

### 4.2 Versioning the v2.2 work

- A status-code/version-only patch could be `0.1.4`, but this design includes public DTO, event,
  cancellation, and download API improvements.
- The complete parity release should therefore be **`0.2.0`**.
- Existing low-level upload methods remain available through the `0.2.x` line where practical.
  New high-level APIs are additive; avoid renaming working methods solely for aesthetics.
- Adding variants to `ServerEvent` breaks exhaustive external matches. Either introduce a new
  non-exhaustive event type with a migration adapter, or make the break deliberately in `0.2.0`.
  Do not silently land the break under a patch release.

### 4.3 Known downstream consumers

The CrossCopy superproject consumes this repository through `zmanager-localsend`, with a separately
pinned vendor revision. Its local path build sees the current checkout, while CI/release workflows
keep using the pinned revision until the gitlink or dependency revision is updated.

Every public-API milestone must therefore run:

- `cargo test -p zmanager-localsend`
- `cargo check -p zmanager-ffi`
- any `apps/xc` build that directly consumes LocalSend events or DTOs

The downstream revision bump is a separate, reviewable change after this repository is tagged.

---

## 5. Source of truth

The oracle is a pinned official implementation, not an unversioned interpretation of prose.

- Official repository: `https://github.com/localsend/localsend`
- Pinned revision: `230fb692962668ca22ce0e61a8f53ce1cfd32102`
- v2.2 constant: `packages/core/src/model/discovery.rs`
- DTOs: `packages/core/src/http/dto_v2.rs`
- client behavior: `packages/core/src/http/client/v2.rs`
- server behavior: `packages/core/src/http/server/v2.rs`
- browser/reverse mode: `packages/core/src/http/server/web.rs`
- discovery behavior: `packages/core/src/discovery/` and `packages/core/src/multicast/`
- official CLI: `cli/`

`e2e/oracle.lock` will record the repository URL, commit SHA, official app version, image
architecture, and any downloaded artifact SHA-256. Updating the oracle is an intentional review
event: run the complete matrix, inspect wire diffs, and update this specification if semantics
changed.

---

## 6. Current implementation assessment

The 2026-07-12 document is no longer an accurate backlog. The current implementation already has a
strong receive/upload half and an internal reverse-download host.

| Area | Current state | v2.2 parity work |
|---|---|---|
| Library-first receive | Implemented through `LocalSendServer::builder()` and `ServerEvent` | Extend lifecycle/cancel detail without an accidental patch-level API break |
| Upload server | Streaming, safe path materialization, random tokens, multi-file sessions, PIN lockout | Return `422` for hash mismatch; preserve retry; apply timestamps |
| Upload client | Register, prepare, file/bytes upload, progress, cancel | Add info, cancellation tokens/timeouts, high-level orchestration, parallel files, URL safety |
| HTTPS client | Strict leaf pin after certificate bootstrap; optional client identity | Cache per target, prevent redirects, improve bootstrap contract, cover multiple peers |
| HTTPS server | Self-signed certificate and cert-derived advertised fingerprint | Request/read peer certificate and bind HTTPS peer identity to it |
| Wire DTOs | One broad `DeviceInfo` is reused for internal, register, and info shapes | Split wire DTOs from endpoint/address state and match official serialization exactly |
| Reverse-download server | Implemented in `server/web_share.rs`; integration tested | Make browser mode explicitly plain HTTP and expose stable lifecycle/URL handles |
| Reverse-download client | Missing | Add prepare/download/stream-to-writer/safe-to-directory APIs |
| CLI/TUI reverse mode | Events are ignored | Add scriptable `share` and `download`; optionally expose TUI actions later |
| Discovery | IPv4 multicast + HTTP fallback/subnet scan | Stateful known-device store, expiry/update events, IPv6/scoped-address support |
| Server listener | IPv4 wildcard only | Dual-stack listeners and reported local addresses |
| Metadata | File name, size, MIME; SHA/timestamps not populated by default | Compute optional SHA-256 and RFC 3339 timestamps; apply received timestamps |
| Docker e2e | One rs-to-rs HTTP scenario; Docker context assumes old superproject layout | Standalone images, scenario runner, HTTPS/PIN/download/discovery/oracle matrix |
| Packaging/docs | Crate builds, but package includes internal/editor material and README is stale | Narrow package contents, feature matrix, generated help/readme parity table |

### 6.1 Important existing reverse-mode behavior

`LocalSendServer::start_web_share`, `/prepare-download`, `/download`, `GET /`, the static browser
assets, approval events, progress events, PIN handling, disk-backed streaming, and tests in
`tests/interop_web_share.rs` already exist. This work must be retained.

The current routes share the main listener. That works in HTTP tests but does not make an HTTPS
LocalSend service usable from an ordinary browser, because the browser cannot trust the
self-signed certificate. The official CLI solves this by restarting the link-sharing listener in
plain HTTP. Our public API needs an explicit equivalent instead of silently serving a link that a
browser rejects.

---

## 7. Normative requirements

Requirements are labelled `P22-*` for use by tests and the implementation plan.

### P22-01 — Version and compatibility

- Outbound announcements, register bodies, info responses, and prepare bodies advertise `2.2`.
- Input accepts compatible v2 peers by major version and remains lenient for missing optional
  fields and unknown device types.
- A peer advertising an incompatible major version is rejected with a typed error before transfer.
- No v2.2 advertisement is released while checksum mismatches still return `500`.

### P22-02 — Endpoint DTOs are distinct from runtime peer state

Introduce endpoint-specific wire types equivalent to the official DTOs:

- `RegisterRequest`: includes `port` and `protocol`.
- `RegisterResponse` and `InfoResponse`: do not serialize `port`, `protocol`, or local-only `ip`.
- `PrepareUploadRequest` contains register-shaped sender info and a file map.
- `PrepareDownloadResponse` contains info-shaped host info, `sessionId`, and a file map.
- Optional fields are omitted or defaulted exactly as the pinned oracle does.

Runtime addressing belongs in a separate peer/channel type. Deserializing a wire response must not
invent a default network port and then mistake it for an endpoint supplied by the peer.

### P22-03 — Prepare-upload behavior

- Empty `files` is `400`, matching the current official v2.2 core.
- A valid text-only preview may return `204`; it emits `TextReceived` and does not require an upload.
- PIN outcomes remain `401` and three failures trigger `429` for the configured cooldown.
- User decline is `403`; a busy receive slot is `409`.
- Partial acceptance returns tokens only for accepted file IDs.
- An aborted pending request releases the receive slot promptly.

### P22-04 — Upload integrity and retry

- The server streams exactly one request body into a held pending destination.
- Body size mismatch remains a receiver failure and never publishes a partial file.
- If a declared SHA-256 does not match, the server aborts the pending file and returns `422`.
- A `422` does not mark the file received, close the session, consume its token, or leave bytes on
  disk. Retrying the same file ID/token with correct bytes succeeds.
- Concurrent duplicate uploads for one file return `409`; other accepted files may proceed.
- Successful publication applies declared modified/accessed timestamps on a best-effort basis and
  reports a typed warning if a platform cannot apply one.

### P22-05 — Complete client API

The low-level client exposes:

- `info`
- `register`
- `prepare_upload`
- `upload` from an async stream with a known length
- `upload_file` and `upload_bytes`
- `cancel`
- `prepare_download`
- `download`
- `download_to_writer`
- `download_to_directory` using the same safe materialization rules as receive

All calls accept cancellation and explicit timeouts. Query values are encoded; IPv6 literals are
bracketed; scoped IPv6 has a tested representation. Redirects are disabled so a peer cannot redirect
file bytes away from the selected local endpoint.

### P22-06 — High-level transfer operations

Low-level methods remain composable, but normal consumers should not have to reproduce the protocol
state machine.

- `send` registers, prepares, uploads accepted files with bounded concurrency, reports per-file and
  aggregate progress, sends cancel on local failure/cancellation when a session exists, and returns
  a per-file outcome summary.
- `download_session` prepares, downloads selected files with bounded concurrency, verifies declared
  sizes/checksums, publishes atomically, and returns final collision-resolved paths.
- Dropping a transfer future or cancelling its token releases local resources promptly.
- No high-level text-send path writes the text to a temporary file.

### P22-07 — Reverse-download and browser share

- Existing `WebShareFile` inline/path sources and streaming are retained.
- A share handle exposes its active URLs, session state, stop method, and event stream.
- Browser share is plain HTTP by explicit configuration. It must not accidentally downgrade the
  normal LocalSend listener without the caller opting into that lifecycle.
- Starting/stopping share updates `download` in announcements/info and triggers a re-announcement.
- A reusable `sessionId` refresh behaves like the oracle; unknown/expired IDs are rejected.
- PIN, approval timeout, selected-file validation, progress, disconnect, and session cleanup have
  conformance tests.

The preferred architecture is a dedicated plain-HTTP share listener so normal HTTPS receiving can
continue. If port compatibility forces a same-port restart, expose that interruption in the API and
events; do not hide it.

### P22-08 — TLS identity and trust

- HTTPS server identity is always SHA-256 of the served certificate DER.
- Persisting and reloading a certificate preserves device identity.
- Clients present their own certificate to official peers that request one.
- A pinned peer mismatch fails before any upload/download payload is sent.
- A client reused for two peers never reuses peer A's pinned `reqwest::Client` for peer B.
- Redirects and ambient proxies are disabled for LAN transfer clients.
- The server extracts a presented client certificate and uses its fingerprint as HTTPS peer identity;
  claimed JSON fingerprints are informational in HTTPS mode.
- Test-only insecure trust remains visibly named and cannot be selected by the normal CLI path.

### P22-09 — Server lifecycle and events

The event model must represent, directly or through a versioned successor:

- peer registration with observed address and verified identity
- prepare-upload requested, accepted, declined, refused, timed out, or aborted
- file upload started/progressed/completed/failed
- session finished versus cancelled
- cancel received from the authorized source
- listener failure and stop completion
- browser-share prepare/download/progress/disconnect

No handler holds `ServerState`'s write lock while waiting for network I/O or a consumer decision.
Progress delivery is lossy/non-blocking; terminal lifecycle events must be delivered reliably or be
observable from a transfer handle.

### P22-10 — Discovery and addressing

- Keep IPv4 multicast `224.0.0.167:53317` and add the pinned oracle's IPv6 multicast group
  `ff12::fd3a:e420` on port `53317`.
- Bind HTTP listeners on IPv4 and IPv6 where supported and expose all usable local addresses.
- Maintain a device store keyed by verified fingerprint plus channel, with discovered/updated/expired
  semantics. `get_known_devices()` must return the store, not an empty vector.
- Feed HTTP registration fallback into the same store as multicast observations.
- Periodic announcements, shutdown announcements where supported, interface filtering, and stale
  expiry are deterministic under paused-time unit tests.
- Docker multicast tests are required on Linux CI. Docker Desktop multicast is diagnostic only and
  cannot be the sole release gate.

### P22-11 — CLI completeness

The binary provides non-interactive, exit-code-stable commands suitable for Docker:

- `discover`
- `send <target> <paths...>`
- `receive --directory ... [--auto-accept]`
- `share <paths...> [--pin ...] [--auto-accept]`
- `download <target> --directory ... [--pin ...]`

All long-running commands handle Ctrl-C by cancelling active sessions and stopping listeners. A
`--json` result mode is preferred for the scenario runner. Human mode prints the actual protocol,
address, port, fingerprint, final paths, and transfer result. `--no-https` remains available for
explicit interoperability tests, not as the default.

### P22-12 — Library packaging

- `cargo package --list` contains public source, license, README, and relevant docs only; exclude
  editor state, agent state, journals, and e2e build output.
- `--no-default-features --lib` remains supported.
- Re-evaluate whether a library dependency should enable `cli` by default. The intended target is a
  small library default plus explicit `cli`, `https`, and `tui` features, with a documented migration
  if changing defaults affects downstreams.
- Public errors are typed enough to distinguish rejection, invalid PIN, rate limit, busy session,
  checksum mismatch, cancellation, timeout, unsafe path, and transport failure.

---

## 8. Target library shape

Names below express ownership and behavior; the implementation may adjust spelling without changing
the requirements.

```rust
let peer = Peer::new(endpoint, discovered_identity);
let client = LocalSendClient::builder(local_identity)
    .certificate(local_certificate)
    .timeouts(timeouts)
    .build()?;

let transfer = client.send(peer, sources)
    .pin(pin)
    .concurrency(4)
    .start();

while let Some(event) = transfer.events().recv().await {
    // progress and lifecycle
}
let summary = transfer.wait().await?;
```

Reverse download follows the same split:

```rust
let offer = client.prepare_download(&peer, pin, None).await?;
let summary = client
    .download_session(&peer, offer)
    .destination(downloads)
    .start()
    .wait()
    .await?;
```

Hosting browser downloads returns a handle rather than only mutating hidden server state:

```rust
let share = server
    .share(files)
    .plain_http(true)
    .pin(pin)
    .start()
    .await?;
println!("{}", share.urls().first().unwrap());
share.stop().await?;
```

The existing low-level calls remain useful to CrossCopy and tests. High-level handles compose them;
they do not create a second protocol implementation.

---

## 9. Test architecture

### 9.1 Test layers

| Layer | Purpose | Required gate |
|---|---|---|
| Unit | DTO serialization, URL construction, stores, timers, state machines, path rules | Every commit |
| Raw conformance | Drive the real server with raw HTTP and assert exact status/body shape | Every commit |
| In-process interop | Real Rust client against real Rust server, byte/hash/lifecycle assertions | Every commit |
| Docker rs-to-rs | Packaging, listener, CLI, volumes, signals, HTTP/HTTPS | Pull request |
| Docker official-core | Bidirectional protocol oracle at pinned revision | Pull request after image cache; required for release |
| Docker discovery | Multicast/IPv6 and expiry on a Linux Docker engine | Linux CI/nightly and release |
| Official CLI | Current user-facing official sender against our receiver | Release |
| Official Linux GUI | Xvfb/noVNC boot and selected manual/automated smoke flows | Optional profile; release evidence, not merge gate |

### 9.2 Docker layout

```text
e2e/
  compose.yaml
  oracle.lock
  docker/
    localsend-rs.Dockerfile
    official-core.Dockerfile
    official-cli.Dockerfile
    official-gui.Dockerfile
  scenarios/
    run.sh
    rs_to_rs_upload.sh
    rs_to_official_upload.sh
    official_to_rs_upload.sh
    reverse_download.sh
    checksum_retry.sh
    pin_lockout.sh
    cancel.sh
    discovery.sh
  fixtures/
    manifest.json
tools/oracle/
  Cargo.toml
  src/main.rs
```

The current Dockerfile assumes `localsend-rs` lives under
`vendors/localsend-rs` in the CrossCopy superproject. The replacement builds from this repository's
root. If optional CrossCopy adapters require sibling path dependencies, build a pure public-feature
binary for e2e rather than reconstructing the entire superproject in the image.

### 9.3 Container roles

- **`rs` image:** this repository's release CLI, with curl and certificate/debug utilities only.
- **`official-core` image:** a small `tools/oracle` executable compiled against the pinned official
  `packages/core`. It exposes deterministic `serve-upload`, `send`, `serve-download`, `download`,
  `info`, and discovery commands. This is the authoritative headless oracle.
- **`official-cli` image:** the official `localsend-cli` from the same commit. Its headless
  `send --to` command provides user-facing official-to-rs coverage. Interactive behavior is not used
  as a machine protocol.
- **`official-gui` image:** the official Linux bundle/AppImage under Xvfb with D-Bus and required
  GTK/appindicator libraries. A noVNC endpoint may be exposed for manual inspection. The artifact
  or source commit is pinned and checksummed.
- **`assertor` image:** owns fixture generation, readiness probes, SHA-256 comparison, JSON result
  collection, timeouts, and scenario exit status.

Installing the official GUI in every test container would increase build time and make core tests
fragile. It belongs in its own Compose profile; official core/CLI images cover deterministic
interoperability.

### 9.4 Compose profiles and networks

- `smoke`: rs-to-rs HTTP upload.
- `secure`: HTTPS upload/download, mutual client certificate, pin mismatch.
- `reverse`: prepare-download/download/browser page.
- `oracle`: rs-to-official-core and official-core-to-rs.
- `official-cli`: official headless sender to rs receiver.
- `discovery`: IPv4/IPv6 multicast on a dual-stack bridge.
- `official-gui`: Xvfb/noVNC smoke environment.
- `all`: every deterministic profile except the GUI.

Use one isolated bridge per scenario with fixed IPv4 and IPv6 subnets. Services communicate by both
DNS name and discovered address. Test data moves through named volumes; protocol traffic stays on the
test bridge. Runtime images do not need internet access.

### 9.5 Required scenario matrix

| ID | Sender/host | Receiver/client | Transport | Assertion |
|---|---|---|---|---|
| E01 | rs | rs | HTTP | single and multi-file SHA-256 equality |
| E02 | rs | rs | HTTPS | mutual cert, advertised/served fingerprint equality |
| E03 | rs | rs | HTTPS | wrong pin/fingerprint rejected before payload bytes |
| E04 | rs | rs | HTTP | partial acceptance and text preview `204` |
| E05 | raw driver | rs | HTTP/HTTPS | bad hash `422`, no file, same-token retry succeeds |
| E06 | rs | rs | HTTP/HTTPS | sender and receiver cancellation clean partials and release slot |
| E07 | rs share | rs download | HTTP | reverse files byte-identical, safe names, progress |
| E08 | rs share | browser driver | HTTP | `/`, prepare, download, PIN, refresh session |
| E09 | rs | official core | HTTPS | upload accepted and byte-identical |
| E10 | official core | rs | HTTPS | upload accepted and byte-identical |
| E11 | rs share | official core | HTTP | official download client consumes our offer |
| E12 | official core share | rs | HTTP | our download client consumes official offer |
| E13 | official CLI | rs | HTTPS | headless `send --to` succeeds |
| E14 | rs/official | rs/official | multicast v4/v6 | discovery, update, expiry, reannounce |
| E15 | official GUI | rs | HTTP/HTTPS | boot, info/register, one transfer smoke where automatable |

Every file scenario records declared size, actual size, expected SHA-256, actual SHA-256, final path,
status code, and both process logs. A timeout is a failure, never an indefinite Compose wait.

---

## 10. Acceptance criteria

The `0.2.0` parity release is ready when:

1. `PROTOCOL_VERSION` is `2.2` and raw conformance tests match the pinned oracle.
2. A checksum mismatch returns `422`, leaves no file, and a correct retry succeeds.
3. Library consumers can complete send, receive, share, and download workflows without CLI/TUI code.
4. CLI `send`, `receive`, `share`, and `download` pass deterministic container scenarios and cancel
   cleanly on SIGINT.
5. HTTP, HTTPS, PIN, partial acceptance, text, multi-file, cancellation, and reverse-download tests
   are green.
6. Strict TLS pinning works across multiple targets and official peers that request a client cert.
7. Dual-stack listener/address tests and Linux multicast discovery tests pass.
8. Required rs-to-official and official-to-rs oracle scenarios pass at the locked revision.
9. The optional official GUI profile builds/boots and its limitations are documented.
10. `cargo test`, `cargo clippy --all-targets --all-features -- -D warnings`, `cargo fmt --check`,
    the feature matrix, and `cargo package` verification are clean.
11. CrossCopy downstream compile/tests pass on the candidate tag and the gitlink/revision bump is
    prepared separately.
12. README and AGENTS status describe what the code actually supports.

---

## 11. Risks and mitigations

| Risk | Mitigation |
|---|---|
| Official main changes underneath us | Pin a commit and checksums in `oracle.lock`; update deliberately |
| Official GUI is not headless | Use official core as oracle, official CLI for headless send, GUI as optional Xvfb smoke |
| Docker Desktop drops multicast | Gate discovery on Linux Docker; keep direct-address scenarios portable |
| Rust enum additions break downstream exhaustive matches | Version event API or make the break only in `0.2.0`; compile-check consumers |
| Browser link cannot trust self-signed HTTPS | Explicit plain-HTTP share listener/lifecycle |
| Per-file hashes cost CPU/I/O | Make low-level hash optional; high-level reliable mode computes it before offer and reports hashing progress |
| Concurrent transfer cleanup races | Use cancellation tokens, held pending receives, deterministic state-machine tests, and bounded timeouts |
| Dual-stack behavior differs by OS | Bind IPv4 and IPv6 independently; tolerate unavailable IPv6 while reporting active addresses |
| Package feature changes surprise consumers | Document defaults, run no-default/all-feature matrix, release as `0.2.0` |

---

## 12. Effort envelope

This is larger than the v2.2 wire delta itself. A realistic single-engineer range, assuming the
existing safe receive work remains intact, is:

| Workstream | Estimate |
|---|---:|
| v2.2 wire/DTO conformance | 2–3 days |
| complete client + high-level send/cancel | 3–5 days |
| reverse-download client and CLI/share lifecycle | 4–6 days |
| TLS identity, dual-stack, discovery store | 5–8 days |
| standalone Docker + official oracle matrix | 4–7 days |
| packaging, docs, downstream migration | 2–3 days |

Allow **3–5 engineer-weeks** including debugging across Linux/macOS and official interoperability.
The minimal “advertise 2.2 correctly” slice is much smaller, but it must not be represented as full
library/CLI parity.
