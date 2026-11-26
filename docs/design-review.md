# Tauri Servo runtime design review

**26 November 2026. Verdict: proceed with conditions.** Build roadmap phases 1
to 3: the stock-engine terminal test, the terminal fixture, the two-window
host, and the carry rehearsal. Phase 4, the privileged bridge, waits for the
investment decision in roadmap task 3.2.3. All six reviewers reached the same
verdict independently.

This review supersedes the verdict of 25 November 2026. That review settled
reply targeting, aggregate budgets, and profile recovery. The source validation
of 26 November then established that none of the required engine interfaces
exists upstream, that Servo's contribution policy bars the workshop's patches,
that the terminal's Canvas fallback has no upstream future, and that another
project already publishes a crate of this name. The cost side of the proposal
rose; the earlier conditions no longer suffice.

The [design](tauri-runtime-servo-design.md) specifies the contracts, the
[conformance gates](conformance-gates.md) define the acceptance criteria, and
the [roadmap](roadmap.md) schedules the work.

## Core bets

| Bet                                                                | Confidence    | Tested by                                  |
| ------------------------------------------------------------------ | ------------- | ------------------------------------------ |
| Servo is worth having for this product                             | Unknown       | Motivation thresholds; roadmap task 3.2.3  |
| The locked xterm stack works in Servo                              | Low to medium | Roadmap tasks 1.1.1 and 2.1.2              |
| One script thread serves a keystroke while 28 to 42 sessions flood | Low           | Roadmap tasks 2.1.5 and 2.2.2              |
| One maintainer can carry two patch series with no upstream route   | Low           | Roadmap tasks 3.2.2, 4.1.4, and 5.2.4      |
| A privileged in-page bridge is required at all                     | Low           | ADR 0001; roadmap tasks 1.1.2 and 1.2.3    |
| An engine failure is rare enough that restarting is acceptable     | Low           | Roadmap task 3.1.6                         |
| One UI-thread owner with identifier-only handles                   | High          | Enforced by the compiler; roadmap step 3.1 |
| Engine-side document validation closes the reply race              | High          | Roadmap task 4.1.3                         |
| The three toolchain pins build as one graph                        | Medium        | Roadmap task 1.2.1; ADR 0002               |

## Findings

Each finding names the reviewer who raised it and the change made in response.
Severity follows the review method: a design flaw must be resolved before the
work it affects; an unresolved risk needs a mitigation or an explicit
acceptance.

### Design flaws

| Reviewer            | Finding                                                                                                                          | Change made                                                                                                                   |
| ------------------- | -------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------- |
| Wafflecat, Dinolump | No document states why Servo is wanted, so there is no benefit to weigh against cost and no stop criterion.                      | Terms of reference gain motivation hypotheses, thresholds, and stop rules; a success criterion requires a measured advantage. |
| Wafflecat           | Alternatives are named and never rejected. The Tauri CEF runtime needs no engine patches and is the null hypothesis.             | Terms of reference gain an alternatives table with dispositions; CEF and Wry become controls in roadmap task 2.3.1.           |
| Doggylump           | An engine failure destroys every agent session, because host loss revokes leases and the backend may share the engine's process. | Design requires a separate backend process, adds a blast-radius table, and separates orderly retirement from host loss.       |
| Doggylump           | No fault is injected before the investment decision.                                                                             | Roadmap task 3.1.6 and gate H-7 inject engine, script-thread, surface, and shutdown failures before task 3.2.3.               |
| Dinolump, Wafflecat | Carrying cost is measured last, after the bridge is built.                                                                       | Roadmap step 3.2 spikes each change and carries one patch across releases before phase 4; task 4.1.4 repeats it.              |
| Dinolump            | Human-observed work has no operator, runner, or budget.                                                                          | Every task names an evidence mode; human-observed tasks are batched; the terms of reference record the hours constraint.      |
| Pandalump           | Phase 2 has no named harness or origin, so its input and accessibility results would not transfer to the real host.              | Roadmap names the harness and its loopback origin, adds an origin probe, and moves host-level input to tasks 3.1.4 and 3.1.5. |
| Buzzy Bee           | Flow control bounds bytes and leaves script-thread time unbounded; many flooding sessions can starve a keystroke.                | Design adds a document-wide write scheduler and derived credit windows; roadmap task 2.2.2 sweeps the session count.          |
| Telefono            | The Tauri patch series has no owner, manifest entry, or distribution route.                                                      | Design lists both series as components, adds the Tauri revision to the manifest, and states distribution through `[patch]`.   |
| Telefono            | The verified subset is first enumerated by the task that tests it.                                                               | Roadmap task 1.1.2 delivers the inventory; stage V tests against it.                                                          |
| Telefono            | No interface tells application code which document invoked a command or when it retired, so the lease cannot be scoped.          | Design specifies a document identifier for commands and an exactly-once retirement observer; gate V-4 tests them.             |

### Unresolved risks

| Reviewer  | Finding                                                                                                      | Change made                                                                                                              |
| --------- | ------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------ |
| Pandalump | The maintenance decision preceded the evidence that could inform it.                                         | Roadmap task 1.2.3 is provisional; task 3.2.3 confirms it with measured cost.                                            |
| Pandalump | Velocetty's unstarted service, credit scheme, and lease were written as internal prerequisites.              | Roadmap marks external dependencies and names a stand-in service that implements a draft contract.                       |
| Wafflecat | The cheapest falsifying experiment was not first.                                                            | Roadmap task 1.1.1 runs the terminal in a stock engine before any build work.                                            |
| Wafflecat | Both patch series sit on moving upstream ground.                                                             | Design confines engine changes behind four adapter ports and classifies each Tauri patch.                                |
| Buzzy Bee | Hidden-pane cost was first measured behind an external dependency.                                           | Roadmap task 2.1.5 sweeps hidden panes without a backend.                                                                |
| Buzzy Bee | The benchmark could not be interpreted: no instrument, no tail statistics, no fixed protocol or noise floor. | Roadmap task 1.2.2 pre-registers the protocol, instrument, statistics, and budgets before any Servo result.              |
| Buzzy Bee | The process-wide GPU bound was unstated and GPU memory might be unobservable.                                | Design states the worst case; task 1.2.2 names a method or proxy; task 3.1.7 runs to context exhaustion.                 |
| Buzzy Bee | The proposed byte limits let one window starve the others.                                                   | Design adds a per-view quota, a reserved share, and a copy multiplier; gate A-4 tests the boundary.                      |
| Buzzy Bee | Build cost was unmeasured.                                                                                   | Roadmap task 1.2.1 records build time, memory, and pin-bump cost for each flag set.                                      |
| Telefono  | Document-targeted delivery had no signature, ordering, or expiry contract.                                   | Design adds a signature sketch, per-document ordering, at-most-once delivery, and expiry handling.                       |
| Telefono  | Provenance cannot separate the top document from a same-origin frame.                                        | Design places same-origin descendants inside the trust boundary and requires a `frame-src` restriction.                  |
| Telefono  | The WebSocket handshake allowed a squatting local process to replay the grant.                               | Design specifies a challenge and response, pre-authentication bounds, a grant lifetime, and a version field.             |
| Doggylump | The blocking engine destructor has no timeout.                                                               | Design orders exit and adds a process-exit watchdog; gate H-4 records it.                                                |
| Doggylump | The threat model was two scattered sentences.                                                                | Design gains a threat-model section, a network allow-list, and a disabled image addon; the terms record the acceptance.  |
| Doggylump | No security-update policy existed.                                                                           | Design gains an engine security policy with a watch, a latency budget, a signed minimum version, and a withdrawal route. |
| Doggylump | A stale lock or a failed engine update could leave a launch loop.                                            | Design requires an operating-system lock, a health definition, and a failed-launch counter with a safe mode.             |
| Doggylump | Failures on users' machines were invisible.                                                                  | Design adds a consented local crash record and a diagnostics bundle.                                                     |
| Dinolump  | The toolchain fallback was undecided.                                                                        | ADR 0002 proposes scoping the workspace flags; task 1.2.1 measures the options.                                          |
| Dinolump  | Four orderings and five artefact names had to be kept in step.                                               | The roadmap is the only order; stage V is named; the context defines four artefacts; the design's glossary is removed.   |

### Improvements

| Reviewer  | Finding                                                             | Change made                                                                                  |
| --------- | ------------------------------------------------------------------- | -------------------------------------------------------------------------------------------- |
| Pandalump | "One terminal outcome" held by convention only.                     | Design specifies one consuming completion type that cancels on drop.                         |
| Pandalump | "Lease", "webview", "frame", and the host's names were overloaded.  | The runtime-side record is a document registration; the context sharpens the other terms.    |
| Wafflecat | A quarter of the design specified a runtime with no consumer.       | Stages A and C are marked unplanned; roadmap phase 6 is one decision task.                   |
| Telefono  | Error reporting needed no core change.                              | The runtime publishes its own error type and a capability query, shrinking the Tauri series. |
| Doggylump | The Electron fallback was asserted, with no task.                   | Roadmap task 2.2.3 and gate V-9 make it a standing gate.                                     |
| Dinolump  | Gates were prose without identifiers, in a 2,100-line design.       | The gates move to their own document with stable identifiers and evidence modes.             |
| Dinolump  | A contributor starting cold lacked fork, harness, and runner facts. | The developers' guide gains a section; task 1.2.1 confirms its proposals.                    |

### Open questions

- Does Velocetty need a persistent browser profile? If not, an ephemeral
  profile removes the migration and rollback hazards. Roadmap task 1.1.2
  answers it. (Doggylump)
- Does Servo's task queue give input priority over terminal parse slices, and
  does a script-thread panic end the process? Roadmap tasks 2.2.2 and 3.1.6
  answer them. (Buzzy Bee, Doggylump)
- Does Servo cap live WebGL contexts? Roadmap task 3.1.7 answers it.
  (Buzzy Bee)
- Can invoke travel on Tauri's protocol route, without a native bridge?
  ADR 0001 records the option. (Telefono)

## Pre-mortem

It is May 2027, and the system has caused a significant incident.

1. **Thirty sessions gone.** A laptop undocks. Surface loss panics a script
   thread and the process aborts. Every window and every PTY dies, with agents
   stopped in the middle of a rebase and a migration. No fault had been
   injected, and no crash record existed. The failed bet: engine failures are
   rare enough to restart. Prevention: the separate backend process, the
   blast-radius table, the session-survival decision, and roadmap task 3.1.6.
2. **A known engine vulnerability stays installed for six weeks.** A
   SpiderMonkey fix ships in a monthly release. The patch series conflicts in
   script code, and the maintainer is away. Every installed build keeps
   rendering agent-printed content without a sandbox. Nobody watched the
   advisories, and no channel could withdraw a build. The failed bet: one
   maintainer can carry the series within a tolerable window. Prevention: the
   engine security policy, the patch-latency budget, and the rehearsals in
   roadmap tasks 3.2.2, 4.1.4, and 5.2.4.
3. **An update produces a launch loop.** An engine update fails graphics
   initialization on one driver family. Affected users cannot open their
   terminal, the retained bundle is never started, and the maintainer learns of
   it days later from an issue. The failed bet: matched bundle and profile
   recovery makes updates safe. Prevention: the health definition, the
   failed-launch counter with a safe mode, and the crash record.

The interrupted-migration scenario of the earlier review remains valid and
keeps its gate. It is no longer the likeliest failure.

## Strongest alternative

Wafflecat's alternative keeps Servo and removes the privileged bridge. The
backend serves the frontend and the terminal WebSocket from one authenticated
loopback origin, which the remote-browser deployment already needs. Desktop
services become messages in Velocetty's protocol. A WebSocket closes with its
document. Successor isolation for pending results depends on the backend lease
contract, which revokes the lease, purges pending payloads, and rejects further
output. Conformance gate V-5 must test this race before the property is claimed.

It plausibly removes the whole Servo and Tauri patch series, survives changes
to Tauri's IPC and Servo's embedding API, and makes the desktop and remote
frontends one artefact. It gives up the in-page Tauri API and plugins and the
general-runtime ambition. It exposes a loopback HTTP listener to local
processes, and it needs a way to deliver a first-navigation grant.

Two cheap results choose between it and the proposal: the native-service
inventory, and the origin and connection probe.
[ADR 0001](adrs/adr-0001-desktop-service-transport.md) records the comparison
and a middle option. If the motive is to leave Electron, with no requirement
for Servo, the Tauri CEF runtime is stronger than either.

## Conditions and next steps

Phases 1 to 3 proceed now. Phase 4 requires all of the following, recorded in
roadmap task 3.2.3.

1. The maintainer names the benefit Servo must deliver, its threshold, the
   patch-latency budget, and the hours available for each phase.
2. A stock engine runs the locked terminal in at least one renderer.
3. The CEF and Wry controls have been measured beside Servo and Electron.
4. The maintainer has chosen a desktop-service transport from ADR 0001.
5. One real patch has been carried across releases, with its cost recorded.
6. Engine and surface failures have been injected, and the maintainer has
   chosen the session-survival policy and accepted the threat model.

The maintainer decides each item. The workshop prepares the evidence. Upstream
maintainers decide whether to accept changes to their projects.

## Findings not applied

- The governing decisions are not yet extracted into decision records. They
  remain proposals, and a decision record captures an accepted decision.
  Roadmap task 3.2.3 records them when the maintainer accepts them.
- The dated upstream audit remains in the design until roadmap task 1.2.1
  generates the per-pin baseline document from a real build.
- The verified-nightly entry in the Polonius policy still names an earlier
  nightly than the toolchain pin. It records a verification that has not been
  repeated, so correcting the date alone would misstate it.

The [source validation record](tauri-runtime-servo-design.md#validation-status)
identifies the pinned interfaces and application evidence behind the review.
