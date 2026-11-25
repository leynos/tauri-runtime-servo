# Tauri Servo runtime — terms of reference

**Status:** Draft v0.1, 25 November 2026. **Audience:** the df12 Productions
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

## Alternatives and stakeholders

Keeping Electron gives the maintainer a working desktop baseline while they
develop the shared service. Tauri's Wry backend offers a system-webview
alternative. The CEF runtime offers another bundled engine. Either warrants a
separate application qualification before a deployment claim. Verso provides
prior integration experience. Its existence alone establishes neither current
Velocetty compatibility nor a reason to replace the current host.
[Runtime comparisons](tauri-runtime-servo-design.md#what-to-carry-forward-from-verso).

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

**Dependent on others.** A privileged release depends on sustainable delivery
of required upstream interfaces or a maintained downstream patch series.
Acceptance by either upstream project is an open dependency. The maintenance
budget, available capacity, and delivery dates remain open.

## Open decisions

| Question                                               | What closes it                                                                                                                              | Decision affected                           |
| ------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------- |
| Which target and display backend should qualify first? | The maintainer identifies a representative user configuration and access to the hardware and accessibility tools it requires.               | First supported product candidate.          |
| Which regressions can users accept?                    | The maintainer approves latency, resource, and failure thresholds against measured Electron workloads.                                      | Promotion from prototype to product.        |
| Which downstream patches can the team sustain?         | The maintainer records available capacity, patch scope, and an update route. Rehearse an engine update and recovery on the declared target. | Distribution to users.                      |
| What would justify wider investment?                   | The maintainer specifies an adoption or maintenance-cost criterion beyond the first application decision.                                   | Expansion beyond the reference application. |

These questions permit prototype work. Before approving distribution, the
maintainer selects the target, sets acceptance thresholds, and accepts the
ongoing patch burden. A durable change to the reference application or the
support boundary warrants an architectural decision record.

The reference application sources above use the pinned revision reviewed on
25 November 2026.

[migration]: https://github.com/leynos/velocetty/blob/40949b2f4b266d8523d6ffe680f1dbc53ad9b541/docs/velocetty-design.md#host-migration-electron-to-tauri
