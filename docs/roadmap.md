# Tauri Servo runtime roadmap

Velocetty is an in-house fork of Vercel Hyper for developers working across
concurrent agent sessions. Changing its host must preserve terminal input,
output, and desktop behaviour. The [terms of reference](terms-of-reference.md)
define that outcome. The [technical design](tauri-runtime-servo-design.md)
defines the proposed contracts. The [conformance gates](conformance-gates.md)
define the acceptance criteria and their identifiers. The [context](context.md)
defines the vocabulary.

Each phase tests a hypothesis. Steps answer the questions that determine
whether the next investment is justified. Tasks are reviewable deliverables.
Progress depends on evidence for a named build and target. The sequence carries
no delivery dates. The agent workshop prepares changes and reproduces the
evidence. The sole human maintainer reviews the evidence and decides which
patches, budgets, and release claims df12 can sustain.

Each task names its evidence mode, as the conformance gates define them: desk
review, headless CI, scripted native runner, or human-observed. A
human-observed task needs the maintainer at the machine; those tasks are
batched into one session for each phase. A task marked external depends on work
in the Velocetty repository that this project does not own.

The design review of 26 November 2026 approved phases 1 to 3. Task 3.2.3
records the decision that admits phase 4.

## 1. A reproducible starting point

Idea: the cheapest experiments can falsify the terminal bet and size the
privileged work before any adapter exists.

### 1.1. Falsify the terminal bet on a stock engine

A stock engine either runs the locked terminal or it does not. This step needs
no Tauri, adapter, or patch, so a failure costs little and stops the programme
early.

- [ ] 1.1.1. Run the locked xterm fixture in a stock Servo 0.8.0 shell.
  - Evidence: scripted native runner, with one human-observed check of typing.
  - See [terminal engine gate][graphics] and gate P-2.
  - Success: with WebGL2 and IntersectionObserver enabled individually,
    record for the WebGL2, Canvas, and DOM renderers whether a recorded trace
    renders, the terminal takes focus, and typed text appears. State whether
    the publicly reported WebGL2 failure reproduces. If no renderer yields a
    usable terminal, stop and keep Electron.
- [ ] 1.1.2. Inventory the product subset and its native services.
  - Evidence: desk review of the pinned Velocetty sources.
  - See [open decisions][decisions] and [stage V][v].
  - Success: a checked-in inventory lists every direct Electron and Node
    dependency, and every runtime operation, plugin command, event, and
    protocol form the product needs, each marked required, unsupported, or
    excluded. It states whether the frontend uses cookies or web storage, and
    whether any requirement needs an in-page Tauri API.

### 1.2. Select the first build and evidence profile

A successful native launch identifies one target on which the terminal and host
experiments can run. A failing launch identifies the dependency to resolve
before extending the adapter.

- [ ] 1.2.1. Record the first target, display backend, Tauri workspace,
      Servo revision, host revision, features, patches, and compiler in a
      versioned manifest.
  - Requires 1.1.1. Evidence: scripted native runner.
  - See [Tauri baseline][tauri], [Servo baseline][servo], and
    [ADR 0002][adr-toolchain].
  - Success: a clean build resolves the recorded graph and launches an inert
    native window on the named machine. Compare the Servo 0.7.0 control with
    0.8.0 and the 0.6 support line on the same target, and advance the Tauri
    pin to the alpha.6 set. Record API gaps. Check the engine preference
    profile against each revision's preference table.
  - Record clean and incremental build time, peak link memory,
    target-directory size, and hosted-runner feasibility for each Makefile
    flag set and each toolchain arrangement in ADR 0002, and the cost of one
    pin bump. Name the machine, GPU, driver, and window system for the
    target, and where each patch series lives.
  - Replace the dated upstream audit in the design with a generated
    `docs/upstream-baseline.md` for the chosen pins.
- [ ] 1.2.2. Define the evidence record and pre-register the measurements.
  - Requires 1.2.1. Evidence: desk review, then scripted native runner.
  - See [measured acceptance evidence][metrics] and
    [gate identifiers][records].
  - Success: a checked-in machine-readable schema keyed by gate identifier,
    with the four outcomes and the evidence mode, plus a one-page summary
    format.
  - Before any Servo measurement, fix the trace provenance, byte and burst
    rates, line length, viewport, DPI, and scrollback; the minimum runs,
    keystrokes in each run, warm-up, and host interleaving; and the
    statistics: p50, p95, p99, maximum, count over a fixed threshold, and
    longest frame gap. Name one host-independent latency instrument:
    injected keys with screen-capture frame detection, or identical in-page
    timestamps reported as key-to-render-callback. Name the GPU-memory method
    for the target or a mandatory proxy.
  - Capture the Electron reference run and an Electron-against-Electron run
    for the noise floor. The maintainer agrees the Electron-relative budgets
    from those runs.
- [ ] 1.2.3. Record the provisional maintenance decision.
  - Requires 1.1.2 and 1.2.1. Evidence: desk review.
  - See [required engine and core work][core], [related runtimes][related],
    and [ADR 0001][adr-transport].
  - Success: the maintainer chooses the desktop-service transport from
    ADR 0001 using the inventory. For the chosen option, each Servo patch and
    each Tauri patch is listed separately, and each Tauri patch is classified
    as an upstream candidate or a permanent carry. The record states the
    audit of the published in-process runtime, the choice to reuse,
    collaborate, or proceed independently, and the crate's publishable name.
  - The decision is provisional: no patch exists yet. Task 3.2.3 confirms or
    reverses it with measured cost.

## 2. An interactive terminal on native hardware

Idea: the locked xterm stack can preserve terminal contents and accept real
input in Servo, including when the accelerated renderer becomes unavailable.

### 2.1. Qualify the browser-safe terminal fixture

This work decides whether Servo meets the first product gate. A target with
unusable rendering or input stops here pending a measured remedy.

The harness for this phase is the inert window of task 1.2.1, serving the
fixture from a loopback HTTP origin. It is not the production host or the
production origin. Its results describe the engine. Host-level input and
accessibility are tasks 3.1.4 and 3.1.5, and the production origin is task
4.1.1.

- [ ] 2.1.1. Isolate the pinned frontend fixture behind browser-safe host
  ports and replay recorded terminal output.
  - Requires 1.2.1. Evidence: headless CI and scripted native runner.
  - See [application boundaries][app] and gate P-1.
  - Success: the fixture renders known cells, selection, and scroll position
    without privileged IPC, Node imports, or a live PTY.
- [ ] 2.1.2. Exercise WebGL2, Canvas, and the DOM renderer with the locked
      addons on a native GPU.
  - Requires 2.1.1. Evidence: scripted native runner.
  - See [terminal engine gate][graphics] and gates P-2 and P-3.
  - Success: record rendering mode and recovery time from context loss to a
    usable terminal, including hidden panes, Unicode, and transparency.
    Record ligature support as observed.
- [ ] 2.1.3. Probe origin and connection policy from a loopback origin.
  - Requires 2.1.1. Evidence: scripted native runner.
  - See [local service authentication][auth] and gate P-5.
  - Success: record `isSecureContext`, the emitted Origin, CSP `connect-src`
    enforcement, a binary WebSocket to an exact loopback endpoint, and
    refusal of a remote endpoint. The result is an input to ADR 0001.
- [ ] 2.1.4. Capture engine-level input results.
  - Requires 2.1.1. Evidence: scripted native runner.
  - See [input and native desktop behaviour][input] and gate P-4.
  - Success: injected key, chord, paste, and composition sequences yield the
    expected bytes, including a focus-only sequence.
- [ ] 2.1.5. Sweep hidden-pane cost without a backend.
  - Requires 2.1.2. Evidence: scripted native runner.
  - See [terminal panes][panes] and gate P-8.
  - Success: replay recorded output into one, eight, 28, and 42 hidden panes
    with IntersectionObserver enabled and disabled. Record script-thread busy
    fraction, idle CPU, and typing latency in one visible pane. A latency
    beyond the agreed budget at 28 panes stops the reference workspace
    pending a visibility adapter.

### 2.2. Establish the terminal transport boundary

Velocetty owns the Node service and its later Rust replacement. Neither has
started. This project builds a controlled stand-in service as a fixture so that
the terminal proof does not wait. The stand-in implements a draft of the credit
and handshake contract; Velocetty must ratify or replace that draft, and the
stand-in is never shipped.

- [ ] 2.2.1. Replay a shared binary session corpus against the stand-in
      service.
  - Requires 2.1.1. Evidence: headless CI and scripted native runner.
  - See [migration dependencies][migration] and gate P-6.
  - Success: byte order and session identity survive split sequences, resize,
    exit, and disconnect. An uncertain input write is never replayed. The
    handshake uses the challenge and response, a version field, and
    pre-authentication bounds.
  - External: repeat against Velocetty's Node service when it exists.
- [ ] 2.2.2. Qualify parser credits, the write scheduler, and control traffic
      under sustained output.
  - Requires 2.2.1. Evidence: scripted native runner.
  - See [binary service and flow control][flow] and gate P-7.
  - Success: the fixture returns credit from xterm's write-completion
    callback, which the pinned frontend does not yet use, through one
    document-wide write scheduler. A stalled parser stops credit return and
    pending bytes plateau. Sweep one, eight, 28, and 42 concurrently flooding
    sessions and record input latency at each count. A developer can switch
    between sessions, type in the newly focused pane, and close it during
    each sweep. The fixture checks the destination session and exact input
    bytes. Record the credit windows that keep input within budget.
- [ ] 2.2.3. Run the browser-safe frontend on the Electron host against the
      same service.
  - Requires 2.2.1. Evidence: scripted native runner.
  - See [decision gates][decision-gates] and gate V-9.
  - Success: the Electron host passes the same corpus as the Servo harness.
    This task becomes a standing gate, repeated in tasks 4.2.2 and 5.2.3, so
    that the fallback stays shippable.

### 2.3. Benchmark xterm.js renderer configurations

Can a visible-pane WebGL pool keep a foreground shell responsive under
background output, and what does it cost when switching tabs? Compare the same
browser-safe frontend across hosts under the protocol fixed in task 1.2.2. A
single pane isolates renderer cost before the reference workspace introduces
contention.

- [ ] 2.3.1. Capture single-pane Canvas, WebGL2, and DOM renderer controls on
      each host.
  - Requires 1.2.2 and 2.1.2. Evidence: scripted native runner.
  - See [terminal engine gate][graphics] and gate P-9.
  - Success: replay the same timed byte trace on a native GPU with locked
    xterm.js and addon versions, fonts, viewport, scrollback, and build mode.
    Record the observed renderer and any fallback. Cover idle typing, burst
    output, and full-screen redraw. Compare transparent Canvas separately.
  - Hosts: Electron, the Servo harness, and the Tauri CEF and Wry runtimes.
    The CEF and Wry controls show what a patch-free Tauri host achieves.
- [ ] 2.3.2. Reproduce Velocetty's per-document pool in the frontend fixture.
  - Requires 2.3.1. Evidence: scripted native runner.
  - See [Velocetty's pool implementation][pool].
  - Success: reuse the pinned pool and pane lifecycle where practical. Trace
    one WebGL addon per accelerated pane, release on hide or close, Canvas
    fallback, and reacquisition on reveal. With three visible panes and a
    test cap of two, verify least-recently-visible eviction and preservation
    of terminal contents. Repeat with the default cap of sixteen and record
    actual addon disposal and context-loss recovery alongside slot counts.
    Slots release on hide, so expect no eviction at the default cap unless
    more than sixteen panes are visible; record that observation.
- [ ] 2.3.3. Compare Canvas-only and pooled rendering on the reference
      workspace.
  - Requires 2.1.5, 2.2.2, and 2.3.2. Evidence: scripted native runner.
  - See [measured acceptance evidence][metrics].
  - Success: replay 14 decorated tabs with two and three panes per tab
    (28 and 42 sessions). Switch tabs and type while background agents stream
    output. Check focus, input destination, terminal contents, and scrollback
    in each mode. Run idle and noisy traces separately on each host, with a
    fixed-rate trace below saturation and a saturating trace.
    The workspace is a fixture of this project: Velocetty has not yet
    implemented tab decoration, so the fixture supplies it. Hidden panes use
    Canvas in both modes, so the modes differ only in the visible panes.
    Report the hidden-pane cost separately from the visible-pane difference.
  - Publish the pre-registered statistics for key-to-echo and
    tab-switch-to-readable-frame times, achieved throughput beside latency,
    CPU, proportional process-tree memory excluding the backend service, and
    GPU memory or its proxy. Record unsupported cases and measurement gaps.
    Compare the paired results against the budgets agreed in task 1.2.2 and
    record which renderer profiles qualify for the two-window trial.

## 3. Two windows sharing one engine

Idea: a single UI owner can independently present two native windows and retire
either surface while the other window continues to accept input, at a carrying
cost the maintainer accepts.

### 3.1. Prove window and event-loop ownership

Inert content keeps this host proof independent of the privileged bridge. A
failure of presentation or re-entrancy blocks qualification on that host.

- [ ] 3.1.1. Present independently identifiable fixtures in two windows.
  - Requires 1.2.1. Evidence: scripted native runner.
  - See [native windows and rendering][windows], [Verso prior art][verso],
    and gates H-1 and H-2.
  - Success: close the first window, restore the second after a DPI change,
    and open a third without stale pixels or a stale native handle.
- [ ] 3.1.2. Integrate wakes, bounded dispatch, and redraw with the host loop.
  - Requires 3.1.1. Evidence: scripted native runner.
  - See [ownership and event-loop contract][loop] and gate H-3.
  - Success: a wake during pumping is delivered, idle windows stop repainting,
    and creating a window from an application callback completes. Demonstrate
    cross-thread calls through Send + Sync handles while all Rc-owned engine
    objects stay on the UI thread.
- [ ] 3.1.3. Exercise pending work during close, veto, and engine teardown.
  - Requires 3.1.2. Evidence: scripted native runner.
  - See [failure and shutdown semantics][failure] and gate H-4.
  - Success: a veto preserves the view and committed closure retires it once.
    Exit follows the declared order, and evidence records the observed
    teardown time and any watchdog firing.
- [ ] 3.1.4. Capture host-level input on the production host loop.
  - Requires 2.1.4 and 3.1.2. Evidence: human-observed.
  - See [native services and accessibility][native] and gate H-5.
  - Success: native IME composition, candidate placement, dead keys, and
    shortcut sequences yield the expected bytes in the terminal fixture.
- [ ] 3.1.5. Capture assistive-technology results on the production host.
  - Requires 3.1.2. Evidence: human-observed, in the same session as 3.1.4.
  - See [native services and accessibility][native] and gate H-6.
  - Success: a screen reader announces focus, cursor movement, and
    representative new output.
- [ ] 3.1.6. Inject engine and surface failures.
  - Requires 3.1.3. Evidence: scripted native runner.
  - See [failure and shutdown semantics][failure] and gate H-7.
  - Success: for a script-thread panic, an engine panic, a WebGL context
    loss, a presentation-surface loss, and a stalled shutdown, record what
    survives and correct the design's blast-radius table from observation.
- [ ] 3.1.7. Measure aggregate memory and input responsiveness across windows.
  - Requires 2.1.5 and 3.1.2. Evidence: scripted native runner.
  - See [terminal panes][panes] and gate H-8.
  - Success: on recorded traces, record queue and GPU peaks, latency, and
    idle CPU for one, two, and ten windows with a fixed number of visible
    panes in each, including a stalled hidden pane. Run to WebGL context
    exhaustion. Repeat the 14-tab workload with a second native window open
    on the settings tab, change a setting, and verify focus and echo latency
    after returning to the shell. Velocetty presents settings as a tab type,
    so the second window is an ordinary workspace window. Agree admission
    limits, a process-wide accelerated-pane cap, and product budgets, or
    narrow the supported profile.

### 3.2. Rehearse the carry and confirm the investment

The dominant known cost is carrying patches that cannot go upstream. This step
measures that cost on the smallest real patch before the privileged work
multiplies it, and gives the maintainer one decision that admits phase 4.

- [ ] 3.2.1. Spike each engine and core change the chosen transport retains.
  - Requires 1.2.3. Evidence: desk review and headless CI.
  - See [required engine and core work][core].
  - Success: for each retained change, a throwaway patch against the pinned
    revision, with the subsystems it touches, its size, and the conflicts it
    meets with open upstream proposals.
- [ ] 3.2.2. Carry the smallest real patch across releases.
  - Requires 3.2.1. Evidence: headless CI and scripted native runner.
  - See [packaging and upgrades][package].
  - Success: carry one script-thread patch from Servo 0.7.0 to 0.8.0 and to
    the next monthly release, and one Tauri patch across one pre-release.
    Record workshop time and the maintainer's review minutes for each move.
- [ ] 3.2.3. Record the investment decision.
  - Requires 2.3.3, 3.1.6, 3.1.7, and 3.2.2. Evidence: desk review.
  - See [decision gates][decision-gates] and
    [motivation and stop criteria][motivation].
  - Success: the maintainer compares the measured benefit over the CEF and
    Wry controls with the measured carrying cost and the declared
    patch-latency budget, records the session-survival policy and the
    acceptance of the threat model, and decides to proceed, narrow, or stop.
    The maintainer also accepts or rejects ADR 0001 and ADR 0002 with the
    evidence gathered, and records the governing decisions as ADRs.

## 4. A document can use desktop services safely

Idea: the packaged app can obtain its local service connection while the host
keeps native authority bound to the originating live document.

The tasks below describe the native-bridge transport. If task 1.2.3 selected
another option from ADR 0001, restate steps 4.1 and 4.2 for it before starting
them.

### 4.1. Establish the packaged origin and document lifecycle

Only a maintained engine and core contract that passes these gates unlocks
privilege. The inert host remains usable for investigation if a gate fails.

- [ ] 4.1.1. Serve a packaged fixture through the reserved origin.
  - Requires 3.1.2 and 3.2.3. Evidence: scripted native runner.
  - See [virtual origins][origins] and gate V-1.
  - Success: the fixture loads under ordinary browser policy. Cancellation,
    an unavailable provider, and byte-limit violations terminate locally.
    Repeat the origin and connection probe from the production origin.
- [ ] 4.1.2. Bind initialization to the eligible document before author script.
  - Requires 4.1.1. Evidence: scripted native runner.
  - See [document bootstrap][bootstrap],
    [navigation and frame policy][navigation], and gate V-2.
  - Success: eligible navigation gets a fresh identity, same-document changes
    preserve it, and ineligible frames receive no bridge. Cover
    `document.open` and a history entry restored from cache.
- [ ] 4.1.3. Carry document identity through one JSON desktop command.
  - Requires 4.1.2. Evidence: scripted native runner.
  - See [contract adaptation notes][contracts] and gates V-3 and V-4.
  - Success: Tauri authorizes the request, the script thread checks the target
    at delivery, and replacement of the document cancels that delivery. One
    accepted command produces at most one observable side effect. A command
    reads its document identifier, and the retirement observer fires once
    and before the successor's bootstrap.
- [ ] 4.1.4. Rebase the privileged patch series across one monthly release.
  - Requires 4.1.3. Evidence: headless CI and scripted native runner.
  - See [engine security policy][security].
  - Success: record the elapsed time and the maintainer's review minutes
    against the declared patch-latency budget. The maintainer confirms or
    reverses the decision of task 3.2.3.

### 4.2. Connect an authorized terminal to the service

This slice decides whether the browser proof and host proof compose under the
production origin. Local backend authorization remains a product duty.

- [ ] 4.2.1. Integrate the lease and the authenticated host channel.
  - Requires 2.2.2 and 4.1.4. Evidence: scripted native runner.
  - See [local service authentication][auth] and gates V-5 and V-6.
  - Success: the packaged frontend can open its scoped session. Retirement,
    service restart, or control-channel loss revokes the old lease and stops
    its associated sessions and output. Killing the host leaves the backend
    alive and its sessions in the recorded state.
  - External: the lease contract is Velocetty's to ratify.
- [ ] 4.2.2. Publish the first product-subset evidence record.
  - Requires 2.2.3, 3.1.4, 3.1.5, 3.1.6, 3.1.7, and 4.2.1. Evidence: scripted
    native runner.
  - See [stage V][v] and gates V-7 to V-9.
  - Success: native runs exercise production origin, renderer fallback,
    reload, window close, and independent windows together. The record names
    every operation in the inventory of task 1.1.2, its outcome, and
    exclusions. The Electron host passes its standing gate.

## 5. A recoverable desktop application

Idea: an installed Velocetty build can preserve terminal and native behaviour
on a named target, including recovery when an update fails.

### 5.1. Qualify the application's desktop workflows

This work identifies which native facilities the product actually uses and
which targets can support those workflows.

- [ ] 5.1.1. Wire the selected menu, clipboard, dialog, and link operations.
  - Requires 4.2.2. Evidence: scripted native runner.
  - See [native services and accessibility][native] and gate B-1.
  - Success: a real user workflow exercises each declared port. Focus returns
    after dialogs and unsupported requirements fail the capability preflight.
- [ ] 5.1.2. Qualify the replacement backend with the same protocol corpus.
  - Requires 4.2.1. Evidence: headless CI and scripted native runner.
  - See [migration dependencies][migration] and gate B-4.
  - External: Velocetty supplies the Rust backend.
  - Success: the Rust backend agrees with the Node reference on sessions,
    Velocetty configuration paths, supported Velocetty plugins, and terminal
    bytes, with adjudicated evidence for every intentional difference.
- [ ] 5.1.3. Run the installed application's native interaction matrix.
  - Requires 5.1.1 and 5.1.2. Evidence: human-observed.
  - See [stage B][b] and gates B-2, B-3, and B-5.
  - Success: record results for each claimed target and display backend,
    including screen reader, keyboard-only use, IME, resize, and close.
    Linux X11 and Wayland have distinct results.

### 5.2. Recover a matched bundle and its profile

An installation and recovery drill decides which package combinations can enter
the support matrix.

- [ ] 5.2.1. Stage checked binaries, resources, licences, and native libraries.
  - Requires 1.2.1 and 5.1.1. Evidence: scripted native runner; signing is
    human-observed.
  - See [packaging and upgrades][package] and gate B-6.
  - Success: a clean machine launches the matched package offline. The
    release has no active development endpoint and passes signing checks
    wherever signing is claimed.
- [ ] 5.2.2. Check profile compatibility before the candidate can write.
  - Requires 5.2.1. Evidence: scripted native runner.
  - See [storage and process model][storage] and gate B-6.
  - Success: concurrent writers fail clearly, ephemeral mode leaves the
    persistent profile intact, and a newer incompatible schema prevents
    unsafe rollback or opens through a compatible recovery procedure. A
    crash followed by a relaunch acquires the lock. Repeated failed launches
    start the retained bundle or safe mode. Skip the migration cases if the
    inventory showed that an ephemeral profile suffices.
- [ ] 5.2.3. Publish the per-target release evidence and recovery drill.
  - Requires 2.2.3, 3.1.7, 5.1.3, and 5.2.2. Evidence: scripted native
    runner.
  - See [diagnostics][diagnostics], [stage B][b], and gate B-7.
  - Success: interrupt an update and fail first launch. Recover a matched
    package and readable profile, compare measured budgets, and disclose
    any lost live sessions. Logs and crash records exclude credentials and
    terminal contents. The Electron host passes its standing gate.
- [ ] 5.2.4. Replay one engine security update through the full series.
  - Requires 4.1.4 and 5.2.3. Evidence: headless CI and scripted native
    runner.
  - See [engine security policy][security] and gate B-8.
  - Success: record elapsed time to rebase, rebuild, qualify, and roll back
    the pinned patch series and the human review effort required. Exercise
    withdrawal through the signed minimum safe version. The maintainer
    reviews the rehearsal against the patch-latency budget and records the
    decision to continue, narrow, or withdraw support.

## 6. General Tauri applications and deferred capabilities

Idea: applications outside Velocetty can reuse the same host safely once a
consumer needs it. No consumer has asked.

### 6.1. Decide whether a general runtime has a consumer

Stages A and C of the conformance gates describe a general runtime and its
extensions. They are not planned. This step keeps the scope defined without
committing work to it.

- [ ] 6.1.1. Record the decision on a general runtime.
  - Requires 5.2.4. Evidence: desk review.
  - See [stage A][a], [stage C][c], and [scope][scope].
  - Success: the maintainer names a second consumer and the criterion from
    the terms of reference that justifies wider investment, or records that
    none exists. A positive decision adds roadmap steps for raw-byte invoke,
    document-bound channels, and the stage A profile on an independent
    fixture. Each requested child view, storage partition, or process
    isolation feature gets a separate decision. Mobile, arbitrary remote
    browsing, printing, and Tauri 2 compatibility require a separate scope
    decision.

[tauri]: tauri-runtime-servo-design.md#tauri-baseline-and-compatibility-policy
[servo]: tauri-runtime-servo-design.md#servo-baseline-retained-improvements-and-required-deltas
[core]: tauri-runtime-servo-design.md#required-engine-and-core-work
[metrics]: tauri-runtime-servo-design.md#establish-measured-acceptance-evidence
[app]: tauri-runtime-servo-design.md#keep-the-application-boundaries-intact
[panes]: tauri-runtime-servo-design.md#terminal-panes-and-engine-surfaces
[pool]: https://github.com/leynos/velocetty/blob/40949b2f4b266d8523d6ffe680f1dbc53ad9b541/lib/utils/webgl-context-pool.ts
[graphics]: tauri-runtime-servo-design.md#make-the-actual-xterm-stack-the-first-engine-gate
[input]: tauri-runtime-servo-design.md#input-and-native-desktop-behaviour
[migration]: tauri-runtime-servo-design.md#follow-the-existing-migration-dependencies
[flow]: tauri-runtime-servo-design.md#preserve-the-binary-service-design-and-add-real-flow-control
[windows]: tauri-runtime-servo-design.md#native-windows-and-rendering
[verso]: tauri-runtime-servo-design.md#what-to-carry-forward-from-verso
[related]: tauri-runtime-servo-design.md#related-in-process-servo-runtimes
[loop]: tauri-runtime-servo-design.md#ownership-and-event-loop-contract
[failure]: tauri-runtime-servo-design.md#failure-and-shutdown-semantics
[origins]: tauri-runtime-servo-design.md#virtual-origins-and-application-resources
[bootstrap]: tauri-runtime-servo-design.md#document-bootstrap-and-trusted-source-identity
[navigation]: tauri-runtime-servo-design.md#navigation-and-frame-policy
[contracts]: tauri-runtime-servo-design.md#contract-adaptation-notes
[auth]: tauri-runtime-servo-design.md#authenticate-the-local-service-independently-of-tauri
[native]: tauri-runtime-servo-design.md#native-services-and-accessibility
[package]: tauri-runtime-servo-design.md#packaging-and-upgrades
[security]: tauri-runtime-servo-design.md#engine-security-policy
[storage]: tauri-runtime-servo-design.md#storage-and-process-model
[diagnostics]: tauri-runtime-servo-design.md#diagnostics-and-developer-experience
[decisions]: tauri-runtime-servo-design.md#open-decisions-and-evidence-required
[decision-gates]: tauri-runtime-servo-design.md#decision-gates
[scope]: tauri-runtime-servo-design.md#scope-and-success-criteria
[motivation]: terms-of-reference.md#motivation-and-stop-criteria
[records]: conformance-gates.md#gate-identifiers-and-evidence-records
[v]: conformance-gates.md#stage-v-verified-product-subset
[b]: conformance-gates.md#stage-b-supported-desktop-application-candidate
[a]: conformance-gates.md#stage-a-complete-general-runtime-alpha
[c]: conformance-gates.md#stage-c-additional-capability-gates
[adr-transport]: adrs/adr-0001-desktop-service-transport.md
[adr-toolchain]: adrs/adr-0002-toolchain-scope-for-the-engine-graph.md
