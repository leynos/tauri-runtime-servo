# Tauri Servo runtime roadmap

Velocetty is an in-house fork of Vercel Hyper for developers working across
concurrent agent sessions. Changing its host must preserve terminal input,
output, and desktop behaviour. The [terms of reference](terms-of-reference.md)
define that outcome. The [technical design](tauri-runtime-servo-design.md)
defines the proposed contracts and conformance stages. The
[context](context.md) defines the vocabulary. Section links below refer to the
technical design.

Each phase tests a hypothesis. Steps answer the questions that determine
whether the next investment is justified. Tasks are reviewable deliverables.
Progress depends on evidence for a named build and target. The sequence carries
no delivery dates. The agent workshop prepares changes and reproduces the
evidence. The sole human maintainer reviews the evidence and decides which
patches, budgets, and release claims df12 can sustain.

## 1. A reproducible starting point

Idea: the workshop can reproduce the proposed baseline from a fresh checkout
and distinguish missing upstream contracts from defects in the adapter.

### 1.1. Select the first build and evidence profile

A successful native launch identifies one target on which the terminal and host
experiments can run. A failing launch identifies the dependency to resolve
before extending the adapter.

- [ ] 1.1.1. Record the first target, display backend, Tauri workspace,
  Servo revision, host revision, features, patches, and compiler in a manifest.
  - See [Tauri baseline][tauri] and [Servo baseline][servo].
  - Success: a clean build resolves the recorded graph and launches an inert
    native window on the named machine. Compare Servo 0.7.0 with 0.8.0 or a
    newer candidate on the same target. Record API gaps and patch costs.
- [ ] 1.1.2. Record the maintenance decision for each required upstream change.
  - Requires 1.1.1. See [required engine and core work][core].
  - Success: the human maintainer accepts the downstream upkeep for a patch or
    records the upstream dependency and the feature it blocks. If the
    required upkeep exceeds available capacity, keep Electron as the product
    host and retain the inert Servo harness.
- [ ] 1.1.3. Define the release evidence record and capture a reference run.
  - Requires 1.1.1. See [measured acceptance evidence][metrics].
  - Success: the record identifies hardware, driver, build mode, fonts, input
    trace, and actual Electron measurements. It distinguishes unsupported,
    untested, and failed cases.

## 2. An interactive terminal on native hardware

Idea: the locked xterm stack can preserve terminal contents and accept real
input in Servo, including when the accelerated renderer becomes unavailable.

### 2.1. Qualify the browser-safe terminal fixture

This work decides whether Servo meets the first product gate. A target with
unusable rendering or input stops here pending a measured remedy.

- [ ] 2.1.1. Isolate the pinned frontend fixture behind browser-safe host
  ports and replay recorded terminal output.
  - Requires 1.1.1. See [application boundaries][app] and [stage P][p].
  - Success: the fixture renders known cells, selection, and scroll position
    without privileged IPC, Node imports, or a live PTY.
- [ ] 2.1.2. Exercise WebGL2 and Canvas with the locked addons on a native GPU.
  - Requires 2.1.1. See [terminal engine gate][graphics].
  - Success: record rendering mode and recovery time from context loss to a
    usable terminal, including hidden panes, Unicode, and transparency.
- [ ] 2.1.3. Capture interactive input and assistive-technology results.
  - Requires 2.1.1. See [input and native desktop behaviour][input].
  - Success: native IME composition and shortcut sequences yield the expected
    bytes, and a screen reader announces focus and representative new output.

### 2.2. Establish the terminal transport boundary

Velocetty owns the Node service and its later Rust replacement. The runtime
consumes the resulting contract. A controlled service can exercise the fixture
while the product implementation is pending.

- [ ] 2.2.1. Replay a shared binary session corpus against the Node service.
  - Requires 2.1.1. See [migration dependencies][migration].
  - Success: byte order and session identity survive split sequences, resize,
    exit, and disconnect. An uncertain input write is never replayed.
- [ ] 2.2.2. Qualify parser credits and control traffic under sustained output.
  - Requires 2.2.1. See [binary service and flow control][flow].
  - Success: a stalled parser stops credit return, pending bytes plateau,
    and a developer can switch between concurrent agent sessions, type in the
    newly focused pane, and close it while another session floods output.
    The fixture checks the destination session and exact input bytes.

### 2.3. Benchmark xterm.js renderer configurations

Can a visible-pane WebGL pool keep a foreground shell responsive under
background output, and what does it cost when switching tabs? Compare the same
browser-safe frontend under Electron/Chromium and Servo. A single pane isolates
renderer cost before the reference workspace introduces contention.

- [ ] 2.3.1. Capture single-pane Canvas and WebGL2 controls on each host.
  - Requires 1.1.3 and 2.1.2. See [terminal engine gate][graphics].
  - Success: replay the same timed byte trace on a native GPU with locked
    xterm.js and addon versions, fonts, viewport, scrollback, and build mode.
    Record the observed renderer and any fallback. Cover idle typing, burst
    output, and full-screen redraw. Compare transparent Canvas separately.
- [ ] 2.3.2. Reproduce Velocetty's per-document pool in the frontend fixture.
  - Requires 2.3.1. See [Velocetty's pool implementation][pool].
  - Success: reuse the pinned pool and pane lifecycle where practical. Trace
    one WebGL addon per accelerated pane, release on hide or close, Canvas
    fallback, and reacquisition on reveal. With three visible panes and a
    test cap of two, verify least-recently-visible eviction and preservation
    of terminal contents. Repeat with the default cap of sixteen and record
    actual addon disposal and context-loss recovery alongside slot counts.
- [ ] 2.3.3. Compare Canvas-only and pooled rendering on the reference
      workspace.
  - Requires 2.2.2 and 2.3.2. See [measured acceptance evidence][metrics].
  - Success: replay 14 decorated tabs with two and three panes per tab
    (28 and 42 sessions). Switch tabs and type while background agents stream
    output. Check focus, input destination, terminal contents, and scrollback
    in each mode. Run idle and noisy traces separately on both hosts.
  - Publish repeated-run counts, warm-up policy, p50/p95 key-to-visible-echo
    and tab-switch-to-readable-frame times, CPU, process-tree memory, and GPU
    memory where observable. Record unsupported cases and measurement gaps.
    Compare the paired results against agreed Electron-relative budgets and
    record which renderer profiles qualify for the two-window trial.

## 3. Two windows sharing one engine

Idea: a single UI owner can independently present two native windows and retire
either surface while the other window continues to accept input.

### 3.1. Prove window and event-loop ownership

Inert content keeps this host proof independent of the privileged bridge. A
failure of presentation or re-entrancy blocks qualification on that host.

- [ ] 3.1.1. Present independently identifiable fixtures in two windows.
  - Requires 1.1.1. See [native windows and rendering][windows] and
    [Verso prior art][verso].
  - Success: close the first window, restore the second after a DPI change,
    and open a third without stale pixels or a stale native handle.
- [ ] 3.1.2. Integrate wakes, bounded dispatch, and redraw with the host loop.
  - Requires 3.1.1. See [ownership and event-loop contract][loop].
  - Success: a wake during pumping is delivered, idle windows stop repainting,
    and creating a window from an application callback completes. Demonstrate
    cross-thread calls through Send + Sync handles while all Rc-owned engine
    objects stay on the UI thread.
- [ ] 3.1.3. Exercise pending work during close, veto, and engine teardown.
  - Requires 3.1.2. See [failure and shutdown semantics][failure].
  - Success: a veto preserves the view, committed closure retires it once,
    and evidence records blocking teardown and its observed duration.
- [ ] 3.1.4. Measure aggregate memory and input responsiveness across windows.
  - Requires 2.3.3 and 3.1.2. See [stage H][h].
  - Success: record queue and GPU peaks, latency, and idle CPU for one, two,
    and ten windows, including a stalled hidden pane. Repeat the 14-tab
    benchmark with the tabbed settings window open, change a setting, and
    verify focus and echo latency after returning to the shell. Agree
    admission limits
    and product budgets or narrow the supported profile.

## 4. A document can use desktop services safely

Idea: the packaged app can obtain its local service connection while the host
keeps native authority bound to the originating live document.

### 4.1. Establish the packaged origin and document lifecycle

Only a maintained engine/core contract that passes these gates unlocks
privilege. The inert host remains usable for investigation if a gate fails.

- [ ] 4.1.1. Serve a packaged fixture through the reserved origin.
  - Requires 1.1.2 and 3.1.2. See [virtual origins][origins].
  - Success: the fixture loads under ordinary browser policy. Cancellation,
    an unavailable provider, and byte-limit violations terminate locally.
- [ ] 4.1.2. Bind initialization to the eligible document before author script.
  - Requires 1.1.2 and 4.1.1. See [document bootstrap][bootstrap] and
    [navigation and frame policy][navigation].
  - Success: eligible navigation gets a fresh identity, same-document changes
    preserve it, and ineligible frames receive no bridge.
- [ ] 4.1.3. Carry document identity through one JSON desktop command.
  - Requires 4.1.2. See [contract adaptation notes][contracts].
  - Success: Tauri authorizes the request, the script thread checks the target
    at delivery,
    and replacement of the document cancels that delivery. One accepted
    command produces at most one observable side effect.

### 4.2. Connect an authorized terminal to the Node service

This slice decides whether the browser proof and host proof compose under the
production origin. Local backend authorization remains a product duty.

- [ ] 4.2.1. Integrate the product-owned lease and authenticated host channel.
  - Requires 2.2.2 and 4.1.3. See [local service authentication][auth].
  - Success: the packaged frontend can open its scoped session. Retirement,
    service restart, or control-channel loss revokes the old lease and stops
    its associated sessions and output.
- [ ] 4.2.2. Publish the first product-subset evidence record.
  - Requires 2.1.3, 3.1.3, 3.1.4, and 4.2.1. See [conformance gates][gates].
  - Success: native runs exercise production origin, renderer fallback,
    reload, window close, and independent windows together. The report names
    every required operation, its outcome, and exclusions.

## 5. A recoverable desktop application

Idea: an installed Velocetty build can preserve terminal and native behaviour
on a named target, including recovery when an update fails.

### 5.1. Qualify the application's desktop workflows

This work identifies which native facilities the product actually uses and
which targets can support those workflows.

- [ ] 5.1.1. Wire the selected menu, clipboard, dialog, and link operations.
  - Requires 4.2.2. See [native services and accessibility][native].
  - Success: a real user workflow exercises each declared port. Focus returns
    after dialogs and unsupported requirements fail the capability preflight.
- [ ] 5.1.2. Qualify the replacement backend with the same protocol corpus.
  - Requires 4.2.1. See [migration dependencies][migration].
  - Success: Velocetty supplies a Rust backend that agrees with the Node
    reference on sessions, Velocetty configuration paths, supported Velocetty
    plugins, and terminal bytes,
    with adjudicated evidence for every intentional difference.
- [ ] 5.1.3. Run the installed application's native interaction matrix.
  - Requires 5.1.1 and 5.1.2. See [stage B][b].
  - Success: record results for each claimed target and display backend,
    including screen reader, keyboard-only use, IME, resize, and close.
    Linux X11 and Wayland have distinct results.

### 5.2. Recover a matched bundle and its profile

An installation and recovery drill decides which package combinations can enter
the support matrix.

- [ ] 5.2.1. Stage checked binaries, resources, licences, and native libraries.
  - Requires 1.1.1 and 5.1.1. See [packaging and upgrades][package].
  - Success: a clean machine launches the matched package offline. The
    release has no active development endpoint and passes signing checks
    wherever signing is claimed.
- [ ] 5.2.2. Check profile compatibility before the candidate can write.
  - Requires 5.2.1. See [storage and process model][storage].
  - Success: concurrent writers fail clearly, ephemeral mode leaves the
    persistent profile intact, and a newer incompatible schema prevents
    unsafe rollback or opens through a compatible recovery procedure.
- [ ] 5.2.3. Publish the per-target release evidence and recovery drill.
  - Requires 3.1.4, 5.1.3, and 5.2.2. See [diagnostics][diagnostics] and
    [stage B][b].
  - Success: interrupt an update and fail first launch. Recover a matched
    package and readable profile, compare measured budgets, and disclose
    any lost live sessions. Logs exclude credentials and terminal contents.
- [ ] 5.2.4. Replay one engine security update through the workshop.
  - Requires 1.1.2 and 5.2.3. See [packaging and upgrades][package].
  - Success: record elapsed time to rebase, rebuild, qualify, and roll back
    the pinned patch series and the human review effort required. The
    maintainer reviews the rehearsal against the update budget and records
    the decision to continue, narrow, or withdraw support.

## 6. General Tauri applications and deferred capabilities

Idea: applications outside Velocetty can reuse the same host safely once each
additional public contract has its own passing evidence. The narrower product
profile remains independently identifiable.

### 6.1. Qualify the complete general runtime alpha

A second application fixture isolates the generic runtime contracts from the
product's backend protocol.

- [ ] 6.1.1. Extend deterministic invoke transport to raw bytes.
  - Requires 4.2.2. See [IPC and asynchronous delivery][ipc].
  - Success: JSON and binary retain distinct meanings, byte limits reject
    overruns, and uncertain dispatch produces one terminal outcome.
- [ ] 6.1.2. Add document-bound channels, events, and payload redemption.
  - Requires 6.1.1. See [IPC and asynchronous delivery][ipc].
  - Success: navigation retires subscriptions and cached payloads, slow
    consumers respect count and byte limits, and other windows make progress.
- [ ] 6.1.3. Run the complete stage A profile on an independent fixture.
  - Requires 6.1.2 and 5.2.2. See [stage A][a].
  - Success: capacity, capability, origin, navigation, and resource-method
    combinations produce the specified results, including unsupported errors.
    The published conformance record identifies exact build and target.

### 6.2. Assess extensions independently after the core release

A proposed extension advances only when a consumer needs it and its independent
contract can be verified.

- [ ] 6.2.1. Assess a requested extension against its own capability gate.
  - Requires 5.2.4 and 6.1.3. See [stage C][c] and [scope][scope].
  - Success: record a separate decision and evidence requirements for each
    requested child view, storage partition, or process isolation feature.
    Mobile, arbitrary remote browsing, printing, and Tauri 2 compatibility
    require a separate scope decision.

[tauri]: tauri-runtime-servo-design.md#tauri-baseline-and-compatibility-policy
[servo]: tauri-runtime-servo-design.md#servo-baseline-retained-improvements-and-required-deltas
[core]: tauri-runtime-servo-design.md#required-engine-and-core-work
[metrics]: tauri-runtime-servo-design.md#establish-measured-acceptance-evidence
[app]: tauri-runtime-servo-design.md#keep-the-application-boundaries-intact
[p]: tauri-runtime-servo-design.md#stage-p-velocetty-engine-and-protocol-proof
[pool]: https://github.com/leynos/velocetty/blob/40949b2f4b266d8523d6ffe680f1dbc53ad9b541/lib/utils/webgl-context-pool.ts
[graphics]: tauri-runtime-servo-design.md#make-the-actual-xterm-stack-the-first-engine-gate
[input]: tauri-runtime-servo-design.md#input-and-native-desktop-behaviour
[migration]: tauri-runtime-servo-design.md#follow-the-existing-migration-dependencies
[flow]: tauri-runtime-servo-design.md#preserve-the-binary-service-design-and-add-real-flow-control
[windows]: tauri-runtime-servo-design.md#native-windows-and-rendering
[verso]: tauri-runtime-servo-design.md#what-to-carry-forward-from-verso
[loop]: tauri-runtime-servo-design.md#ownership-and-event-loop-contract
[failure]: tauri-runtime-servo-design.md#failure-and-shutdown-semantics
[h]: tauri-runtime-servo-design.md#stage-h-host-proof
[origins]: tauri-runtime-servo-design.md#virtual-origins-and-application-resources
[bootstrap]: tauri-runtime-servo-design.md#document-bootstrap-and-trusted-source-identity
[navigation]: tauri-runtime-servo-design.md#navigation-and-frame-policy
[contracts]: tauri-runtime-servo-design.md#contract-adaptation-notes
[auth]: tauri-runtime-servo-design.md#authenticate-the-local-service-independently-of-tauri
[gates]: tauri-runtime-servo-design.md#conformance-and-release-gates
[native]: tauri-runtime-servo-design.md#native-services-and-accessibility
[b]: tauri-runtime-servo-design.md#stage-b-supported-desktop-application-candidate
[package]: tauri-runtime-servo-design.md#packaging-and-upgrades
[storage]: tauri-runtime-servo-design.md#storage-and-process-model
[diagnostics]: tauri-runtime-servo-design.md#diagnostics-and-developer-experience
[ipc]: tauri-runtime-servo-design.md#ipc-and-asynchronous-delivery
[a]: tauri-runtime-servo-design.md#stage-a-complete-general-runtime-alpha
[c]: tauri-runtime-servo-design.md#stage-c-additional-capability-gates
[scope]: tauri-runtime-servo-design.md#scope-and-success-criteria
