# Tauri Servo runtime design review

**25 November 2026. Verdict: proceed with conditions.** Build the inert
terminal and two-window prototypes. A successful prototype earns a decision
about investing in the privileged bridge. It establishes neither application
compatibility nor a support commitment. The
[design](tauri-runtime-servo-design.md) specifies the contracts and the
[roadmap](roadmap.md) records the exit tests.

## Findings and decisions

| Review lens               | Finding and confidence                                                                                                                                                  | Decision                                                                                                                                        |
| ------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| Pandalump · structure     | The thread boundary has strong source support. Whether the pinned engine and host can present two independent surfaces on a target remains open.                        | Keep Servo objects on their owner thread. Demonstrate independent resize, redraw, and teardown with inert content.                              |
| Telefono · interfaces     | Release blocker. Evaluation against a webview cannot establish which document receives a delayed result. Navigation can preserve the view while replacing the document. | Require engine-side validation of the original document at execution. Retiring that document must cancel or discard its pending delivery.       |
| Buzzy Bee · capacity      | Unresolved risk. A bound per connection gives little assurance about total retained bytes across noisy panes and windows.                                               | Charge queued and in-flight work against aggregate budgets, reserve capacity for lifecycle operations, and measure foreground input under load. |
| Doggylump · operations    | Release blocker. A previous executable can be unusable after an interrupted profile migration.                                                                          | Rehearse recovering a compatible bundle and profile together on a clean installation, including interruption after the first migrated write.    |
| Wafflecat · alternatives  | High confidence in the existing Electron host as a comparison point. Workload parity still needs a recorded run.                                                        | Keep the browser-safe frontend and shared backend usable on Electron while Servo qualification proceeds.                                        |
| Dinolump · sustainability | One human maintainer bears the release decision. Patch upkeep and upstream acceptance remain unproved.                                                                  | Have the workshop replay a security update. Record elapsed time, human review effort, and recovery evidence before accepting the patch burden.  |

## Pre-mortem

1. Background agent sessions saturate the process while a developer switches
   panes. Each connection stays within its own quota, yet foreground input and
   close requests stall. Measure process-wide retained bytes, oldest queued
   work, foreground input latency, and close completion across one, two, and
   ten windows. A representative failing run blocks that workload profile.
2. An installer replaces the application and the engine starts migrating the
   profile. Power fails. A binary-only rollback launches an older reader
   against newer data. Interrupt the upgrade on either side of the first write,
   recover the matching checkpoint, and verify previously stored data before
   reopening the profile for writes.

## Alternatives and decision authority

Keeping Electron in production lets Velocetty develop its agent-oriented
workspace and shared service while Servo runs the same fixture in an inert
harness. This preserves a usable product and costs the team two host fixtures.
It leaves the benefits and maintenance cost of shipping Servo to be measured.
The sole df12 maintainer chooses the first target and acceptable regressions
and decides whether the evidence warrants adoption. The workshop prepares the
comparisons. Upstream maintainers decide whether to accept changes to their
respective projects.

The [source validation record](tauri-runtime-servo-design.md#validation-status)
identifies the pinned interfaces and application evidence behind the review.
