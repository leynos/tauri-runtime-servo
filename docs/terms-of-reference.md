# Tauri Servo runtime — terms of reference

**Status:** Draft v0.3, 26 November 2026. **Audience:** the df12 Productions
maintainer and agent workshop.

The [context document](context.md) defines the domain vocabulary. The
[runtime design](tauri-runtime-servo-design.md) owns implementation decisions.
The [roadmap](roadmap.md) sequences the evidence needed to make a release
decision.

## Background and domain

Velocetty is an in-house fork of Vercel Hyper. Its product vision is “VSCode
for terminals” for agentic developers. Velocetty owns the terminal workspace
experience. Its host migration gives the maintainer a narrower decision. Can
they ship a Servo-backed desktop terminal that preserves the behaviour people
rely on today? A shell that accepts keystrokes but loses input during
composition, stalls under background output, or cannot be operated with a
screen reader would fail that test even if it displayed the first terminal
prompt.

The reference application already runs on Electron. Its design calls for a
desktop frontend and a remote browser frontend to share a backend protocol.
That creates an opportunity to evaluate the desktop host independently of the
shell and terminal services. The repository currently contains a library stub.
The proposed runtime and application migration supply requirements for this
charter. [Library source](../src/lib.rs) and
[Velocetty host migration][migration].

## Motivation and stop criteria

These documents have so far described what a Servo host must not regress. They
have not stated what the maintainer gains from it. Velocetty's own design asks
only for a Tauri webview, which the Wry and CEF runtimes supply today without
engine patches. Until a benefit is named and measured, every phase of the
roadmap is cost.

The maintainer records which of these hypotheses motivates the work. The first
three are measured benefits, each with a falsifying measurement taken against
the best alternative host in the same runs. The last two are strategic
motivations with no measurement.

| Candidate benefit                   | Falsifying measurement                                                                              |
| ----------------------------------- | --------------------------------------------------------------------------------------------------- |
| Smaller installed package           | Installed size against Electron and the Tauri CEF runtime on the first target.                      |
| Lower memory with many sessions     | Proportional process-tree memory on the reference workspace against Electron and CEF.               |
| One engine on every platform        | The same build passing the terminal gates on each claimed target, where Wry's system webviews vary. |
| Independence from Chromium          | A recorded strategic choice; no measurement, and the cost must be accepted on that basis.           |
| A Servo runtime as a goal in itself | A recorded research or community goal; the product framing then becomes a test workload only.       |

The maintainer sets a threshold for each chosen measured benefit and a limit
for each cost before the investment decision in roadmap task 3.2.3. A strategic
motivation has no threshold. It counts towards the investment gate only when
the maintainer records, at task 3.2.3, a written acceptance that names the
motivation and accepts the measured carrying cost on that basis. The benefit
side of the gate passes when at least one chosen measured benefit meets its
threshold, or when a strategic motivation is accepted in this way. Neither
route waives a cost limit. The limits are a patch-latency budget for engine
security fixes and a budget of maintainer hours for each phase. Stop, and keep
the existing host, when any of the following holds:

- No renderer yields a usable terminal on a stock engine.
- No chosen measured benefit meets its threshold against the best alternative
  host, and no strategic motivation has a recorded acceptance.
- A carry rehearsal exceeds the patch-latency budget.
- The maintainer does not accept the unsandboxed engine described in the
  design's [threat model](tauri-runtime-servo-design.md#threat-model).

If the motive proves to be leaving Electron, with no requirement for Servo, the
Tauri CEF runtime satisfies it with the same Tauri integration and no patch
series.

## Alternatives and stakeholders

| Alternative                                     | What it offers                                                            | Disposition                                                                     |
| ----------------------------------------------- | ------------------------------------------------------------------------- | ------------------------------------------------------------------------------- |
| Stay on Electron                                | A working desktop host while the shared service is developed              | The fallback; a standing gate keeps it shippable (roadmap task 2.2.3).          |
| Tauri with the CEF runtime                      | The same Tauri integration, a mature engine, and no engine patches        | Undecided; the null hypothesis, measured as a control in roadmap task 2.3.1.    |
| Tauri with the Wry runtime                      | System webviews and the smallest package                                  | Undecided; measured as a control in roadmap task 2.3.1.                         |
| Adopt or collaborate with the published runtime | An existing in-process Servo runtime for Tauri 2, under this crate's name | Undecided; audited in roadmap task 1.2.3.                                       |
| Tauri 2 in place of a Tauri 3 pre-release       | A stable contract                                                         | Rejected for now: it lacks the runtime URL hook and dynamic runtime selection.  |
| Servo without the in-page Tauri bridge          | Most of the benefit with no patch series, if the product needs no bridge  | Undecided; compared in ADR 0001 and chosen in roadmap task 1.2.3.               |
| Verso                                           | Prior integration experience                                              | Rejected as a base: unmaintained, per-window subprocesses, and lost provenance. |

The published runtime is described under
[related runtimes](tauri-runtime-servo-design.md#related-in-process-servo-runtimes),
and Verso under
[runtime comparisons](tauri-runtime-servo-design.md#what-to-carry-forward-from-verso).
The transport comparison is
[ADR 0001](adrs/adr-0001-desktop-service-transport.md). Tauri's maintainers
have said they prefer a Servo runtime to live outside their repository, so no
first-party option is expected. This crate needs a different published name
whichever alternative is chosen.

The df12 Productions development team consists of one human maintainer and the
agent workshop. Product direction, Velocetty integration, and release approval
are responsibilities held by that maintainer.

| Participant                            | Job and evidence needed                                                                                                                                                |
| -------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Agentic developers (primary end users) | Keep working across sessions, panes, and windows while preserving input, focus, accessibility, and recovery.                                                           |
| df12 Productions maintainer            | Compare candidate hosts, decide which patches and targets to carry, and approve promotion or rollback from recorded evidence.                                          |
| Agent workshop                         | Carry out delegated investigation, implementation, and qualification. Supply reproducible builds, workload traces, and release evidence for the maintainer's decision. |
| External Tauri and Servo maintainers   | Decide whether to accept changes in their respective upstream projects under their own policies.                                                                       |

When choosing a desktop host, the maintainer wants to replay a real terminal
workload against comparable builds, so they can justify a migration or keep
shipping the existing host. When reviewing changes while a coding agent streams
output in another pane, an agentic developer wants keyboard input to reach the
focused shell and the diff to stay readable, so they can decide what to accept.
The host must keep input and presentation responsive under that workload.
Velocetty chooses how to organize panes and sessions.

## Goals and exclusions

1. Establish whether the reference application can perform its ordinary
   desktop workflows on an explicitly identified target.
2. Preserve the distinction between trusted application content and other
   documents when granting access to native operations or terminal services.
3. Give the maintainer reproducible evidence that a packaged build can be
   installed, updated, and recovered on each supported configuration.

The initial scope covers packaged desktop content and top-level windows. Mobile
deployment, general-purpose remote browsing, privileged embedded frames,
independent per-view browser profiles, and printing are outside it. Velocetty
owns shell lifecycle, its service protocol, and migration of its backend. The
runtime project consumes those contracts. Backwards compatibility with Vercel
Hyper configurations and plugins is outside Velocetty's product commitment.

## Success criteria

| Outcome                                      | Decision evidence                                                                                                                                                                                                                            |
| -------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Users complete representative terminal work  | On each claimed target, exercise multi-window interaction, input composition, keyboard navigation, screen-reader use, and foreground input during noisy output with the locked frontend. Compare the observed result to the agreed baseline. |
| Users retain control of native privileges    | Reloading, navigating, or closing a document retires its authority. Exercises show that a successor cannot receive its predecessor's pending privileged results or reuse its connection grant.                                               |
| The maintainer can support the result        | Install a matched package outside a developer checkout. Demonstrate an update and the documented recovery behaviour with an existing profile. Publish the revisions, target, exclusions, results, and measured budgets.                      |
| The host is worth its cost                   | A chosen benefit meets its threshold against the best alternative host in the same runs, and the measured cost of carrying the patch series stays within the declared budgets.                                                               |
| The maintainer can make a migration decision | Record which workloads passed, failed, or await evidence and the rationale for adopting, deferring, or stopping on that target. Wider adoption and revenue targets remain open.                                                              |

## Constraints and assumptions

**Established.** Support claims attach to a target and display backend. Linux
X11 and Wayland require separate evidence. The product depends on a
browser-safe frontend and a shared backend service that Velocetty owns. Where
those dependencies are pending, recorded output can qualify an engine
prototype, while a desktop product claim requires the complete path.

**Assumed.** The existing Electron build can serve as a comparator on
representative hardware with the same application revision, fonts, terminal
traces, and workload. If those conditions differ, record the differences and
repeat the measurement before attributing a regression to the host.

Velocetty will accept a Tauri 3 pre-release and Servo as its desktop host. Its
own design names a Tauri webview without choosing a Tauri version or an engine,
and its backend service, shared protocol, and Tauri packaging have not started.
Tauri 3 has no published stabilization date. The maintainer confirms or revises
this assumption when choosing the first target.

The locked terminal stack is the right first workload. Its Canvas fallback was
removed from later xterm.js releases, and no public evidence shows the
accelerated renderer working in a current Servo release. The first terminal
experiment tests both points before further investment.

**Dependent on others.** A privileged release depends on sustainable delivery
of required upstream interfaces or a maintained downstream patch series.
Acceptance by either upstream project is an open dependency. As of
26 November 2026, none of the required Servo interfaces exists in a release or
on the main branch, and the one relevant upstream proposal has been idle for a
year. Servo's contribution policy excludes generated code, so the workshop
cannot submit its patches upstream. A downstream patch series is the working
assumption. The maintenance budget, available capacity, and delivery dates
remain open.

**Constraints of the team.** One person observes every screen-reader, input
method, physical-display, and signing result. The roadmap marks those tasks as
human-observed and batches them into one session for each phase. The maintainer
states the hours available for each phase; a phase whose human-observed work
exceeds them is narrowed before it starts.

**Accepted risk.** The engine runs in the application process without a sandbox
on Windows and Apple Silicon, and renders attacker-influenced terminal output.
A distributed build requires the maintainer's recorded acceptance of that risk.

## Open decisions

| Question                                               | What closes it                                                                                                                              | Decision affected                           |
| ------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------- |
| Which target and display backend should qualify first? | The maintainer identifies a representative user configuration and access to the hardware and accessibility tools it requires.               | First supported product candidate.          |
| Which regressions can users accept?                    | The maintainer approves latency, resource, and failure thresholds against measured Electron workloads.                                      | Promotion from prototype to product.        |
| Which downstream patches can the team sustain?         | The maintainer records available capacity, patch scope, and an update route. Rehearse an engine update and recovery on the declared target. | Distribution to users.                      |
| What would justify wider investment?                   | The maintainer specifies an adoption or maintenance-cost criterion beyond the first application decision.                                   | Expansion beyond the reference application. |
| Under which name is the crate published?               | The maintainer chooses a free registry name or agrees a shared one with the existing project's author.                                      | Any public release.                         |
| Why Servo, and what threshold must it meet?            | The maintainer selects a benefit hypothesis and its threshold from the motivation table.                                                    | The investment decision.                    |
| How long may a known engine fix stay unreleased?       | The maintainer declares a patch-latency budget in days; the carry rehearsals measure against it.                                            | The investment decision; distribution.      |
| How many maintainer hours does each phase have?        | The maintainer states the hours for human-observed work in each phase.                                                                      | Scope of each phase.                        |
| Do sessions survive an engine failure?                 | The maintainer chooses between destroying sessions on abnormal host loss and a bounded reattachment period.                                 | Backend contract; product subset.           |
| May the backend share the engine's process?            | The design proposes that it may not. Velocetty's maintainer confirms the placement for the Rust backend.                                    | Backend architecture.                       |
| Which desktop-service transport is used?               | The native-service inventory and the origin probe select an option from ADR 0001.                                                           | Size of the patch series.                   |

These questions permit prototype work. Before approving distribution, the
maintainer selects the target, sets acceptance thresholds, and accepts the
ongoing patch burden. A durable change to the reference application or the
support boundary warrants an architectural decision record.

The reference application sources above use the pinned revision reviewed on
25 November 2026. The external claims were revalidated on 26 November 2026; the
design's [validation status](tauri-runtime-servo-design.md#validation-status)
records the result.

[migration]: https://github.com/leynos/velocetty/blob/40949b2f4b266d8523d6ffe680f1dbc53ad9b541/docs/velocetty-design.md#host-migration-electron-to-tauri
