# Tauri Servo runtime conformance gates

**Status:** Proposed acceptance criteria. No implementation tests have run.

**Audience:** The agent workshop producing evidence, and the maintainer
auditing it.

The [runtime design](tauri-runtime-servo-design.md) specifies the contracts
these gates test. The [roadmap](roadmap.md) schedules them. The
[context](context.md) defines the vocabulary.

A release claim requires evidence for the exact Tauri revision, Servo revision,
patch-series revisions, runtime revision, build configuration, and target.
Successful compilation establishes API compatibility only; it does not
establish browser policy, native integration, or lifecycle correctness. A
successful no-op or fabricated empty result cannot substitute for an
unsupported-operation error. Do not claim an untested target from another
platform's success.

## Gate identifiers and evidence records

Each gate has a stable identifier: a stage letter and a number. Identifiers are
never reused or renumbered; a withdrawn gate keeps its number and is marked
withdrawn. An evidence record holds one entry for each gate that applies to a
build and target. Each entry states the gate identifier, the outcome, the
evidence mode, and a reference to the log or capture that supports it.

An outcome is one of four values. *Pass* and *fail* describe a gate that ran.
*Unsupported* describes an operation the build declares it does not provide and
rejects visibly. *Untested* describes a gate that did not run; it never counts
towards a claim.

Roadmap task 1.2.2 delivers the machine-readable schema for the record and a
one-page summary format the maintainer can audit quickly.

## Evidence modes

Every gate and every roadmap task names the mode in which its evidence is
gathered. The modes differ in what they cost the one human on the team.

| Mode                   | Who runs it                             | What it can establish                                                    |
| ---------------------- | --------------------------------------- | ------------------------------------------------------------------------ |
| Desk review            | Workshop, from sources and documents    | Inventories, classifications, and decisions awaiting evidence            |
| Headless CI            | Workshop, on hosted runners             | Build, unit, protocol, and software-rendered behaviour                   |
| Scripted native runner | Workshop, on a named machine with a GPU | Native presentation, injected input, timing, and resource measurements   |
| Human-observed         | The maintainer, at the named machine    | Screen-reader speech, IME candidate placement, signing, physical display |

Headless and software-rendered runs provide useful additional evidence but
cannot satisfy a native presentation gate. Human-observed gates are batched
into one session for each roadmap phase, and each session's duration is
recorded against the maintainer's budget in the
[terms of reference](terms-of-reference.md#constraints-and-assumptions).

The build manifest of roadmap task 1.2.1 names the machine for each target, its
GPU, driver, window system, and display configuration. A scripted native runner
is a self-hosted machine the workshop can drive without the maintainer present.
Where no such machine exists for a target, that target's native gates are
human-observed, and the manifest says so.

## Stages and roadmap phases

Stages group gates by the claim they support. They are not a delivery order;
the roadmap is the only delivery order.

| Stage | Claim                                       | Roadmap             | Planned              |
| ----- | ------------------------------------------- | ------------------- | -------------------- |
| P     | The terminal stack works in the engine      | Steps 1.1, 2.1–2.3  | Yes                  |
| H     | One engine hosts independent native windows | Step 3.1            | Yes                  |
| V     | The verified product subset is safe to use  | Steps 4.1 and 4.2   | After the 3.2 gate   |
| B     | A supported desktop application candidate   | Steps 5.1 and 5.2   | After stage V        |
| A     | A complete general runtime alpha            | Step 6.1's decision | No consumer; not yet |
| C     | Additional capabilities                     | Step 6.1's decision | No consumer; not yet |

Stage V does not claim stage A conformance. Conditional groups identify
applicable exercises explicitly: JSON-only invoke and native-events-only
support do not imply binary invoke or channels. Completing every Tauri channel
API must not precede the terminal proof.

## Stage P: terminal engine and protocol proof

Use Velocetty revision `40949b2f4b266d8523d6ffe680f1dbc53ad9b541` and its
actual locked dependencies: xterm 5.3.0, WebGL addon 0.16.0, and Canvas addon
0.5.0.

**P-1 Fixture.** Extract a browser-safe fixture with explicit host ports; do
not attempt to run Electron imports or privileged legacy plugin hooks inside
Servo. Preserve the chosen frontend assets and record any deliberate fixture
differences.

**P-2 Renderers.** Prove WebGL2 rendering, the Canvas fallback, and the DOM
renderer as three separate modes. Cover ResizeObserver delivery, visibility and
context-pool transitions, context loss and recovery, font metrics, Unicode, and
selection. Exercise several xterm panes inside one document; these panes do not
require native child webviews. State whether the build reproduces the publicly
reported WebGL2 failure of rendering, focus, and text input.

**P-3 Engine dependencies.** Verify `document.elementFromPoint` and
`getBoundingClientRect` against the pane occlusion check, `createImageBitmap`
for the Canvas addon, and 2D canvas text measurement for the glyph atlas. Test
search, link, and ligature addons only with their declared support policy,
reporting unsupported features explicitly. The image addon is disabled in the
initial profile.

**P-4 Engine-level input.** Keyboard modifiers, shortcut chords, paste, and
composition events reach xterm with the expected bytes, including a focus-only
sequence. This gate runs on the phase 2 harness and establishes engine
behaviour only.

**P-5 Origin and connection policy.** From a loopback HTTP origin, verify
`isSecureContext`, the emitted `Origin` header, CSP `connect-src`, a binary
WebSocket to an exact loopback endpoint, and refusal of a forbidden remote
endpoint. Do not disable browser policy or treat loopback as authentication.
Stage V repeats this gate from the production origin.

**P-6 Protocol.** Start with recorded output and a controlled service, then
test Velocetty's protobuf/WebSocket service on the existing Node backend when
it exists. Verify binary frames, split UTF-8 sequences, session multiplexing,
and resize and close ordering. Preserve immutable source output and its
ordering as untrusted data: titles, links, and terminal control sequences must
not acquire native authority.

**P-7 Flow control.** Bound output by byte credits and queue limits; return
credit after xterm's write-completion callback, not merely after WebSocket
receipt. A stalled parser stops credit return and pending bytes plateau. Sweep
one, eight, 28, and 42 concurrently flooding sessions and record input latency
at each count.

**P-8 Hidden-pane cost.** Replay recorded output into one, eight, 28, and 42
hidden panes with IntersectionObserver enabled and disabled, without a backend.
Record script-thread busy fraction, idle CPU, and typing latency in one visible
pane.

**P-9 Measurement.** Measure PTY-to-parse and input-to-backend latency on a
native GPU under the pre-registered protocol. Velocetty's deterministic
synthetic benchmark remains a contract fixture and cannot satisfy this gate.

Gates P-2 to P-5 run on the phase 2 harness, which is not the production host.
Their results describe the engine. Host-level input, IME, and
assistive-technology behaviour depend on the native host and are gates H-5 and
H-6; a pass here does not carry over.

## Stage H: host proof

**H-1 Two windows.** Build one process-owned Servo instance and two
independently controlled native windows. Load fixtures without installing
privileged IPC. Prove resize, presentation, navigation, input, focus, and
orderly close. Close the first-created window while the second remains usable,
then create another window from an application callback. Window destruction
cannot invalidate another window's surface, renderer, or engine ownership.

**H-2 Surface lifecycle.** Exercise minimize, occlusion, restore, zero-sized
surfaces, DPI changes, display disconnection, and surface recreation. Rendering
must recover or report a terminal failure without presenting another view's
content.

**H-3 Event loop.** The event loop wakes for work, remains idle without
continuous repainting, and preserves window responsiveness during animation and
command bursts. A wake during pumping is delivered. Cross-thread calls use
`Send + Sync` handles while every engine object stays on the owner thread.

**H-4 Close and teardown.** A veto preserves the view and committed closure
retires it once. Application exit follows the declared order, and the record
states the observed teardown time and whether the watchdog fired.

**H-5 Host-level input.** Input routing preserves physical and logical
coordinates, modifiers, pointer capture, and IME composition, including
candidate-window placement and dead keys, on the production host loop.

**H-6 Assistive technology.** A screen reader announces focus, cursor movement,
and representative new terminal output on the production host.

**H-7 Fault injection.** Inject a script-thread panic, an engine panic, a WebGL
context loss, a presentation-surface loss, and a stalled shutdown. Record what
survives each, and compare the result with the design's blast-radius table.

**H-8 Aggregate capacity.** Record queue and GPU peaks, input latency, and idle
CPU for one, two, and ten windows with a fixed number of visible panes in each,
including a stalled hidden pane. Run to WebGL context exhaustion and record the
behaviour.

Test native GPU paths on every claimed platform. Record GPU, driver, window
system, scale factors, and display configuration with failures.

## Stage V: verified product subset

Stage V covers exactly the operations in the product-subset inventory of
roadmap task 1.1.2. An operation outside the inventory is unsupported and must
fail visibly. The gates below assume the native-bridge transport; if
[ADR 0001](adrs/adr-0001-desktop-service-transport.md) selects another option,
V-2 to V-4 are restated for it before any are run.

**V-1 Production origin.** The packaged fixture loads from the reserved origin
under ordinary browser policy. Cancellation, an unavailable provider, and
byte-limit violations terminate locally. A reserved-origin miss never reaches
DNS or the network. Repeat P-5 from this origin.

**V-2 Bootstrap.** Initialization runs before the first author script in every
eligible document. An eligible navigation gets a fresh identity and a
same-document change preserves it. Cover `document.open` and a history entry
restored from cache. Ineligible frames receive no bridge, and the packaged
`frame-src` restriction holds.

**V-3 Document-bound command.** One JSON desktop command carries the engine's
document identity. Tauri authorizes the request and the script thread checks
the target at delivery. Replacing the document cancels the delivery. One
accepted command produces at most one observable side effect.

**V-4 Retirement interface.** A command reads its document identifier. The
retirement observer fires exactly once for each retired document and before the
successor's bootstrap.

**V-5 Lease lifecycle.** A one-use, scoped bootstrap grant binds a frontend
connection to a backend lease through the trusted host control path. The
handshake uses the challenge and response and rejects an unknown protocol
version. Document retirement revokes grants and leases, rejects further work
and output, closes frontend connections, and destroys their sessions. Race
revocation against grant redemption and pending work. Restarting the backend
invalidates every grant and lease issued by its previous instance.

**V-6 Abnormal host loss.** Kill the host process. The backend survives, and
its sessions follow the recorded policy: destroyed, or held for the declared
grace period and reattached.

**V-7 Threat-model restrictions.** The privileged view loads nothing outside
its allow-list. Terminal-supplied links, titles, and control sequences trigger
no native action without the command policy.

**V-8 Together.** Native runs exercise production origin, renderer fallback,
reload, window close, and independent windows in one session. The record names
every inventoried operation and its outcome.

**V-9 Electron parity.** The Electron host runs the same browser-safe frontend
against the same service and passes the same protocol corpus. This gate keeps
the fallback shippable and is repeated at stage B.

## Stage B: supported desktop application candidate

**B-1 Native integration.** For each claimed operating system and window
system, verify macOS titlebar and control integration and Windows and Linux
frameless drag, resize, focus, and control behaviour. Check menus,
accelerators, supported tray behaviour, focus restoration after dialogs,
clipboard text and file lists, paste ordering, external-link policy, drag and
drop, native handles, and application exit vetoes.

**B-2 Accessibility.** Check keyboard-only navigation, accessible names and
roles, focus announcements, text editing, selection, and dynamic updates with
the target screen reader. A missing accessibility path prevents a supported
desktop claim for that target.

**B-3 Application behaviour.** Velocetty is the primary application fixture.
Verify tabs and splits, terminal buffers and input bytes, sustained output,
fallback rendering, clipboard sequencing, links, configuration changes, and the
`ready`, session-add, session-exit, and update-availability flows. Generic
React and SolidJS applications may extend later coverage; they cannot replace
the terminal acceptance suite.

**B-4 Backend replacement.** Compare the protocol-backed Node implementation
with the Rust replacement using the same corpus. Differences require
adjudication against the declared contract.

**B-5 Session lifecycle.** Preserve Velocetty's policy that document reload and
window teardown destroy their sessions unless a separate product decision
specifies detach and reattach. Test duplicate close, late output, cancellation,
process exit, and uncertain spawn, write, and restart outcomes without
automatic replay. Preserve backend-owned JSON5 configuration and plugin paths;
do not require browser-database persistence for product behaviour that does not
use it. Any claimed browser persistence, clearing, or profile locking still
needs its own tests.

**B-6 Packaging and recovery.** Exercise packaged binaries on clean machines,
including signing where claimed. Test matched application, runtime, engine, and
resource installation, interrupted updates, failed first startup, a crash
followed by relaunch, and recovery to a compatible prior package. The
failed-launch counter reaches its limit and starts the retained bundle or safe
mode. Recovery does not promise live-session or scrollback restoration. Release
builds disable development endpoints; debug grants name their exact development
origin.

**B-7 Budgets.** Record startup, idle CPU, memory, frame timing, and command
latency; compare them with the budgets agreed before measurement.

**B-8 Security update.** Replay one engine security update through the full
patch series and record whether it met the declared patch-latency budget.

## Stage A: complete general runtime alpha

No consumer has asked for this stage. It records the contract a general runtime
would need, so that the decision in roadmap step 6.1 has a defined scope. None
of it is planned work.

Support two top-level windows, one browser profile, packaged content, bounded
binary custom-protocol requests, and the complete chosen invoke and channel
transport. Reject independent child-view creation, conflicting browser-profile
settings, per-view incognito, and unsupported configuration before loading a
privileged page. This restriction permits the proposed single engine-wide
ephemeral profile; it does not promise isolated per-view storage.

**A-1 Initialization and document identity.** Prove that required
initialization executes before the first author script in every document
eligible for the bridge. Cover initial empty documents, reload, redirects,
same-document history changes, new documents, iframe creation, `srcdoc`,
`about:blank`, blob documents, and restored history entries where supported.
Assert the declared frame policy for each case. A frame must not acquire
privilege merely by inheriting a URL, sharing a parent, or requesting an
application asset.

**A-2 Delivery under churn.** Use engine-issued frame and document identities.
Queue replies while navigating, destroying frames, replacing windows, and
recycling registry slots. Servo must validate the target at actual script
execution or message delivery; a check on the UI thread before enqueueing is
insufficient. Old replies, events, and channel data must never reach a
replacement document, even when it uses the same URL and callback number.

**A-3 Authorization and protocol policy.** Attack the local-origin provider
with forged `Origin`, `Referer`, `Host`, invoke-key, and callback headers;
missing or opaque origins; remote frames; redirected requests; conflicting
scheme registrations; and canonicalization variants involving case, trailing
dots, ports, userinfo, encoded separators, and traversal. Authorization must
use trusted initiating context and Tauri's capability checks. Unprivileged
requests remain unprivileged when their destination is a local asset.

**A-4 Fetch policy and limits.** Verify ordinary fetch policy across local and
network resources: CSP, CORS and preflight, credentials, redirects, MIME
handling, caches, and any enabled service-worker interception. Test `GET`,
`HEAD`, byte ranges, and binary `POST` bodies against the same protocol
contract. Exercise every declared byte limit at the boundary and one byte
beyond it, including empty bodies and arbitrary non-UTF-8 data, and the
per-view quota with a second view competing for the reserved share. Reject
overflow deterministically; never truncate a request or encode its body into a
header. Cancellation and handler failure complete or abandon requests without
leaving an unlimited pending map.

**A-5 Invoke, channels, and events.** Use a fixture command with an observable
counter to detect duplicate dispatch. Force failures before dispatch, after
acceptance, during transport completion, and after the command's side effect.
An uncertain outcome must not trigger automatic resend. Test success and error
results, binary payloads, Unicode, malformed messages, invalid callback
identifiers, channel setup and closure, cross-window routing, and application
events. Flood each bounded queue, stall consumers, close documents with queued
messages, and verify the declared overflow policy and ordering within each
channel. Event listeners unsubscribe on document destruction. High-volume
delivery does not starve native input or engine updates.

**A-6 Re-entrancy and failure.** Application callbacks query windows, create
windows, request close, and enqueue engine work without deadlock. Hold no
registry lock or mutable engine borrow across those callbacks. Exercise handler
panics, dropped response receivers, engine shutdown, and partially created
windows. Every accepted request reaches a documented terminal state.

## Stage C: additional capability gates

No consumer has asked for these capabilities. Each is a separate decision with
its own gate.

**C-1 Child views.** Independent child views require bounds, clipping, z-order,
focus, input, destruction, and reparenting tests.

**C-2 Profile isolation.** Profile isolation requires adversarial separation of
cookies, storage, credentials, permissions, caches, and clearing operations.

**C-3 Out-of-process engine.** An out-of-process engine requires an
independently reviewed protocol, cancellation and crash recovery, native
surface ownership, version negotiation, and demonstrated sandbox restrictions
on each supported target. Process separation alone does not satisfy the sandbox
gate.

## Running the gates

Run the affected gates for every engine or core update, and every gate of the
claimed stages before a release. Publish the evidence record with the exact
revisions, target configuration, logs, captured failures, explicit exclusions,
and measured budgets.
