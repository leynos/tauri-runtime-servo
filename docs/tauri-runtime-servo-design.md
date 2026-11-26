# Tauri Servo runtime design

**Status:** Proposed architecture; no implementation or compatibility claim.

**Source baseline:** 29 October 2026. **Design review:** 26 November 2026.
**Source validation:** 26 November 2026.

**Audience:** Runtime implementers, application developers evaluating the\
backend, and reviewers of the Servo and Tauri integration boundaries.

Descriptions of existing behaviour refer to the dated source pins unless a
later revision is named.

The [terms of reference](terms-of-reference.md) define the product outcome. The
[context](context.md)
defines the vocabulary and the [roadmap](roadmap.md) sequences the evidence
needed for delivery.

The [design review](design-review.md) records the panel's verdict and the
conditions for proceeding. The [conformance gates](conformance-gates.md) define
the acceptance criteria. Two proposed decision records qualify this design:
[ADR 0001](adrs/adr-0001-desktop-service-transport.md) on the desktop-service
transport and [ADR 0002](adrs/adr-0002-toolchain-scope-for-the-engine-graph.md)
on toolchain scope.

## Architectural decision

Build `tauri-runtime-servo` as a desktop Tauri runtime with Velocetty as\
its reference application. The runtime owns its native windows and\
consumes Servo through the public `servo` crate. Target the\
Tauri 3 runtime model. Preserve useful Tauri integration behaviours from\
`tauri-runtime-verso`, while replacing its subprocess browser shell,\
engine-internal dependencies, and IPC workarounds.

The initial architecture has one platform event loop, one process-wide\
Servo instance, multiple native windows, and one application storage\
profile. It keeps Tauri's command router, capability system, asset\
handlers, plugin infrastructure, menus, tray integration, and application\
lifecycle. Servo owns the web engine: documents, JavaScript, layout,\
fetching, and painting. The runtime translates between these boundaries.

Five decisions govern the implementation:

1. Use Tauri 3's runtime traits and dynamic runtime selection. Carry a\
   pinned alpha baseline until the relevant interfaces stabilize.

2. Prefer `tauri-winit`, including its GTK4 integration, as the native\
   host. Use the existing CEF runtime as integration evidence, without\
   importing CEF or assuming its event-loop implementation is reusable\
   unchanged.

3. Keep engine objects on the UI thread. Cross-thread handles carry\
   identifiers, generations, and bounded command channels.

4. Require engine-authenticated document identities for privileged IPC\
   and for delivery of its asynchronous results. Implement the necessary\
   Servo and Tauri changes explicitly.

5. Serve app resources through reserved virtual HTTP(S) origins inside\
   Servo's normal fetch policy pipeline. Do not manufacture request\
   origins or use an early interception shortcut as the security boundary.

The resulting adapter should be much smaller than a browser shell, but\
it is a substantive integration project. Modern Servo removes much of the\
old rendering and coordination work; it does not already supply every\
contract a privileged application runtime needs.

## Scope and success criteria

The first useful application release supports Velocetty's React/xterm\
frontend, its planned binary protobuf/WebSocket backend connection,\
multiple top-level windows, and native desktop integration. Tauri IPC\
serves desktop integration and connection bootstrap; terminal traffic\
uses Velocetty's shared protocol. Each claimed platform must also pass its\
native-window, accessibility, installer, and representative application\
tests. Development servers form an explicitly configured development mode.

The target platforms are Windows, macOS, and Linux. Release claims attach\
to tested target triples and display backends, not to an operating-system\
name alone. Linux X11 and Wayland require separate results. This design\
does not assign calendar deadlines.

The initial compatibility profile excludes mobile, arbitrary remote\
browsing, privileged subframes, Tauri's iframe isolation pattern,\
independent per-webview profiles, printing, and live child-webview\
reparenting. These exclusions prevent unsupported behaviour from\
masquerading as ordinary Tauri support. A later stage may add each feature\
after its contract and tests exist.

A successful runtime must:

- Load unmodified representative frontend application code using the\
  supported Tauri API, subject to documented engine and runtime limits.

- Preserve Tauri command authorization and capability scopes.

- Preserve binary request and response meaning across the transport.

- Distinguish a native window, a webview, a frame, and a document.

- Report unsupported operations and failures without successful no-ops.

- Build and package reproducibly from a declared compatibility manifest.

- Upgrade Servo without maintaining a duplicate constellation or\
  compositor implementation.

## Velocetty as the reference application

Velocetty is an in-house fork of Vercel Hyper for developers working with
coding agents. Its product brief is “VSCode for terminals”. A developer may
monitor several agent sessions while typing in another pane, switch between
windows, and inspect earlier output. The runtime must keep input responsive and
deliver events to the intended window under that workload. Velocetty owns the
workspace experience, pane layout, and session semantics. Velocetty makes no
backwards-compatibility promise for Hyper configurations or plugins. Product
fixtures exercise the configuration and plugin contracts Velocetty publishes.

Use Velocetty revision\
`40949b2f4b266d8523d6ffe680f1dbc53ad9b541`, from 29 October 2026, as the\
first application baseline. That revision runs on Electron,\
React 19, Redux, and xterm 5.3.0. It already has a transport facade,\
visible-pane WebGL allocation, Canvas fallback, and runtime telemetry.\
The protobuf/WebSocket service, Rust PTY backend, and Tauri packaging are\
planned work. The frontend still contains Electron and Node dependencies.\
[Package manifest][package-manifest], [transport facade][transport-facade],\
[terminal implementation][terminal-implementation],
[locked dependencies][locked-dependencies],\
[roadmap](https://github.com/leynos/velocetty/blob/40949b2f4b266d8523d6ffe680f1dbc53ad9b541/docs/roadmap.md).

At the November comparison revision, Velocetty's manifest declares Electron
`^45.2.1` (Chromium 156) and xterm `^5.3.0`. The older fixture declares Electron
`^44.4.1` (Chromium 152). Both revisions carry a `build:backend` script, which
only type-checks a placeholder package: neither revision contains a backend
service, and the product roadmap's protocol, service-shell, Rust PTY, and Tauri
items remain open. No commit between the two revisions touches the transport,
terminal renderer, context pool, or session batcher. Keep the October lockfile
for reproducible comparisons. Qualify a newer product revision with the same
terminal workload and an end-to-end backend session before promoting it.
[November Velocetty manifest][velocetty-november].

Velocetty's own design names “the Tauri webview” without selecting a Tauri
major version or a web engine, and does not mention Servo. Targeting Tauri 3
and Servo is therefore a decision of this project that the product has yet to
adopt. The [terms of reference](terms-of-reference.md) record it as an
assumption for the maintainer to confirm.

The product's technical design specifies that both the local Tauri\
frontend and remote browser frontend use the same protobuf/WebSocket\
protocol. It also explicitly requires proving that protocol against the\
existing backend before replacing the backend with Rust. This runtime\
design preserves those decisions.
[Host migration and protocol design](https://github.com/leynos/velocetty/blob/40949b2f4b266d8523d6ffe680f1dbc53ad9b541/docs/velocetty-design.md).

### Keep the application boundaries intact

| Layer                         | Responsibility in the Velocetty target                                                                                 |
| ----------------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| React and xterm frontend      | Tabs, split panes, terminal parsing/presentation, command UI, settings UI, keybinding contexts, and decoration display |
| Velocetty frontend transport  | Binary protobuf/WebSocket framing, connection lifecycle, multiplexing, flow control, and typed backend requests        |
| Velocetty backend service     | PTY/process ownership, command authorization, configuration, filesystem access, plugin storage, and redaction          |
| Tauri application integration | Native window/menu/clipboard/link operations and authenticated local-connection bootstrap                              |
| `tauri-runtime-servo`         | Tauri's runtime contract, native window hosting, Servo embedding, and the supported privilege boundary                 |

Terminal output must not traverse a new Tauri-specific PTY command or\
event protocol. Doing so would create a second frontend service contract\
and defeat the product's local/remote architecture. Native Tauri IPC\
remains available for desktop services that a remote browser cannot\
provide. PTY backpressure and command permissions belong to Velocetty's\
service contract, not to Servo's generic event loop.

The frontend composition boundary must also replace direct Node and\
Electron services. A transport barrel alone cannot make code that calls\
`fs`, `child_process`, `@electron/remote`, `webFrame`, or\
`shell.openExternal` browser-safe. Port those operations into the\
backend or explicit desktop-service ports. Preserve the product's new\
plugin API; do not recreate unrestricted Node access inside the Servo\
document to preserve legacy Hyper internals.

### Terminal panes and engine surfaces

Velocetty's split panes are xterm objects and canvases inside a React\
document. They are not child webviews. Keep one Servo webview per native\
application window, with the existing frontend managing its panes.

| Resource                               | Owner               | Scale                                                                 |
| -------------------------------------- | ------------------- | --------------------------------------------------------------------- |
| Native window and presentation surface | Runtime host        | One per desktop window                                                |
| Servo webview and page scene           | Servo and runtime   | One per application window initially                                  |
| Terminal pane and xterm state          | Velocetty frontend  | Many per document                                                     |
| WebGL2 context and glyph textures      | Browser/xterm addon | One per active accelerated pane; controlled by the application's pool |

The existing pool manages logical ownership slots, not a shared browser\
GL context. Its default cap is sixteen; each acquired pane still owns a\
separate WebGL addon/context. The runtime must not add another terminal\
pool or map each pane to a native view. Test aggregate GPU pressure with\
multiple application windows, since each document's pool does not prove\
a process-wide memory bound.
[Terminal rendering](https://github.com/leynos/velocetty/blob/40949b2f4b266d8523d6ffe680f1dbc53ad9b541/lib/components/term.tsx),
[context pool][velocetty-pool].

A pane releases its slot when it stops being visible, and every hidden pane
runs the Canvas renderer. Least-recently-visible eviction therefore occurs only
when the visible panes outnumber the cap, which the default of sixteen rarely
permits. A workspace with many tabs is mostly Canvas renderers in every
configuration; the accelerated and Canvas-only configurations differ only in
the visible panes. The pane-visibility check also calls
`document.elementFromPoint` and `getBoundingClientRect`, so hit-testing
correctness in Servo is part of the terminal gate.

Each document's pool admits sixteen accelerated panes, and one engine process
hosts every window. Ten windows therefore permit 160 terminal contexts plus ten
presentation contexts in one process. Electron gives each window its own
renderer process, so the comparator never meets that total. Whether Servo caps
live WebGL contexts, or serializes them on one thread, is unverified. The host
proof measures context exhaustion with a fixed number of visible panes in each
window. The maintainer then sets a process-wide cap on accelerated panes, or
records the decision to rely on the per-document pool.

### Make the actual xterm stack the first engine gate

Velocetty probes `canvas.getContext('webgl2')`. Servo 0.7.0's defaults\
disable `dom_webgl2_enabled` and `dom_intersection_observer_enabled`,\
while enabling `dom_resize_observer_enabled`. Therefore a default engine\
build does not establish the intended accelerated terminal path.\
[Servo release preferences](https://github.com/servo/servo/blob/aac43a3f31a259f04a574f5ec4e959c943ad7cc7/components/config/prefs.rs).

Create a declared Velocetty engine profile that explicitly enables and\
tests WebGL2 and, where correct, IntersectionObserver. Set each preference\
individually. The blanket experimental-feature switch is a servoshell\
command-line flag backed by a shell-private list; libservo offers no\
equivalent, and the list enables unrelated features. A preference proves\
that an API can be\
requested; it does not prove shader, atlas, context-loss, or platform\
driver compatibility. If a target cannot pass the accelerated path,\
qualify a clearly identified Canvas mode against the actual workload\
before treating it as an acceptable product mode.

Declare the profile per engine revision. Servo panics on an unknown preference
name, and names change between releases: 0.8.0 removed `layout_grid_enabled`
and renamed four layout preferences, and main removed
`dom_resize_observer_enabled` on 20 November 2026. Validate the profile against
the pinned revision's preference table at build time. Confirm that the chosen
Cargo feature set compiles WebGL at all; the `webgl` feature belongs to the
crate's web-feature group and should be checked against the selected default
features.
[Servo experimental features](https://book.servo.org/design-documentation/experimental-features.html).

Treat accelerated rendering as unproved. WebGL2 remains disabled by default at
0.7.0, 0.8.0, and current main, with open gaps tracked upstream. A public
report against the xterm.js demonstration describes working WebGL2 output in
January 2025 and broken rendering, focus, and text input in December 2025, with
no recorded resolution. Whether the pinned releases reproduce that failure is
the first question for the terminal gate. xterm's WebGL renderer also
rasterizes glyphs through a 2D canvas, so slow 2D canvas operations affect the
accelerated path. IntersectionObserver is partial upstream and marked as a 1.0
blocker. [WebGL2 tracking][servo-webgl2], [xterm.js on Servo][servo-xterm].

The Canvas fallback has no upstream future. xterm.js 5.4.0 deprecated the
unscoped `xterm` packages, and 6.0.0 removed the Canvas renderer addon in
favour of WebGL and the built-in DOM renderer. Qualifying the locked 5.3.0
stack remains the right first experiment because it is the shipped workload.
Measure the DOM renderer as a third mode in the same runs: it needs neither
WebGL2 nor a 2D canvas, and it is the fallback Velocetty inherits on any later
xterm upgrade. [xterm.js 6.0.0][xterm-6].

Run the exact locked xterm/addon combination before upgrading it. Test\
WebGL2 glyph atlases, fonts, fractional DPI, combining characters,\
wide characters, emoji, selection, cursor rendering, search,\
hyperlinks, and long scrollback. Validate cell layout and terminal\
contents as well as platform-specific screenshots. Do not require\
identical subpixel rasterization across different operating systems.

Velocetty deliberately uses Canvas for transparent terminal rendering,\
and currently enables its ligatures addon on the Canvas path. Exercise\
those paths directly. A successful opaque WebGL terminal is not evidence\
that transparent windows or ligatures work.
[Terminal implementation](https://github.com/leynos/velocetty/blob/40949b2f4b266d8523d6ffe680f1dbc53ad9b541/lib/components/term.tsx).

The locked ligatures addon (0.7.0) finds fonts through a Node module or the
Local Font Access API. A browser-safe fixture in Servo has neither. Expect
ligatures to be unsupported there until a host port supplies font data, and
report that outcome explicitly.

Hidden-pane Canvas fallback must preserve terminal state while avoiding\
unnecessary painting. xterm's hidden-render optimization depends on\
IntersectionObserver, so test the declared preference and its actual\
behaviour. Without the API, xterm 5.3.0 never pauses rendering, which wastes\
work. An observer that reports a pane as hidden and then fails to report its\
return would leave the renderer paused, which blanks a visible terminal;\
test that case specifically. Both renderers also construct a\
`ResizeObserver` unconditionally. If visibility reporting is unreliable, add\
an explicit application visibility\
adapter rather than assuming `display: none` eliminates browser work.\
Test restoration from hidden/minimized state, font and DPI changes while\
hidden, and release/reacquisition of WebGL contexts.\
[xterm render service](https://github.com/xtermjs/xterm.js/blob/5.3.0/src/browser/services/RenderService.ts).

Measure the whole context-loss interval, from the browser event to usable\
fallback, not just Velocetty's recovery callback. The pinned WebGL addon\
waits 3,000 ms for restoration before emitting its context-loss callback\
if the context remains lost. Keep terminal model state\
alive across renderer replacement and verify that no PTY output or\
selection/scroll position disappears merely because the addon changed.\
[xterm WebGL renderer](https://github.com/xtermjs/xterm.js/blob/5.3.0/addons/xterm-addon-webgl/src/WebglRenderer.ts).

### Input and native desktop behaviour

Use real keyboard and IME scenarios before throughput tuning: composed\
text, candidate positioning, dead keys, AltGr, Ctrl/Alt combinations,\
keybinding chords, paste, bracketed paste, focus changes, and terminal\
search/palette interactions. Retain the command system's ownership of\
shortcut resolution. Browser accelerators must not steal terminal input\
that the application expects to receive. xterm receives composition through\
a hidden textarea, and an open Servo defect reports a spurious\
`compositionupdate` when an input gains focus; include focus-only sequences\
in the composition tests.
[Servo composition defect](https://github.com/servo/servo/issues/49873).

Check xterm's accessibility mode with actual screen readers, including\
cursor movement, new output, focus announcements, and composition.\
Transporting an accessibility tree is necessary but does not establish\
terminal usability.

Native parity includes multiple top-level windows, native application\
and context menus, clipboard text and file-drop semantics, external-link\
actions routed through commands, geometry/fullscreen events, close\
vetoes, and correct focus after dialogs. The current application also\
uses frameless Windows/Linux windows and macOS titlebar/transparency\
features. Qualify or explicitly redesign those product behaviours; do\
not silently ignore the requested attributes.
[Window implementation](https://github.com/leynos/velocetty/blob/40949b2f4b266d8523d6ffe680f1dbc53ad9b541/app/ui/window.ts),\
[product requirements](https://github.com/leynos/velocetty/blob/40949b2f4b266d8523d6ffe680f1dbc53ad9b541/docs/velocetty-product-requirements-document.md).

### Preserve the binary service design and add real flow control

The current Electron batcher decodes chunks with a UTF-8 `StringDecoder`\
and transports strings with a session UID prefix. Its flush triggers are\
16 milliseconds or 200 KiB; these are batching triggers, not complete\
queue bounds. The target protobuf sketch uses a separate session ID and\
raw `bytes`. `Term.write` already accepts a string or a `Uint8Array`.\
[Current PTY batching][current-pty-batching],
[terminal input to xterm][terminal-implementation],\
[target protocol](https://github.com/leynos/velocetty/blob/40949b2f4b266d8523d6ffe680f1dbc53ad9b541/docs/velocetty-design.md).

Use `WebSocket.binaryType = 'arraybuffer'` and byte-preserving protobuf\
frames. Feed the resulting terminal bytes into xterm without Base64 or\
JSON-array conversion. Preserve ordering independently for each session.\
Test UTF-8 and escape sequences split at every possible chunk boundary,\
large writes, resize races, restart, EOF, and process exit. Preserve\
current behavioural contracts while making any intentional byte-level\
changes explicit.

Add application-level byte credits or acknowledgements to the planned\
protocol. The backend limits each session's outstanding output and the\
connection's aggregate outstanding bytes. The frontend returns credit\
after xterm's write-completion callback confirms parser consumption,\
not merely after a WebSocket message arrives. Parser completion is not\
proof of pixel presentation; keep those metrics separate.

When credit runs out, the backend pauses or otherwise bounds PTY reads\
according to the platform's supported mechanism. Use bounded frame sizes\
and fair session scheduling so one noisy process cannot monopolize the\
connection or the browser main thread. Give input, resize, close, and\
control messages priority over bulk output. Browser `bufferedAmount`\
alone is not an end-to-end consumption acknowledgement. In Servo it is not\
a signal at all: the implementation resets the value to zero on the next\
task because a background thread has already taken the bytes.

Credit return is new frontend work. The pinned frontend calls
`term.write(data)` without a completion callback. xterm also holds a hard-coded
50 MB write buffer and discards data beyond it, so a missing credit scheme
loses output silently under sustained load. Its flow-control guide recommends
keeping the high watermark below 500 kB.
[xterm.js flow control](https://xtermjs.org/docs/guides/flowcontrol/).

Credits bound bytes; they do not bound script-thread time. Every terminal in a
document parses on that document's one script thread, and each xterm instance
drains its own write buffer in its own time slice. Many sessions flooding at
once can queue one slice per session ahead of a keystroke, so input latency
grows with the number of noisy sessions even while every byte budget holds.
Whether Servo's task queue gives input priority over those slices is unverified.

Give the frontend one write scheduler for the document. It owns every
terminal's pending output, spends a fixed parse budget in each frame, serves
the focused pane first, and yields to input between slices. State the credit
windows as numbers once measured. Keep each session's window at or below
xterm's 500 kB guidance. Derive the aggregate window from measured parse
throughput multiplied by the agreed input-latency budget, so that draining a
full window cannot exceed that budget. Each document opens one WebSocket
connection, so the connection budget covers one window and the service budget
covers them all.

The protocol sketch does not already specify all of these credit,\
generation, cancellation, or uncertain-outcome rules. Add them when\
freezing the backend service contract. Never automatically replay PTY\
input, spawn, or restart commands after a disconnect if the backend may\
already have applied them. A write to a shell is not an idempotent HTTP\
read.

Maintain explicit connection, session, and document lifetimes. Current\
Velocetty destroys sessions on window closure and subsequent navigation.\
Preserve that behaviour in the initial migration unless the product\
explicitly adopts a different detach/reconnect policy. This design does\
not introduce durable terminal sessions or full scrollback persistence.\
The runtime must report frontend loss; the Velocetty backend decides how\
to dispose of the associated sessions.
[Current teardown](https://github.com/leynos/velocetty/blob/40949b2f4b266d8523d6ffe680f1dbc53ad9b541/app/ui/window.ts),\
[roadmap exclusions](https://github.com/leynos/velocetty/blob/40949b2f4b266d8523d6ffe680f1dbc53ad9b541/docs/roadmap.md).

### Authenticate the local service independently of Tauri

The WebSocket service is a separate privilege boundary. Tauri's invoke\
capabilities do not authorize requests that bypass invoke. Velocetty's\
backend must authenticate the connection and enforce per-command/session\
capabilities, including filesystem, process creation, and plugin storage.\
The product already requires a host-managed local token and\
permission-aware backend; the following lifecycle details are proposed\
refinements.
[Authentication design](https://github.com/leynos/velocetty/blob/40949b2f4b266d8523d6ffe680f1dbc53ad9b541/docs/velocetty-design.md).

Bind the local endpoint only to loopback. The trusted desktop bootstrap\
provides its exact endpoint and a short-lived, narrowly scoped connection\
grant to an authorized document. Keep long-lived credentials in the\
host-managed store. Authenticate before accepting commands or sending\
PTY output. Validate an exact allowed browser Origin in addition to the\
grant; Origin alone does not authenticate a non-browser client. Put\
credentials in the authentication exchange, not URLs, logs, or terminal\
output.

Specify the handshake before the service contract is frozen. A bearer grant
sent in the first frame to whichever process holds the port is replayable: a
local process that binds the port first, or races the service after a restart,
receives the grant and can present it with a forged Origin. Use a challenge and
response instead. The server sends a fresh nonce, and the client answers with a
keyed digest over the nonce, the grant identifier, and the protocol version, so
the grant itself never crosses the socket. The first client frame carries a
protocol version, and the server rejects a version it does not implement with a
structured error. Before authentication completes, bound the deadline, the
frame size, and the number of unauthenticated connections. State the grant's
lifetime as a number. Origin plus grant already defeats DNS rebinding from a
remote page.

The host registers a backend-side lease scoped to the backend instance,\
permitted capabilities, session namespace, and native document/view\
generation. The backend stores that mapping through a trusted control\
path; the native `DocumentToken` never enters JavaScript. A single-use\
bootstrap grant is consumed into a connection attached to this lease.\
The WebSocket handshake proves possession of that scoped bearer grant\
and supplies the browser Origin; it cannot independently recover Servo's\
native document identity.

Key each lease by backend instance and document identifier. Revocation is
idempotent. The application learns the document identifier and its retirement
through the runtime's extension interface described under
[contract adaptation](#document-identity-for-application-code); a Tauri command
otherwise sees only a window label, which cannot scope a lease.

Run the backend in a process separate from the engine. Servo is an in-process
engine before its first stable release, and several known defects panic. A
backend sharing its process would lose every PTY to any engine failure.
Electron keeps PTYs in its main process, away from the renderer, so a shared
process would be a regression the product has not accepted. Both the
transitional Node backend and the Rust replacement therefore use a private
inherited pipe or an equivalently authenticated local control channel. This
constrains Velocetty's backend placement; the
[terms of reference](terms-of-reference.md#open-decisions) record it for the
maintainer's confirmation.

Distinguish orderly retirement from abnormal host loss. On orderly\
retirement, the host revokes the lease over the authenticated\
supervisory channel. The backend rejects further commands and output on\
every associated connection, closes them, and destroys their PTYs under\
the existing lifecycle policy. Abnormal loss of the host channel means the\
engine process died without retiring its documents. The initial profile\
treats that as revocation too, which destroys every session the process\
owned. That is a product consequence: one engine failure ends every agent\
session in every window. The alternative holds those sessions for a bounded\
grace period so that a restarted host can reattach. It requires a\
reattachment protocol that Velocetty does not have, and the maintainer\
decides whether to adopt it. Validate lease state at\
command admission and output delivery; already accepted side effects\
cannot be undone by revocation. Bounded grant lifetimes and backend\
instance identifiers prevent old credentials from gaining authority\
after a service restart. Remote browser authentication remains a\
separate backend contract.

Use canonical `http://tauri.localhost` for the initial packaged page and\
an exact `ws://127.0.0.1:<port>` endpoint for the local service. Servo's\
source treats localhost domains and loopback IPs as potentially\
trustworthy. The design relies on those normal trust rules, not on\
disabling mixed-content checks. Test `isSecureContext`, actual emitted\
Origin, CSP `connect-src`, the authenticated handshake, forbidden remote\
endpoints, and reserved-origin routing. HTTPS/WSS remains the separate\
remote-browser deployment contract.
[Servo origin model](https://github.com/servo/servo/blob/aac43a3f31a259f04a574f5ec4e959c943ad7cc7/components/url/origin.rs).

Terminal output remains untrusted data even when the application document\
is trusted. Do not turn escape sequences, titles, links, or image metadata\
into HTML, JavaScript, or automatically authorized desktop actions.\
Clipboard and link operations must follow Velocetty's command policy.\
Arbitrary plugin JavaScript in the privileged page shares that page's\
authority; a plugin ID field alone cannot create isolation. Keep the\
product's schema-driven default and explicit trust model for custom\
plugin panels.
[Plugin and command requirements](https://github.com/leynos/velocetty/blob/40949b2f4b266d8523d6ffe680f1dbc53ad9b541/docs/velocetty-product-requirements-document.md).

### Establish measured acceptance evidence

The existing frame-timing benchmark script generates deterministic\
synthetic frame-duration and input-latency samples. It is useful as a\
metric/reporting fixture, but it does not measure Electron or Servo.\
Its thresholds express test policy rather than observed performance.\
[Benchmark source](https://github.com/leynos/velocetty/blob/40949b2f4b266d8523d6ffe680f1dbc53ad9b541/scripts/run-pty-frame-timing-benchmark.mjs).

Capture real measurements for the same locked application, fonts,\
terminal traces, hardware, and build mode under Electron and Servo.\
Report keydown-to-xterm input processing, backend write acceptance,\
echo-to-model update, presented-frame latency where measurable, rAF\
gaps, actual painted frames, output throughput, queue peaks, RSS/GPU\
memory, and idle CPU as distinct metrics. Existing keydown-to-`onData`\
instrumentation does not measure input-to-photon latency.

Test interactive shell use during a noisy background session, rapid\
fullscreen editor redraw, large output bursts, many hidden tabs,\
multiple visible panes, multiple native windows, context loss, and\
screen-reader mode. Do not invent a throughput guarantee before this\
baseline exists. Set product promotion budgets from those measurements.\
The current 500 ms fit/PTY-resize debounce also belongs to the application\
path; separate it from engine resize latency when analysing results.

Fix the measurement protocol before any Servo result exists, so that the result
cannot shape it. Roadmap task 1.2.2 records the trace provenance and rates, the
run and warm-up counts, one host-independent latency instrument, and the
budgets agreed from an Electron run and an Electron-against-Electron noise
floor. Report the 99th percentile, the maximum, and the count over a fixed
threshold beside the median: the expected failures are tail stalls from context
loss, parse starvation, and garbage collection, which a 95th percentile hides.
A credit-driven replay is a closed loop, so a slower host receives fewer bytes;
report achieved throughput beside latency and run a fixed-rate trace as well as
a saturating one. Electron spreads work across several processes and Servo uses
one, so compare proportional memory over the whole process tree and exclude the
backend service from both.

### Follow the existing migration dependencies

1. Prove the locked terminal renderer and a small React shell under\
   Servo, using recorded data and native input.

2. Freeze the backend service contract, implement the planned loopback\
   protobuf/WebSocket shell around the existing Node services, and prove\
   behaviour through that path.

3. Complete a browser-safe frontend and run it under Servo against the\
   same service. Qualify binary flow control and real GPU/input behaviour.

4. Add the Tauri desktop adapter and its authenticated bootstrap/native\
   services. Verify two top-level windows and the existing close/reload\
   behaviour.

5. Replace the Node services with Rust behind the already-tested contract,\
   including PTY lifecycle, filesystem/configuration, and plugin storage.

6. Ship and test matched frontend/runtime/backend packages, signing,\
   configuration paths, update compatibility, and rollback behaviour.

Steps two and five belong to Velocetty's existing programme. They are\
integration dependencies of this runtime design, not extra functionality\
to place inside `tauri-runtime-servo`. Full generic Tauri conformance can\
expand independently after the product's verified subset works.

## Tauri baseline and compatibility policy

Use Tauri commit\
`78f49ba3993fed5a1b921235c547731006ad2a74`, from 21 October 2026, as the\
initial integration reference. This revision belongs to the Tauri 3 alpha\
line. Its packages have different alpha version suffixes, so pin the\
workspace source revision and lockfile rather than inventing a shared\
version number.

Version 3 development happens on the upstream `v3` branch. The newest
pre-release set, published on 6 November 2026, comprises `tauri` 3.0.0-alpha.6,
`tauri-runtime` alpha.5, and `tauri-runtime-cef` alpha.7. No beta, release
candidate, or stabilization date has been published. Between the pin and the
branch head of 24 November 2026, the `tauri-runtime` sources changed by one
line and the invoke script, channel implementation, and command-line runtime
detection did not change. Advancing the pin to the alpha.6 set is therefore
expected to be cheap; confirm it in the first build task.

Open upstream proposals touch the areas this design adapts. They would turn the
channel fetch command into a capability-gated core plugin, make the channel
type opaque, add asynchronous window builders with a re-entrancy guard, and
stop the CEF runtime waiting on queued main-thread getters. Treat the channel
route and the CEF dispatch precedent as moving targets, and re-read them
whenever the pin moves. [Tauri v3 branch][tauri-v3].

Tauri 3 moves runtime selection into a runtime crate and exposes\
runtime-specific operations through extension traits. Its runtime URL\
hook removes the earlier assumption that custom-scheme representation\
depends solely on the operating system. These changes directly address\
the integration problems this project must solve. The official migration\
guide still recommends Tauri 2 for production applications; this design\
chooses v3 for a new experimental runtime, not as a production migration\
instruction. [Tauri migration guide][tauri-migration-guide],
[runtime guide][runtime-guide],
[pinned runtime contracts][pinned-runtime-contracts].

Expose a configuration and selection type named `Servo`, and a concrete\
implementation type named `ServoRuntime<T>`. The engine type remains\
`servo::Servo`. The crate path shown below is provisional. Another project\
published `tauri-runtime-servo` 0.1.0 on crates.io on 24 August 2026, and\
the registry treats hyphens and underscores as the same name, so this crate\
cannot be published as `tauri_runtime_servo`. Choose a publishable name, or\
agree a shared one with that project, before any public release; see\
[related work](#related-in-process-servo-runtimes). The intended\
application shape is:

```rust
// Proposed API shape; the repository currently contains a stub.
tauri::Builder::default()
    .runtime(tauri_runtime_servo::Servo::default())
    .run(tauri::generate_context!())
    .expect("application runtime failed");
```

Implement `Runtime`, `RuntimeHandle`, `WindowDispatch`,\
`WebviewDispatch`, `WindowBuilder`, and `EventLoopProxy`, together with\
`RuntimeInitAttrs` and conversion into `DynRuntimeInitAttrs`. Implement\
runtime-specific extension traits for both the concrete runtime and\
`DynRuntime`. Preserve the existing generic application/plugin model.

The selection type must satisfy `Default + Send + Sync + 'static`. The runtime
handle, both dispatchers, and the event-loop proxy must be `Send + Sync`, and a
`with_webview` closure must be `Send` even though the value it receives need
not be. The contract also requires a `webview_version` query and the DevTools
methods regardless of feature selection, and passes GTK pointers as untyped raw
pointers.

At this revision, Tauri owns scoped event-listener registries. The runtime\
emits the corresponding `RunEvent` values; it must not duplicate the old\
Verso listener bookkeeping. `with_webview` receives a Servo-specific\
wrapper on the owner thread. This wrapper is not a Wry object and need\
not implement `Send`.

The pinned Velocetty application runs on Electron and does not supply an
existing\
Tauri 2 compatibility obligation. Do not build a Tauri 2 adapter in the initial
programme. If a consumer\
requires one, add a distinct adapter over the same engine host and run a\
separate compatibility suite. Tauri 2 plugins do not become Tauri 3\
plugins merely because both are written against a generic runtime.

## What to carry forward from Verso

The audited runtime revision is\
`dc1c1bfeadf65f3f4e38a4a3c6c960a70b749337`. Its actual browser dependency\
is `tauri-apps/verso` at\
`572f47ca7ead69ce3e8b0c433e5cff237f0514c9`, which pins Servo from\
27 September 2025. This is distinct from the archived original Verso\
repository and its earlier Servo pin.
[Runtime manifest](https://github.com/versotile-org/tauri-runtime-verso/blob/dc1c1bfeadf65f3f4e38a4a3c6c960a70b749337/Cargo.toml),\
[fork manifest](https://github.com/tauri-apps/verso/blob/572f47ca7ead69ce3e8b0c433e5cff237f0514c9/Cargo.toml).

The existing implementation starts a window-owning `versoview` process\
for each Tauri window. A Tao loop in the parent controls a Winit loop in\
the child. The runtime cannot cleanly represent independent windows and\
webviews, and its parent lacks the real native window handles. Its\
engine fork also assembles private Servo components directly.\
[Runtime creation][runtime-creation], [controller][controller].

| Prior-art element                                        | Disposition                                     | Reason                                                                  |
| -------------------------------------------------------- | ----------------------------------------------- | ----------------------------------------------------------------------- |
| Runtime and dispatcher decomposition                     | Preserve the concept                            | It identifies the Tauri boundary that must be implemented.              |
| Main-thread dispatch, close veto, and exit behaviour     | Reimplement and test                            | These behaviours matter independently of the old process architecture.  |
| Monitor, theme, and DPI conversions                      | Audit for small-scale code reuse                | These helpers may have little engine coupling.                          |
| Example applications and plugin exercises                | Adapt into conformance fixtures                 | They provide observable requirements and migration examples.            |
| Per-window browser subprocess and controller protocol    | Remove                                          | The runtime will own native windows and engine lifetime directly.       |
| Private Servo crate imports and custom compositor        | Remove                                          | Use the current embedding boundary and rendering contexts.              |
| Browser tabs, navigation chrome, bookmarks, and panel UI | Remove                                          | They belong to applications, not a Tauri runtime.                       |
| Synthetic local `Origin` and body-in-header IPC          | Remove completely                               | They lose caller provenance and binary request semantics.               |
| Successful dummy methods                                 | Replace with real behaviour or explicit failure | Applications must be able to distinguish support from omission.         |
| Automatic browser download during `build.rs`             | Replace with explicit artefact preparation      | Builds need declared versions, integrity checks, and offline behaviour. |

The old README acknowledges that its hardcoded origin prevents reliable\
local-versus-remote capability checking. The source also encodes an\
invoke body into `Tauri-VersoRuntime-Invoke-Body` because its engine\
message omits body bytes. These are requirements to repair, not\
compatibility features to retain. [README][readme],
[request adapter][runtime-creation].

Retain applicable licence notices when copying isolated code. The\
project should start with new architecture and contracts rather than a\
renamed fork whose internal dependencies survive by accident.

The Verso runtime has not changed since the audited revision of
10 September 2026 and still targets Tauri 2.7. The original Verso repository
was archived on 8 October 2025. A Tauri maintainer describes the runtime as
outdated.

### Related in-process Servo runtimes

The design is not the first in-process attempt. The `copse-dev` project
published a crate named `tauri-runtime-servo` in August 2026. It embeds
libservo 0.5 in the application process, creates windows with Tao, targets
Tauri 2.11, and replaces Tauri's invoke script to carry IPC. Its last commit
dates from 21 September 2026 and it has no Tauri 3 port. It began as a Wry pull
request; when closing that request, Tauri maintainers stated that an
out-of-tree Servo runtime is their preferred direction because they lack the
capacity to maintain one. No first-party Servo runtime was found.
[copse-dev runtime][copse-runtime], [Wry discussion][wry-servo].

| Related work                           | Disposition                                    | Reason                                                                                        |
| -------------------------------------- | ---------------------------------------------- | --------------------------------------------------------------------------------------------- |
| `copse-dev` window and engine hosting  | Audit before the host proof                    | It already hosts libservo in process and may answer surface and event-loop questions cheaply. |
| `copse-dev` replacement invoke script  | Do not adopt                                   | A copied invoke script is the maintenance burden the transport-policy hook exists to avoid.   |
| `copse-dev` request-body limitation    | Record as corroboration                        | It independently reports that interception omits request bodies.                              |
| Published crate name                   | Resolve before any release                     | The registry name is taken; see [Tauri baseline](#tauri-baseline-and-compatibility-policy).   |
| Collaboration with the existing author | Decide when recording the maintenance decision | Shared upkeep could reduce the patch burden; divergent Tauri majors and goals may prevent it. |

## Servo baseline, retained improvements, and required deltas

### Baseline and update policy

Reproduce the October control with **Servo 0.7.0**, released on 5 October 2026,
at commit `aac43a3f31a259f04a574f5ec4e959c943ad7cc7`. Maintain a small,
reviewed patch queue for the embedding contracts this runtime requires. The
comparison snapshot of Servo main is
`69911d3671ff2c75fc0d12d2553718b16956a2b1`, dated 29 October 2026. Pin the
complete dependency graph and record the patches with the runtime release; do
not follow a moving branch in application builds. [S1][s1], [S2][s2]

Servo published 0.8.0 on 9 November 2026 at commit
`4e136d8ef31b7c4748fbfac606ca463f3efd7537`. It is the newest release. Its
public embedding sources are identical to the comparison snapshot, so the
snapshot findings in this document describe 0.8.0. It builds with Rust 1.99.1.
At the first gate, compare its compiler, patch burden, and native launch
against the control. Record a maintainable patch target before committing to
the host. Keep each result associated with the release actually tested.
[Servo 0.8.0 release](https://github.com/servo/servo/releases/tag/v0.8.0).

Servo releases monthly and maintains a long-term support line. A new line opens
every six months and receives security fixes only for an expected nine months,
on a best-effort basis. The current line is 0.6, released on 29 September 2026.
Its patch releases may carry Git dependencies and may not appear on crates.io.
The 0.7.0 control is neither the current release nor a supported line, so it
serves reproduction only. Choose the patch target between the current release
and the 0.6 line. [Servo long-term support policy][servo-lts].

Release 0.7.0 changed embedder contracts that the host must follow. It removed
`WebView::set_throttled`, so throttling follows hide and show. It removed
`notify_focus_changed` and `Servo::focus`/`blur`, so the embedder calls
`WebView::set_focused`. It added `Servo::forward_accessibility_action`.
Accessibility remains behind the `accessibility_enabled` preference.
[September 2026 engine changes][servo-september].

| Property                          | Inspected value                                         | Design consequence                                                 |
| --------------------------------- | ------------------------------------------------------- | ------------------------------------------------------------------ |
| 0.7.0 toolchain                   | Rust 1.97.1                                             | Use as the upstream control build.                                 |
| 0.8.0 and main toolchain          | Rust 1.99.1                                             | Adopting the current release also requires this compiler.          |
| Main manifest MSRV                | Rust 1.88.0                                             | Distinguish declared MSRV from the release build toolchain.        |
| Main rendering dependencies       | Surfman 0.14.0; WebRender 0.70.1; raw-window-handle 0.6 | Integrate through raw handles and the public rendering contract.   |
| Main shell/example window library | Winit 0.30.13                                           | This does not require the runtime to use Servo's shell event loop. |

The runtime workspace requires its dated nightly and Polonius flags. Build
Servo alone with its release compiler as a control. Then compile and launch the
combined dependency graph with the workspace pin. Record both compiler versions
and target triples. An incompatible pair blocks host qualification until a
reviewed pin or patch passes the same comparison.
[Workspace toolchain](../rust-toolchain.toml), [Polonius policy](polonius.md).

The workspace flags would otherwise apply to the whole Servo and mozjs graph:
the nightly compiler, the Polonius analysis, and the development code
generator. Each of the Makefile's flag sets is then a separate full engine
build, and each nightly bump rebuilds all of it, while the patch series must
still build on Servo's own compiler.
[ADR 0002](adrs/adr-0002-toolchain-scope-for-the-engine-graph.md) proposes
scoping the workspace flags to this project's crates and building the engine
with its release compiler. The first build task measures both arrangements
before the decision is accepted.

The toolchain files and manifest establish these values. Core public method
signatures in `servo.rs`, `webview_delegate.rs`, and `rendering_context.rs`
match between release and comparison snapshots. Main adds geolocation delegate
methods to `WebView`; exported field types can still change, including
`WebResourceRequest.url`. Compile the adapter against the precise chosen
revision. [S2][s2], [S3][s3], [S4][s4]

### Improvements that remove the old architectural rationale

Servo introduced public WebView handles, delegates, unified input delivery, and
interchangeable rendering contexts during early 2025. Builders subsequently
replaced much of the manual initialization sequence. In November 2025,
individual webviews acquired individual rendering contexts, enabling multiple
native windows without an embedder-maintained compositor architecture. These
changes let the new runtime depend on libservo's public embedding interface.
[S5][s5], [S6][s6]

Further improvements matter to application behaviour: refresh drivers and input
completion callbacks, asynchronous JavaScript evaluation, screenshots,
native-control delegates, clipboard integration, storage and network managers,
user-content management, and AccessKit integration. CSS Grid became enabled by
default in 2026; text selection, navigation, iframe behaviour, and SpiderMonkey
security updates also advanced substantially. These gains reduce compatibility
work, but they do not establish complete Tauri compatibility. Test the actual
application corpus and privilege boundaries. [S6][s6], [S7][s7], [S8][s8]

### Dependency boundaries

The permitted dependency edges are:

- `tauri-runtime-servo` → the pinned Tauri runtime contract and its
  window/event types.

- `tauri-runtime-servo` → a private, narrow Servo adapter module → the public
  `servo` crate.

- Window integration → the selected platform event loop and raw-window-handle
  interfaces.

- Presentation → `servo::RenderingContext` and its public implementations.

- Accessibility integration → the compatible AccessKit platform adapter and
  Servo's public tree/action APIs.

- Servo itself → its constellation, script, network, paint, layout, and
  JavaScript-engine implementation crates.

Runtime code must not import Servo's private constellation, paint, script, or
network implementation crates. A missing capability becomes a proposed public
API and a reviewed Servo patch. The public `servo::protocol_handler` module
currently exposes low-level networking types; its existence does not justify
rebuilding the engine's fetch machinery in the runtime.

The private adapter module is the engine host's only view of the patched
engine. It exposes four internal ports: document bootstrap, page-to-host
messages, document-targeted delivery, and local resources. Everything above the
adapter depends on those ports and never on a patched Servo signature. When
upstream ships its own message ports or bootstrap hook in a different shape,
the change stays inside the adapter. The build asserts the expected Servo and
Tauri patch-series revisions alongside the preference-table check, so a
mismatched engine fails at compile time.
[S4](https://github.com/servo/servo/blob/69911d3671ff2c75fc0d12d2553718b16956a2b1/components/servo/lib.rs)

### Available contracts and required additions

| Concern                   | Available in the inspected API                                                 | Required runtime or upstream work                                                                                            |
| ------------------------- | ------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------- |
| Engine/window integration | `ServoBuilder`, `WebViewBuilder`, delegates, `EventLoopWaker`                  | UI-thread ownership; ID-based command dispatch; wake coalescing; event ordering.                                             |
| Presentation              | Window, offscreen, software, and custom rendering contexts                     | Native surfaces, redraw scheduling, child composition, clipping, and coordinate transforms.                                  |
| Page initialization       | `UserContentManager` with script/style insertion                               | Guaranteed once-per-document bootstrap before authored JavaScript, including initial empty documents and non-HTML documents. |
| Host-to-page execution    | `WebView::evaluate_javascript` with asynchronous result                        | A document-targeted engine operation that validates identity at execution, plus cancellation of stale callbacks.             |
| Page-to-host messaging    | No general native message handler in the inspected public delegates            | Native message bridge with engine-derived webview, document, frame, origin, and generation provenance.                       |
| Local resources           | Custom protocol registry and resource interception                             | A proposed `LocalResourceProvider` preserving normal fetch security and cancellation semantics.                              |
| Storage                   | Engine-wide directory/temporary-storage options; cookie/local/session managers | Explicit app-profile policy; reject unsupported per-view isolation and incomplete clearing claims.                           |
| Accessibility             | AccessKit tree updates, activation, and action forwarding                      | Correct subtree grafting, native adapter integration, geometry, focus, and platform verification.                            |
| Shutdown/restart          | Final engine-handle drop performs shutdown                                     | Bounded asynchronous shutdown requires an upstream contract; in-process engine recreation is currently unsupported.          |

The public embedding surface and inspected implementations support this
division. The proposed additions are design requirements, not existing Servo
methods. [S4][s4], [S9][s9], [S10][s10], [S11][s11]

The November validation found every listed gap still open at 0.7.0, the
snapshot, 0.8.0, and main of 25 November 2026. Upstream activity is limited. A
pull request adding bidirectional message ports to the embedding API has been
open and awaiting a rebase since November 2025; its tracking issue last sought
design feedback in June 2026, and reviewers noted that Servo has no isolated
script worlds. A request for a Tauri-style bridge was closed as a duplicate of
that issue. A pull request granting custom schemes a secure context was closed
as stale in June 2026. No upstream work was found for document-start scripts,
document-targeted evaluation, bounded shutdown, IndexedDB site data, or
rendering-context replacement. Plan for a downstream patch series; do not plan
around imminent upstream delivery. [Message-port proposal][servo-message-ports],
[embedder messaging issue][servo-messaging-issue].

#### Bootstrap and local-resource requirements

Current user scripts run through `HTMLHeadElement::bind_to_tree`, which queues
a task that executes when DOM script/layout safety blockers clear. This can run
early during normal HTML parsing. It does not provide a universal
document-created hook, a once-only guarantee, frame/origin selection, or an
isolated execution world. `UserScript` accepts only a source and a file name,
and manager updates take effect on the next reload. Scripts run in every frame,
and an open defect reports that they panic the script thread in a sandboxed
frame that disallows scripts. Embedder-originated JavaScript also runs
regardless of the page's CSP or sandbox since 0.7.0. The runtime needs a
dedicated bootstrap contract instead of relying on head insertion timing.
[Sandboxed-frame defect](https://github.com/servo/servo/issues/48774). [S10](https://github.com/servo/servo/blob/aac43a3f31a259f04a574f5ec4e959c943ad7cc7/components/script/dom/userscripts.rs)

For local resources, use exact reserved HTTP(S) origins and a provider inserted
at the HTTP transport boundary before DNS/TLS. Velocetty's initial profile uses
the canonical HTTP localhost origin described above. Return provider responses
through ordinary HTTP fetch processing so CORS, redirects, response filtering,
credentials, and policy checks retain their normal positions. A reserved-origin
miss must fail closed. Worker requests must follow the same reservation rules.
This is a proposed upstream design, subject to fetch conformance tests.

The existing interceptor is unsuitable as an assumed security-preserving
shortcut. It runs before normal same-origin/CORS route selection; later
response CSP, mixed-content, MIME, and nosniff checks remain. CORS, CORP, and
redirect handling also occur inside the HTTP path that interception can skip.
This source ordering creates a verification requirement; it is not an
experimentally demonstrated vulnerability. Interception also accumulates chunks
through an unbounded channel until completion, so its chunk API does not
establish streaming or backpressure. [S11][s11], [S12][s12]

Four further properties rule interception out even as a stopgap for privileged
IPC. The intercepted request carries no body and no frame or document identity,
so it cannot serve a binary invoke or attribute its caller. An embedder
response replaces a network error that a request-side CSP or mixed-content
check has already set. Intercepted responses bypass the step that assigns
response tainting, so a cross-origin response may be exposed as a basic one.
The interceptor is held under one asynchronous lock across the embedder round
trip, which appears to serialize every intercepted fetch. The last three are
inferences from source order; each needs a test before the provider design
relies on the opposite behaviour.

Custom protocols remain useful, but registering one does not establish a
standard tuple origin. A non-special scheme receives an opaque origin.
`ProtocolHandler::is_secure` documents only mixed-content treatment; the
secure-context check consults URL trustworthiness and never the protocol
registry. Servo's origin model delegates to the URL library. Implementing
comprehensive custom-scheme origin semantics would touch more engine subsystems
than a virtual HTTP(S) transport.
[S13](https://github.com/servo/servo/blob/69911d3671ff2c75fc0d12d2553718b16956a2b1/components/net/protocols/mod.rs)

### Lifecycle and platform limits

Keep one `Servo` instance on the UI thread. Its handles use `Rc`;
process-global options use a `OnceLock`, and a second initialization panics; it
does not return an error. Preferences also remain global. Public
`spin_event_loop()` returns `()`. Final `ServoInner` drop sends shutdown and
synchronously pumps until completion, with no timeout. Cancel runtime work,
release webview/delegate references, and preserve surface lifetimes before
allowing that final drop. Do not promise in-process recovery by rebuilding
Servo after failure.
[S9](https://github.com/servo/servo/blob/69911d3671ff2c75fc0d12d2553718b16956a2b1/components/servo/servo.rs)

Rendering currently requires OpenGL/GLES methods. Attach refresh scheduling
through `WindowRenderingContext::new_with_refresh_driver` or
`RenderingContext::refresh_driver`, not the obsolete builder hook. A webview
fills its context; resizing clamps to at least 1×1. There is no public webview
context-replacement API. Whole-window surface rebinding exists, but arbitrary
child reparenting remains a separate gate. Software rendering also needs
platform validation. [S3][s3], [S14][s14]

Accessibility activation returns a subtree ID. Graft it into the application's
AccessKit tree before forwarding its updates; the documented ordering violation
can panic. Profile selection is engine-wide, and the public site-data
categories omit IndexedDB. Finally, multiprocess compilation, process
separation, and OS sandboxing are distinct: runtime multiprocess/sandbox
options default to false, and current sandbox activation explicitly rejects
Windows, ARM/aarch64, Android, iOS, RISC-V, and OHOS targets. The aarch64
exclusion covers Apple Silicon, so the sandbox is unavailable on current Mac
hardware as well as on Windows. Supported-platform claims must reflect these
limits. [S9][s9], [S14][s14], [S15][s15]

Additional baseline references: [main manifest][main-manifest],
[main toolchain][main-toolchain], [global options][global-options],
[HTTP checks][http-checks], and
[origin model](https://github.com/servo/servo/blob/69911d3671ff2c75fc0d12d2553718b16956a2b1/components/url/origin.rs).

## Component boundaries

The table assigns responsibilities for the proposed runtime, engine host, and
build companion. Start the host proof with private modules in the runtime
crate. Split the host when an independent conformance executable needs it.
Introduce the build companion when an installer needs its resource inventory.

| Component                             | Owns                                                                                                                                 | Must not own                                                        |
| ------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------- |
| `tauri-runtime-servo`                 | Tauri trait implementations, configuration validation, ID translation, events, extension traits, IPC adaptation                      | Browser UI, duplicated Tauri authorization, private Servo internals |
| `servo-host`                          | One engine, native host loop, view registry, rendering surfaces, input conversion, document registrations, resource-provider adapter | Tauri command semantics, arbitrary application business logic       |
| `tauri-runtime-servo-build`           | Compatibility metadata, resource staging, native dependency inventory, packaging validation                                          | Unpinned downloads, application signing keys, runtime policy        |
| Conformance applications and fixtures | Reproducible behavioural evidence and regression cases                                                                               | Production compatibility shims                                      |

Two downstream patch series are components in their own right. Applications
build against the patched Tauri core, and the engine carries the patched Servo,
so each series has an owner, a revision, and an upkeep decision.

| Patch series | Owns                                                                                                      | Must not own                                                              |
| ------------ | --------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------- |
| Servo series | The engine deltas this design requires, one purpose per patch, against one pinned release                 | Application-specific behaviour; changes to engine policy defaults         |
| Tauri series | Document context through invoke, resolver, channels, and listeners; the runtime-trait additions they need | Servo-specific types; any change the Wry and CEF runtimes cannot build on |

Dependencies point one way: application, patched Tauri core, runtime, engine
host, then the `servo` facade. The host never names a Tauri type; it calls the
runtime back through a port the runtime implements. The host defines
`WindowKey`, `ViewKey`, and `DocumentToken`, because it owns the registry that
backs them. The runtime defines `InvocationId` and `ResourceRequestId`. Tauri
core sees a document identity only as an opaque value behind a trait with a
default implementation, so other runtimes compile unchanged and core never
depends on this crate.

The engine host, the host loop, and `servo-host` name one component. This
document calls it the engine host; `servo-host` is its crate name after
extraction.

Keep `servo-host` workspace-private after extraction. It may depend on\
explicit public graphics/windowing crates such as Surfman and\
raw-window-handle, but only the `servo` facade may provide engine\
objects. It must not import constellation, script, layout, or internal\
compositor crates. If a required engine operation lacks a public API,\
record and implement an engine delta.

The source layout separates `runtime`, `window`, `webview`, `dispatch`,\
`platform`, `document`, `ipc`, `protocol`, `storage`, and `diagnostics`\
modules. `platform` translates native events and handles;\
`document` owns privilege lifetimes. Neither should depend on a\
JavaScript framework.

## Ownership and event-loop contract

The UI thread owns `servo::Servo`, its `WebView` handles, rendering\
contexts, native windows, and registries. Sendable runtime handles contain\
weak access to a bounded command queue and typed IDs. They never carry\
engine `Rc` values across threads.

Use distinct identities:

| Identity              | Meaning and lifetime                                                           |
| --------------------- | ------------------------------------------------------------------------------ |
| `WindowKey`           | Native-window slot plus generation; survives document navigation.              |
| `ViewKey`             | Webview slot plus generation; never inferred from a window label.              |
| Engine frame identity | A particular browsing context, supplied by the engine.                         |
| `DocumentToken`       | Opaque engine-backed document identity, scoped to a view generation and frame. |
| `InvocationId`        | One admitted command attempt within one document.                              |
| `ResourceRequestId`   | One resource request with one terminal response or cancellation.               |

Generations prevent delayed commands from reaching objects that reuse a\
registry slot or label. Labels remain application-facing names and\
capability selectors; they are not unique lifetime identities.

Implement `EventLoopWaker` using a platform event-loop proxy. A wake\
marks engine work pending and coalesces redundant notifications. After\
processing a bounded batch of host commands, pump Servo, collect\
delegate notifications, release internal borrows, and dispatch Tauri\
events. A ready frame schedules redraw of its own window.

Clear the pending-wake flag before pumping, then recheck pending work\
before sleeping. This avoids dropping a wake that arrives while the\
engine is running. Do not run a permanent polling loop when no work or\
animation is pending. Servo's refresh driver and native frame scheduling\
must determine animated work; arbitrary fixed-rate polling is not the\
default.

Tauri's synchronous APIs impose a stronger rule than merely sending\
everything to the UI thread. Some methods schedule a closure and wait for\
its result. The runtime must execute suitable owner-thread closures\
inline, or those calls deadlock. The pinned CEF implementation provides\
evidence of this requirement and of macOS modal-loop re-entrancy hazards.\
[CEF event loop](https://github.com/tauri-apps/tauri/blob/78f49ba3993fed5a1b921235c547731006ad2a74/crates/tauri-runtime-cef/src/runtime.rs).

Apply these rules:

- Execute owner-thread native getters and safe closures directly.

- Never wait on the owner thread for a response whose producer requires\
  that thread to pump Servo.

- Queue Tauri `RunEvent` callbacks and engine-mutating work that cannot\
  execute safely inside the current callback.

- Release registry locks, pending-request locks, and mutable engine\
  borrows before entering application code.

- Treat nested native modal loops as possible re-entry points. Defer\
  wake delivery that would recursively enter the platform handler.

- Return a creation error if the contract cannot synchronously create a\
  window in the current context. Do not report creation success and\
  merely log a later failure.

For engine-backed synchronous methods, use the actual engine's\
synchronous operation only where it does not require UI-thread progress.\
Otherwise the compatibility work must introduce an asynchronous core\
path, provide a documented current-state snapshot where the API permits\
it, or reject the operation. A nested event-pump workaround does not\
solve re-entrancy safely.

## Native windows and rendering

Use the Tauri CEF runtime's `tauri-winit`/GTK4 integration as the initial\
native-host precedent. The inspected version is\
`tauri-winit 0.31.0-beta.3`. Servo's minimal examples use a different\
Winit line. Share raw-handle contracts where compatible; explicitly\
adapt input events, lifecycle events, and AccessKit integration instead\
of assuming these libraries share Rust types.
[CEF manifest](https://github.com/tauri-apps/tauri/blob/78f49ba3993fed5a1b921235c547731006ad2a74/crates/tauri-runtime-cef/Cargo.toml).

The runtime creates and owns real native windows. It supplies the window\
and display handles, GTK parent/container handles where required, and\
Tauri's after-creation callback. Preserve each API's ownership convention\
for GTK references. Tauri continues to own menus and tray behaviour.

For the first application profile, attach one `WebView` and its own\
`WindowRenderingContext` to each top-level webview window. One engine\
can drive all these views. Bare native windows remain a distinct\
supported host primitive; they do not need a dummy webview.

The rendering sequence is:

1. Native input, commands, or timers cause engine work.

2. Servo notifies the runtime that a frame is ready for a particular view.

3. The runtime requests redraw for the owning window.

4. The redraw handler calls `WebView::paint()` and then presents that\
   context.

Attach a refresh driver through the current rendering-context API.\
`ServoBuilder::refresh_driver` is not the contemporary interface. Track\
physical size, logical size, scale factor, visibility, occlusion, and\
surface validity separately. Suspend presentation at zero native size;\
do not mistake Servo's minimum backing size for a visible one-pixel\
window. Resize, display migration, and GPU/surface loss need explicit\
transitions.
[Rendering contexts](https://github.com/servo/servo/blob/aac43a3f31a259f04a574f5ec4e959c943ad7cc7/components/shared/paint/rendering_context.rs).

Later child-view support gives each child its own\
`OffscreenRenderingContext` and uses the public render-to-parent\
mechanism. The runtime then owns z-order, clipping, bounds, focus,\
hit-testing, coordinate transforms, and accessibility placement. The\
first implementation may constrain children to non-overlapping\
rectangles. It must not resize several independent views through a\
single shared context.

Live reparenting remains unsupported until Servo can move or replace\
the relevant rendering-context relationship without losing the document.\
Recreating a page is not a valid implementation of a state-preserving\
reparent operation. A software context supports selected headless\
fixtures; it cannot establish real GPU-driver or compositor correctness.

## Threat model

The privileged document displays attacker-influenced bytes. A coding agent can
print anything: escape sequences, links, titles, and image payloads. Those
bytes reach the engine's text shaping, font handling, image decoding, WebGL
texture uploads, and link handling.

Servo runs in the application process. Its sandbox is unavailable on Windows
and on every aarch64 target, including Apple Silicon, and this design does not
enable it elsewhere. A memory-safety defect in the engine or its JavaScript
engine therefore yields the user's full privileges: files, credentials, and
every terminal session. Electron renders the same content in a sandboxed
renderer process, so this is a reduction in defence in depth. The maintainer
accepts that reduction explicitly, or the profile stays experimental.

| Asset                         | Threat                                              | Defence in the initial profile                                                      | Residual risk                                      |
| ----------------------------- | --------------------------------------------------- | ----------------------------------------------------------------------------------- | -------------------------------------------------- |
| Native operations             | A non-application document invokes a command        | Engine-authenticated source identity and Tauri capabilities                         | Script already running in the authorized document  |
| Pending privileged results    | A successor document receives a predecessor's reply | Validation of the target document on the script thread                              | None intended; proved by the navigation race tests |
| Terminal service              | A local process or remote page connects             | Loopback binding, exact Origin, challenge and response, backend lease               | A process running as the same user                 |
| The user's account            | Engine compromise through rendered terminal output  | Network allow-list, image addon disabled, prompt engine security updates            | Full user compromise; no sandbox                   |
| Packaged origin and resources | A request escapes to the network or another origin  | Fail-closed reserved origins inside the fetch pipeline                              | Defects in the proposed provider                   |
| Stored credentials            | Exposure through logs, URLs, or crash records       | Host-managed store; credentials excluded from URLs, logs, traces, and crash records | A compromised host process                         |

The privileged view applies a fail-closed network allow-list: its reserved
origins and the exact loopback service endpoint. It loads nothing else. The
initial profile disables the terminal image addon, which removes image decoding
of terminal-supplied data from the attack surface. Both restrictions are part
of the engine profile and of the conformance evidence.

## Document bootstrap and trusted source identity

Register bindings and all initialization scripts before a view's first\
navigation. Introduce an engine hook that creates a document identity and\
runs host bootstrap once, after the global exists and before author\
scripts execute. The hook must specify behaviour for ordinary HTML,\
XML/SVG, initial empty documents, `document.open`, navigation, and frames.\
Current `UserContentManager` does not provide that complete contract.

The runtime preserves Tauri's script ordering and\
`for_main_frame_only` flags. Installing a privileged native binding is a\
separate policy decision from running a non-privileged initialization\
script in a subframe. The initial application profile grants the bridge\
only to a registered top-level packaged document, or to the exact\
development origin when development mode explicitly permits it.

The engine supplies the calling document's identity, frame identity,\
current URL, effective origin, and lifecycle state on each message.\
Opaque and inherited origins retain their actual engine meaning. The\
runtime never substitutes the destination URL, a JavaScript field,\
`window.location`, a referrer, or the last top-level URL notification for\
that source identity.

Provenance identifies the realm that owns the binding it was called through. It
cannot distinguish the top document from a same-origin descendant: a same-origin
`about:blank` or `srcdoc` child can call `parent.ipc.postMessage`, and the
engine reports the parent as the caller. Same-origin descendants of an
authorized document are therefore inside its trust boundary. The
main-frame-only rule stops the bridge being installed in a frame; it does not
stop a frame the application created from reaching its parent. The packaged
profile sets a `frame-src` restriction so that the application creates no
frames it did not intend.

`DocumentToken` is an opaque native capability to address a document,\
not a secret that JavaScript can mint or a JSON structure that the host\
deserializes. Its implementation includes engine-instance and\
view-generation scope. Public diagnostics may expose a redacted\
correlation ID, but the transport treats that diagnostic value as\
untrusted.

Before Tauri dispatches privileged work, the runtime checks the live\
document registration and frame/origin policy. It then supplies the genuine\
caller URI and token to Tauri's existing invoke handler. Tauri still\
checks its invoke key and resolves permissions using the configured\
window/webview capabilities. Source authentication complements those\
checks; it does not replace them.

The design does not protect an application's privileges from malicious\
JavaScript that already executes within its authorized document. CSP,\
safe application code, and command scopes still matter. Nor does an\
in-process JavaScript bridge form an OS boundary against a native engine\
compromise.

## IPC and asynchronous delivery

This is the generic runtime IPC contract. Velocetty's PTY bytes and\
backend command bus remain on its planned protobuf/WebSocket transport.\
Do not make completion of every generic channel feature a prerequisite\
for an unprivileged terminal-rendering prototype. A product slice may\
claim only its verified desktop bridge while broader runtime operations\
remain explicitly unsupported.

Use a native `window.ipc.postMessage` binding for ordinary JSON invokes.\
Continue using Tauri's protocol transport for raw byte invokes and\
large channel payload retrieval. This preserves the difference between\
`InvokeBody::Json` and `InvokeBody::Raw`, rather than silently converting\
bytes into a JSON array.
[Tauri transport script](https://github.com/tauri-apps/tauri/blob/78f49ba3993fed5a1b921235c547731006ad2a74/crates/tauri/scripts/ipc-protocol.js),\
[payload serialization](https://github.com/tauri-apps/tauri/blob/78f49ba3993fed5a1b921235c547731006ad2a74/crates/tauri/scripts/process-ipc-message-fn.js).

Add a small runtime transport-policy hook to Tauri and its dynamic\
adapter. Tauri's own generated JavaScript selects the transport before\
dispatch. The hook, its enum, and the generated-script adjustment are\
proposed core changes; no such selector exists in the inspected\
revision. Do not carry a local copy of the entire invoke script.

Attach `DocumentToken` and authenticated source context through native\
request metadata, using `http::Request` extensions at the existing\
handler boundary. The core adaptation captures that identity in the\
invoke request, resolver, channel, and relevant event subscription.

The complete delivery path is:

1. The engine authenticates the calling realm and emits its document\
   token with the message or protocol request.

2. The runtime checks policy, capacity, and lifecycle state.

3. Tauri checks its invoke key and capabilities and starts the command.

4. The resolver retains the original token while Rust work proceeds.

5. A completion uses a new document-targeted evaluation/delivery method.

6. Servo validates and resolves that token on its script execution\
   endpoint before executing the reply.

Step 6 is essential. A UI-thread epoch check followed by ordinary\
evaluate-in-current-webview still has a race with navigation.

Current Tauri native responders and channels use evaluation against the\
current webview. The runtime cannot infer the missing original document\
from that call. Add `eval_script_for_document` to the runtime contract\
and dynamic dispatch, and propagate the token through the core.\
Preserve ordinary application-requested `Webview::eval` as an explicit\
current-view operation.
[Current channel implementation](https://github.com/tauri-apps/tauri/blob/78f49ba3993fed5a1b921235c547731006ad2a74/crates/tauri/src/ipc/channel.rs).

The proposed operation has two outcomes, reported separately:

```rust
// Proposed shape; names are provisional.
fn eval_script_for_document(
    &self,
    target: &DocumentToken,
    script: String,
    on_executed: Box<dyn FnOnce(Result<(), DeliveryError>) + Send>,
) -> Result<(), AdmissionError>;
```

The return value reports admission: the queue accepted the work, or refused it
for capacity or an already-retired target. The callback reports execution on
the script thread, including `DocumentExpired`. Each admitted evaluation runs
at most once. Evaluations for one document run in admission order, and in order
relative to ordinary `eval` calls for the same view, because events and channel
messages depend on that ordering. Tauri's invoke resolver returns nothing to
its caller, so an expired delivery has no recipient: the runtime drops the
reply, emits a structured trace event with the invocation identifier, and
increments a counter. It never retries against another document.

The bridge is not the only way to carry an invoke. Tauri's protocol route
already separates JSON from raw bodies, and its reply returns through the
fetch, which the engine binds to the initiating document without any new
delivery operation. A third option removes the in-page Tauri bridge altogether
and carries desktop services over the authenticated loopback service.
[ADR 0001](adrs/adr-0001-desktop-service-transport.md) compares the three and
proposes deciding between them before any engine patch is written. The
remainder of this section specifies the native-bridge option.

Channel payload caches must include view generation and document token\
in their keys. A new document must not redeem an old payload using the\
same window label, callback number, or channel identifier. Retire\
subscriptions and cached data when their document retires. A channel\
created deliberately from Rust must capture an explicit document or\
document its current-view semantics; it must not acquire an implicit\
permission to follow navigation.

Navigation cancels unadmitted requests and result delivery. It does not\
undo a command's already-executed file writes or other side effects.\
Application work cancellation requires cooperation from the command.

Choose the transport deterministically. Do not resend a possibly\
dispatched side-effecting command merely because response delivery\
failed. If a later implementation needs cross-transport retries, add a\
bounded deduplication protocol keyed by document and invocation identity,\
with explicit uncertain-completion behaviour. The initial contract\
reports transport failure after possible dispatch instead.

## Virtual origins and application resources

Represent registered application protocols using\
`http[s]://{scheme}.localhost` consistently on all platforms, honouring\
Tauri's `useHttpsScheme` setting. Use `http://tauri.localhost` for the\
initial Velocetty profile, alongside its authenticated loopback\
WebSocket connection. Reserve the virtual origin inside the engine; no\
HTTP server serves those packaged assets. Test localhost trust and the\
actual WebSocket/CSP path in the first engine experiment. Do not silently\
change a deployed application's origin and storage identity.

Implement Tauri's `custom_scheme_url(scheme, https)` contract exactly,\
including interpolation of the literal `{protocol}` placeholder used by\
`convertFileSrc`. Validate real protocol names at registration, not in\
the template-expansion function. An HTTPS profile must separately qualify\
its local WebSocket or WSS policy. Remote browser deployments use normal\
HTTPS/WSS; this local profile does not relax their transport requirements.\
[Runtime URL contract](https://github.com/tauri-apps/tauri/blob/78f49ba3993fed5a1b921235c547731006ad2a74/crates/tauri-runtime/src/lib.rs).

This uses the browser's existing tuple-origin model and fits Tauri 3's\
runtime hook. Registering an arbitrary Servo scheme and marking it\
secure would not, by itself, establish tuple origins, secure contexts,\
storage semantics, or service-worker behaviour.

Introduce a public engine `LocalResourceProvider`. The provider owns\
the transport for exact registered virtual origins before DNS/TLS, while\
returning responses through the ordinary fetch processing that enforces\
CSP, CORS, preflight, credentials, redirects, response filtering, and\
cache rules. The name is a proposal; this interface does not currently\
exist.

Do not implement this boundary by simply responding through current\
`load_web_resource`. Source inspection shows interception before the\
normal same-origin/CORS selection and outside the usual HTTP-fetch\
checks. Some later response checks remain. This is an ordering risk\
that requires conformance tests, not an assertion that an exploit was\
demonstrated. [Fetch implementation][fetch-implementation],
[HTTP loader][http-loader].

The provider contract must include:

- Method, destination URL, headers, request bytes, and redirect state.

- Engine-authenticated initiating context when a request has one,\
  including frame/document identity or a distinct worker context.

- Response status, headers, bytes, terminal completion, and cancellation.

- Admission limits, total buffer budgets, and an observable request\
  lifecycle.

- Global routing for worker and other requests that lack a webview\
  delegate.

Map these requests to Tauri's registered protocol handlers, including\
the bundled asset handler, scoped filesystem asset handler, user\
protocols, and binary IPC/channel requests. Native JSON IPC does not\
eliminate this generic protocol requirement. A remote document requesting\
an app-origin resource remains a remote caller.

Reserve exact canonical scheme/host/port origins. Do not grant authority\
to the whole `*.localhost` suffix. Reject protocol collisions, unexpected\
userinfo or ports, malformed hostnames, and unauthorized redirects.\
Normalize URLs once using the engine's URL model and retain unambiguous\
path treatment. Missing resources and dropped handlers fail closed for\
reserved origins; they never fall through to networking.

The initial profile disables service-worker interception for app-owned\
origins and gives privileged IPC responses `no-store` treatment. Asset\
caching, if enabled, keys entries by application build identity and\
profile as well as the ordinary request semantics. Enable service\
workers only after controller selection, cache replacement, worker\
identity, and update behaviour have dedicated tests. This is a\
compatibility restriction, not a claim that service workers cannot be\
implemented.

Preserve status codes, MIME type, CSP and other security headers,\
`HEAD`, conditional requests, and single-range responses. Specify invalid\
or unsupported range behaviour rather than returning an unrelated full\
success response. Never widen Tauri's asset scopes or emit permissive\
cross-origin headers to make loading succeed.

Tauri's current custom-protocol API buffers requests and responses.\
Servo's current interception implementation also accumulates response\
chunks. Accordingly, the initial implementation promises bounded\
buffering, not constant-memory streaming. Genuine streaming needs a\
separate end-to-end contract. A response limit can bound runtime-owned\
copies; it cannot stop an application callback from allocating its own\
oversized body before returning it.

## Navigation and frame policy

Register allowed packaged origins and exact development origins before\
first navigation. A development URL retains its actual origin and\
network transport; it does not acquire a synthetic packaged identity.\
Release configuration must reject a leftover development-origin grant.

Use an explicit view/document lifecycle:

| State or transition                | Required behaviour                                                                       |
| ---------------------------------- | ---------------------------------------------------------------------------------------- |
| View allocated                     | No application IPC; all bootstrap and origin policy installed before load.               |
| Initial empty document             | No privilege by default, even when it has an inherited origin.                           |
| Authorized document created        | Mint its token and install its permitted bridge before author code.                      |
| Same-document history change       | Retain the document token; use the current engine URL when policy is path-sensitive.     |
| `document.open` on a live document | Retain the token only if the engine keeps the same document; otherwise retire and mint.  |
| History entry restored from cache  | Treat as a new document: retire any retained registration and bootstrap again.           |
| Cross-document navigation          | Retire the old registration and create a separate identity for the new document.         |
| Frame detached or view replaced    | Cancel pending delivery, subscriptions, and payload redemption for the retired identity. |
| Closing or crashed                 | Reject new work and settle native pending operations exactly once.                       |

Default to denying remote top-level navigation inside privileged views.\
The application can open ordinary external links through its configured\
system-browser integration. Disable unapproved remote frames, remote\
script sources, automatic popup creation, and bridge inheritance to\
`about:blank`, `srcdoc`, blob documents, and workers in the initial\
profile. Future support must account for the effective origin and source\
context of each case rather than using URL spelling as identity.

The supported profile must either disable back/forward-cache document\
restoration or revoke and reauthorize restored documents with an explicit\
engine lifecycle contract. Do not assume that a navigation counter alone\
describes retained or restored documents. The v0.1 gate must verify which\
mechanism the chosen engine revision actually supports.

## Storage and process model

Browser storage and Velocetty terminal profiles are different concepts.\
Shell, environment, working-directory, and JSON5 configuration profiles\
remain backend-owned application data. The runtime's one-browser-profile\
limit does not restrict those terminal configurations.

Configure one persistent application-specific browser profile, or an explicitly\
ephemeral engine-wide session, before constructing Servo. Acquire an\
exclusive profile lock before any engine writes. The single-instance\
plugin is optional application behaviour and does not replace this lock.

Use an operating-system advisory lock that the kernel releases when the process
dies. A lock expressed as a file's existence would survive a crash and block
the relaunch, and this architecture makes crashes more likely than the Electron
host did.

Velocetty may not need a persistent browser profile at all. Its configuration,
plugin storage, and credentials belong to the backend and the host-managed
store. If the frontend keeps nothing in cookies or web storage, the reference
application runs with the ephemeral profile, and the profile migration,
rollback, and stale-lock hazards disappear for it. The native-service inventory
answers this before any migration machinery is built.

Ephemeral mode must not reuse a persistent `config_dir`. The inspected\
network state loader/writer uses that directory for authentication,\
cookies, and HSTS state without relying on `temporary_storage` alone.\
Use no persistent directory, or a fresh disposable whole-profile\
directory with explicit cleanup, alongside the appropriate storage\
settings. Test that an ephemeral session neither reads existing network\
state nor leaves it behind.
[Network-state persistence](https://github.com/servo/servo/blob/aac43a3f31a259f04a574f5ec4e959c943ad7cc7/components/net/resource_thread.rs).

Reject conflicting per-view data directories, per-view incognito requests,\
storage identifiers, or global engine preferences. Multiple windows\
share the app profile and normal origin-based storage; they do not share\
document tokens or callback namespaces. Existing Servo globals prevent\
using several independent `Servo` instances in one process as a profile\
workaround.

Map cookie and supported site-data operations to the actual manager\
APIs. Do not claim comprehensive browsing-data clearing while public\
storage controls omit classes such as IndexedDB. Unsupported full-clear\
operations must fail visibly, or a future engine API must implement\
them. Profile schema changes and downgrade behaviour belong in the\
compatibility manifest. Preserve a recoverable backup before a declared\
destructive migration.

The first runtime embeds Servo in the application process. Its engine\
threads remain distinct from the platform event loop, but no separate\
renderer-security claim follows from that fact. Current Servo\
multiprocess compilation, content-process startup, and sandbox support\
are separate mechanisms with platform restrictions. A future process\
split needs explicit brokered capabilities, source identity, surface\
transport, crash recovery, and per-platform sandbox verification.

Do not resurrect one browser subprocess per native window. If isolation\
becomes necessary, preserve app-owned native windows and design engine\
hosts around declared profile/trust boundaries. That is a separate\
architecture stage with its own cost and API constraints.

## Native services and accessibility

Translate keyboard, pointer, wheel, touch, focus, theme, cursor, and IME\
events through public Servo APIs. Preserve physical/logical coordinate\
conversions and input completion semantics. Forward native composition\
updates, committed text, candidate-window placement, dead keys, and\
keyboard-layout changes. Do not synthesize text from keycodes alone.

Use Tauri's menu/tray facilities and preserve native dialog parenting.\
Handle Servo requests for file selection, clipboard access, permissions,\
authentication, downloads, and popups through a small host policy layer.\
Every outstanding responder receives one answer or a defined\
cancellation when the parent disappears. The initial profile denies\
unsupported camera, microphone, Bluetooth, notification, and similar\
permissions rather than treating absence of UI as consent.

Clipboard browser permissions and Tauri clipboard-plugin permissions\
remain separate paths. User activation and origin policy must apply to\
the web API even if the application can use its own Rust-side plugin.

Integrate Servo's accessibility tree into the native window tree. Activate\
the view, obtain its subtree identity, graft it into the parent tree, and\
only then forward incremental updates. Route native accessibility actions\
back to Servo. Keep coordinate transforms synchronized with DPI and\
child-view placement. Verify actual screen-reader interactions on every\
supported desktop platform; a generated accessibility tree is not a\
complete accessibility claim.

## Failure and shutdown semantics

State the cost of each failure plainly. The engine, every document's script
thread, and every window share one process.

| Failure                        | What is lost                                           | What survives                                      | Recovery                                                   |
| ------------------------------ | ------------------------------------------------------ | -------------------------------------------------- | ---------------------------------------------------------- |
| WebGL context loss in one pane | That pane's accelerated renderer                       | Terminal state, session, other panes               | Fallback renderer after xterm's wait; reacquire on restore |
| Presentation surface loss      | Frames for one window until the surface is rebuilt     | Documents, sessions, other windows                 | Rebuild the surface; report a terminal failure if it fails |
| Document reload or navigation  | That document's sessions, under current product policy | Other windows and their sessions                   | New document, new lease, new sessions                      |
| Script-thread panic            | Unverified: possibly the whole process                 | Unverified                                         | Measured by fault injection before any privileged work     |
| Engine panic or abort          | Every window and, by lease revocation, every session   | Backend process; backend-owned configuration       | Restart the application; sessions are gone                 |
| Stalled engine shutdown        | Nothing further, if the order below holds              | Sessions already settled; profile already released | Process-exit watchdog                                      |
| Unknown preference at start-up | The launch                                             | Profile and previous bundle                        | Build-time preference validation; failed-launch counter    |

The engine-failure row is the significant regression from the Electron host,
where a renderer crash leaves PTYs alive in another process. Keeping the
backend out of the engine's process preserves the PTYs; whether sessions then
outlive the host is the reattachment decision recorded under
[local service authentication](#authenticate-the-local-service-independently-of-tauri).

An admitted native operation owns a one-shot completion. View retirement,\
timeout, shutdown, or a disconnected worker resolves it once and removes\
its registry entry. Dropping an internal request must not accidentally\
become network fallback or successful partial completion.

Differentiate a requested close from destruction. Emit the Tauri\
close-request callback before committing to closure, honour its veto,\
and resolve any supported engine unload decision. Once closure commits,\
stop admitting work, retire document registrations, cancel resource requests,\
drop view handles, release their rendering contexts, and finally destroy\
native windows. Emit destruction once. Application exit remains separate\
from closing one window; a tray app may continue without a visible view.

Current Servo teardown occurs in a blocking destructor and pumps until\
engine shutdown completes. A timer around dropping it does not create a\
non-blocking or safely cancellable shutdown. The host proof must measure\
ordinary teardown and exercise pending work. A bounded-shutdown claim\
requires an explicit engine begin/poll/complete API; otherwise the alpha\
documents the blocking limit. Never terminate an individual engine thread\
and reuse its process state.

Order application exit so that a stalled destructor costs nothing durable.
First settle or hand back terminal sessions through the backend. Then flush
runtime-owned state. Then drop the final engine handle, and release the profile
lock only after its destructor has returned, because the engine's own writers,
such as the network state writer, may persist into the profile until then. Arm
a watchdog when the handle is dropped: if the destructor has not returned
within a declared interval, exit the process, which releases the advisory lock.
Exiting a process is safe where terminating one engine thread is not, because
nothing reuses the process state. Record the observed teardown time and every
watchdog firing.

On a view crash, show or emit a recoverable error only if the engine\
remains in a verified usable state. A fatal engine failure requires\
application-process restart. Process-global Servo initialization prevents\
assuming that a new engine can simply replace the failed one in-process.

Add proposed Tauri errors for unsupported operations, expired documents,\
capacity exhaustion, and runtime failure where appropriate. The inspected\
runtime enum does not already contain a generic `Unsupported` variant.\
Until the core change lands, boxed creation errors support configuration\
rejection. Required methods returning `()` need documented diagnostics\
and preflight checks; their signatures cannot return a typed error.

Initial proposed limits are 1 MiB per JSON invoke, 16 MiB per request\
body, 64 MiB per protocol response, 128 pending invocations per view, and\
128 MiB of aggregate runtime-owned protocol buffers. They are design\
defaults, not measured optimal values. Check limits before allocation\
where the engine supplies the necessary admission point; reject excess\
work rather than accumulating unbounded queues. Permit explicit app\
configuration and test it. Do not impose a universal command-execution\
timeout on legitimate long-running application work.

These defaults do not yet compose. One view's 128 pending invocations at 1 MiB
each equal the whole aggregate budget, and two maximum responses exhaust it, so
one window can starve nine others. Add a per-view byte quota below the
aggregate, and reserve a share of the aggregate that no single view can
consume. The aggregate also understates resident memory. A buffered response
exists in Tauri's handler, in the runtime's copy, in Servo's accumulation, and
in the script's buffer, so plan for roughly three to four times the nominal
figure until measurement replaces the estimate.

The runtime owns aggregate admission for its command and resource paths across
all windows. Charge queued and in-flight bytes until the final consumer
releases them. Reserve scheduling capacity for cancellation, close, and engine
progress even at the data limit. The terminal service owns a separate budget
across its connections. Charge output against both session and service limits
until the agreed acknowledgement. Exercise a stalled consumer and a second
interactive window, then close both. Measure peak outstanding bytes, close
latency, and recovery of credit before accepting defaults.

## Diagnostics and developer experience

Expose engine and runtime versions, source revisions, patch-set identity,\
enabled features, graphics backend, and actual supported capabilities.\
Report the profile mode without revealing filesystem contents or secrets.

Use structured tracing for window/view/document lifecycle, wake/pump/\
paint timing, queue depth, resource-request completion, invoke admission,\
and rejected delivery. Logs contain correlation IDs and rejection reasons,\
not command bodies, credentials, file contents, or invoke keys.

Provide a capability report before application launch. Separate\
implemented operations, platform restrictions, deliberate exclusions,\
and engine/core blockers. A successful build is not evidence that a\
plugin's behaviours work.

DevTools remain an opt-in development facility. Use Servo's supported\
Firefox debugging integration and bind any server to loopback. Shipping\
debugging capability and enabling a listening endpoint are separate\
settings. Do not promise an in-app Chromium DevTools panel or Chrome\
DevTools Protocol compatibility.

Tracing serves a developer at the machine. The maintainer also needs to learn
of failures on other people's machines without collecting terminal contents.
Install a panic hook that writes a bounded local crash record: runtime, engine,
and patch-series revisions, target, GPU and driver, the failing thread, and the
panic location. It contains no terminal contents, command bodies, paths under
the user's home, or credentials. At the next launch the application offers to
submit it; nothing leaves the machine without consent. Provide a user-triggered
diagnostics bundle with the same exclusions and the capability report.
Libraries must not install global hooks, so the application installs this one
at its composition root.

Four artefacts carry these facts, and each has one meaning. The *compatibility
manifest* declares what a build is: its revisions, patch series, features, and
supported operations. The *capability report* is the manifest's
supported-operation list as the running build reports it. An *evidence record*
holds the outcome of each conformance gate for one build and target. A *crash
record* describes one failure on one machine. The [context](context.md) defines
them for the whole documentation set.

## Packaging and upgrades

Bundle the engine selected by the application. Use the public Servo\
crate's bundled-resource support where appropriate and stage remaining\
native libraries, media dependencies, licences, and platform resources\
explicitly. Do not depend on a user's installed browser or on runtime\
downloads.

The Tauri v3 CLI already classifies third-party runtimes as `Other` and\
performs no Wry/CEF-specific bundling. The initial build companion can\
use that path and normal resource configuration. It must add Servo's\
requirements itself. A packaging hook may improve this later; a CLI fork\
is not necessary merely to suppress WebView2 installation.\
[Runtime guide][runtime-guide], [manifest detection][manifest-detection].

Publish one compatibility manifest containing:

- Runtime source revision and version.

- Servo release, base SHA, patch-series revision, and feature selection.

- Tauri workspace SHA, Tauri patch-series revision, and frontend package
  versions.

- A schema version for the manifest itself.

- The product-subset inventory: every runtime operation, plugin command,
  event, and protocol form, each marked required, unsupported, or excluded.

- Rust toolchain, target triple, native ABI baseline, and compiler/linker\
  settings that affect distributed artefacts.

- Resource and binary checksums, native dependency inventory, and notices.

- Storage-format compatibility and the conformance result identifiers.

Do not promise that a prebuilt Rust `rlib` is a stable SDK independent of\
toolchain, target, and dependency graph. Initially reuse ordinary Cargo\
caches and trusted native dependency binaries. A stable C ABI or dynamic\
engine SDK would be another project, not a packaging shortcut.

The Tauri series changes types that every application and plugin compiles
against. An application therefore builds against the patched core through a
Cargo `[patch]` section that names the pinned revision of the series; the
runtime crate alone is insufficient. The Wry and CEF runtimes must still build
against the patched traits, which the series' own checks verify. Rebase the
Tauri series whenever the pin moves, and classify each patch as an upstream
candidate or a permanent carry. Tauri accepts contributions that Servo's policy
excludes, so the upstream-candidate class is real for this series.

`build.rs` validates and stages prepared inputs. An explicit preparation\
command may fetch versioned artefacts with digest verification and a\
clear offline mode. Reproducible builds must not silently fetch the latest\
browser or patch an unpinned dependency graph.

Linux packages declare their actual GTK4, graphics, libc, and optional\
GStreamer requirements. Windows packages declare the actual graphics and\
native runtime dependencies. macOS bundles account for engine/JIT needs,\
code signing, notarization, and the entitlements established by tested\
builds. Do not copy Chromium's entitlement list without validating Servo's\
requirements. Test installation and launch outside a developer checkout.

Stage an update beside the last working bundle. Under the exclusive profile
lock, check the stored format against the candidate manifest before the first
write. If migration is required, checkpoint the profile and retain the matching
old bundle until candidate launch and profile health checks succeed. On
interruption, recover one compatible bundle/profile pair. Exercise interruption
before and after the first migrated write and verify the previous user data on
restart. Stop an incompatible downgrade before opening the profile for writes.

Define health so that recovery has a trigger. A launch is healthy when the
first frame is presented, the bootstrap grant is redeemed, and one terminal
session echoes input. Before initializing the engine, increment a failed-launch
counter in a file outside the browser profile; clear it when the launch becomes
healthy. When the counter reaches its limit, start the retained bundle, or
start a safe mode that selects the DOM renderer and software rendering. An
engine update that fails graphics initialization on one driver family then
degrades to a working terminal instead of a launch loop.

### Engine security policy

A bundled engine makes the application responsible for the engine's
vulnerabilities. Servo publishes monthly, the support line receives best-effort
security fixes for about nine months, and the 0.6 line therefore expires around
June 2027. The patch series cannot go upstream, so every engine update is a
rebase.

- The workshop watches Servo and SpiderMonkey advisories and each monthly
  release, and reports findings to the maintainer on a fixed cadence.
- The maintainer declares a patch-latency budget: the longest interval
  between an upstream security fix and a released build containing it. The
  carry rehearsals measure whether the team can meet it.
- The update manifest is signed and carries a minimum safe version, so an
  installed build learns that it has been withdrawn.
- Withdrawal has a destination. While the Electron host remains shippable,
  withdrawing a Servo build means updating its users to the Electron build.
- If the budget is exceeded, or the maintainer is unavailable beyond it, the
  profile reverts to experimental and receives no new installations.

Track Servo security updates independently of feature work. Keep engine\
changes as a small, documented patch series with one purpose per patch\
and upstream tracking where appropriate. Rebase through the conformance\
suite. Compare a current monthly release with the long-term support line\
when their required embedding contracts match; do not freeze an obsolete\
engine merely to avoid adapter maintenance. A support line receives\
best-effort security fixes for about nine months and may depend on Git\
sources, so following it still means a rebase at least twice a year and a\
dependency policy that admits pinned Git revisions.

Servo's current contribution policy excludes LLM-generated code,\
documentation, issues, and comments, with an exception for minimized test
cases. Therefore upstream acceptance is a\
delivery dependency that cannot be satisfied by submitting this generated\
design or generated patches. Arrange contributions that comply with the\
project's policy, or explicitly fund and maintain a downstream patch\
series. At df12 Productions, the sole human maintainer decides which patches to
carry and whether to release the combined graph. The agent workshop can prepare
and exercise the candidate. Rehearse one security update, time the rebase and
conformance run, and demonstrate recovery of the previous compatible
bundle/profile pair. Keep the profile experimental if that work lacks an owner
or exceeds the release owner's agreed update window.
[Servo contribution policy](https://book.servo.org/policy/ai-usage.html).

## Required engine and core work

The adapter can be developed before every change lands. Velocetty's\
minimum privileged bridge requires early bootstrap, authenticated source\
identity, exact-document JSON replies, normal-policy packaged resources,\
and every native service/event used by its chosen plugins. Broader raw\
invoke and channel compatibility can follow when no selected dependency\
requires it. The table distinguishes those scopes.

| Change                                                                | Owner                                              | Required scope and acceptance condition                                                                                                                                               |
| --------------------------------------------------------------------- | -------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Guaranteed document bootstrap and native messaging                    | Servo                                              | Product minimum: correct timing and frame policy across supported document creation paths; engine-authenticated source context.                                                       |
| Exact-document execution/delivery                                     | Servo                                              | Product minimum: stale tokens fail at the script execution endpoint; navigation cannot redirect a bootstrap credential or native-service reply.                                       |
| Policy-preserving local resource transport                            | Servo                                              | Product minimum: packaged assets and required request forms retain fail-closed routing and ordinary fetch security. General runtime: all declared custom-protocol methods and bodies. |
| Document context through invoke, events, channels, and payload caches | Tauri core and runtime traits                      | Product minimum: JSON replies and all events/plugins actually used. General runtime: channels and payload redemption also remain attached to the initiating document.                 |
| Explicit transport policy                                             | Tauri core                                         | Product minimum: deterministic native JSON selection without blind retry. General runtime: protocol-byte and channel-data routes satisfy the same dispatch rules.                     |
| Honest unsupported/failure reporting                                  | Tauri core and adapter                             | Every supported profile: preflight and operation errors expose unsupported behaviours; no security-sensitive successful no-ops.                                                       |
| Non-blocking shutdown                                                 | Servo, if a bounded-shutdown guarantee is required | Begin/poll/complete lifecycle supports progress and diagnostics without blocking the host loop.                                                                                       |

These are proposed changes, not claims that upstream has accepted their\
names or signatures. Keep them generic enough for other embedders, but\
prove them with concrete Tauri scenarios before generalizing their APIs.

Every Servo row is a permanent downstream carry under the current contribution
policy unless the maintainer writes and submits it personally. Each Tauri row
is either an upstream candidate or a permanent carry, and the maintenance
decision classifies it. The Tauri rows touch the invoke, resolver, and channel
code that open upstream proposals are changing, so expect them to conflict at
each pin move. The error-reporting row needs no core change if the runtime
publishes its own error type, as
[contract adaptation](#document-identity-for-application-code) describes.

The table is the cost of the native-bridge option. Carrying invoke on the
protocol route removes the messaging and transport-policy rows and leaves
targeted delivery for events alone. Carrying desktop services over the loopback
service removes every row above the shutdown row.
[ADR 0001](adrs/adr-0001-desktop-service-transport.md) records the comparison.
Spike each delta that the chosen option retains before accepting its upkeep:
record the engine subsystems it touches and its size, then carry the smallest
one across a release.

## Contract adaptation notes

The names below describe proposed interfaces. The pinned Tauri contract is
commit `78f49ba3993fed5a1b921235c547731006ad2a74`. Qualification requires a
compiled adapter and the behavioural evidence specified below.

### Trusted document identity

The runtime creates `SourceContext` from the engine's actual calling document.
Its full source URL supports URL-sensitive capability checks; its origin
identifies the browser principal. Neither a URL in the frontend envelope nor an
unverified `Origin` header establishes provenance. Opaque origins require
explicit policy and cannot acquire local application privileges by serializing
as `null`.

`DocumentToken` holds a weak reference to a private identity record. The live
document registry revokes that record before removing it when the document
retires. An in-flight upgraded reference must not keep authority alive. Tokens
retain no engine document, JavaScript realm, GPU surface, window, or raw native
pointer. Labels and native handles can be reused; document identities cannot.
Engine target generations must fail on exhaustion. The weak reference is a
lifetime check, not sufficient execution authorization: Servo must validate the
exact document/context identity on its script thread before executing a
targeted operation. Opaque origins retain distinct engine identities even
though their browser serialization is `null`.

### Tauri adaptation

The existing native handler receives `http::Request<String>`. The adapter
inserts `SourceContext` into the request's extensions, outside the frontend
JSON envelope. Core retains it through `InvokeRequest`, `InvokeMessage`, the
invoke resolver, channels, and frontend event-listener registrations. Add
document-targeted evaluation to the runtime and its dynamic adapter. Keep
ordinary `Webview::eval` explicitly directed at the current view. Admission and
execution have separate acknowledgements. The execution completion reports
`DocumentExpired` when the target no longer matches; it never falls back to
that current view.

Channel payload caches need keys that include both the webview instance and
document identity. Retirement revokes delivery, purges pending payloads, and
rejects further ingress from that document. It cannot reverse a Rust command's
completed effects. Cancelling application work requires a separate cooperative
mechanism. Rust-created channels should offer an explicit document-targeted
constructor.

On the protocol route, Tauri core currently takes the caller from the request's
`Origin` header. `SourceContext` must override that header wherever core
derives a caller, or a forged header would defeat the engine's identity.

### Document identity for application code

Application code needs two things the pinned contract does not provide. A
command must learn which document invoked it, and the application must learn
when that document retires. Without them the backend lease cannot be scoped or
revoked, because a command sees only a window label.

Expose both through the runtime's extension trait:

- A command can read a redacted document identifier for its invocation. The
  identifier is stable for the document's life, unique within the process, and
  never accepted as authority on any inbound path.
- The application can register a document-retired observer. The runtime
  delivers each retirement exactly once, on the owner thread, and before the
  successor document's bootstrap runs. An observer registered after a
  retirement does not receive it.

The same trait carries the capability query. A typed enumeration names each
optional operation, and `supports(capability)` answers before the application
depends on it. Calling an unsupported method that returns `()` emits a
structured warning and increments a counter; a strict mode turns that into a
panic for conformance runs. Tauri's runtime error type is non-exhaustive and
creation errors are boxed, so the runtime publishes its own downcastable error
type for unsupported operations, expired documents, and capacity. That needs no
change to Tauri core and shrinks the Tauri series.

### One completion type

Every admitted native operation holds one value of a single completion type.
Completing it consumes the value and removes the registry entry. Dropping it
without completing resolves the operation as cancelled. Invocations and
resource requests share the type, so “exactly one terminal outcome” is enforced
by ownership.

### Bootstrap and transport

Tauri already supplies ordered initialization scripts with a main-frame-only
flag. The runtime installs the native binding and those scripts before any
author script executes. It preserves core, plugin, and application ordering. A
failure prevents the real navigation; load-complete injection cannot meet this
contract.

Tauri currently tries custom-protocol fetch before native messaging on desktop.
`NativeJsonProtocolBinary` therefore requires a proposed runtime policy hook
and a change to Tauri's generated template. It must not depend on an unseen
existing selector or a copied private template. Native JSON uses the existing
Tauri envelope. Binary invokes and channel data fetches retain the protocol
route because the current native decoder produces JSON payloads. Pure native
binary delivery would require another explicit decoder contract. Both
transports require trusted source metadata. Automatic retries after dispatch
need shared deduplication; otherwise report failure without resending a
side-effecting command.

### Resource delivery and failures

The proposed provider supplies local resources within Servo's normal fetch
policy pipeline. It does not replace CSP, CORS, origin, MIME, or secure-context
checks. Preserve repeated headers, byte bodies, status codes, and range
behaviour. Distinguish document, worker, internal-engine, and unattributed
requests. Worker/internal requests still obey reserved-origin routing; only an
authorized document source can enter a privileged IPC route in the initial
profile.

Tauri's current protocol request and response bodies are buffered. Enforce
allocation and admission budgets, absolute deadlines, cancellation, and
one-shot completion; converting a buffered body into engine chunks does not
make the public interface streaming. Invoke completions outside mutable engine
borrows and locks. Closing a view releases provider work even if its consumer
disappears.

`RuntimeFailure` is a proposed vocabulary. Existing Tauri errors lack generic
unsupported-operation and expired-document variants. Add those upstream, reject
unsupported creation settings early, and provide preflight capability checks
for required methods whose signatures return no result.

## Conformance and release gates

The [conformance gates](conformance-gates.md) define the acceptance criteria
for every claim in this design. Each gate has a stable identifier, an evidence
mode, and a place in an evidence record. No implementation tests have run.

Gates are grouped into stages by the claim they support: P for the terminal
stack in the engine, H for the native host, V for the verified product subset,
B for a supported desktop candidate, and A and C for the general runtime and
further capabilities. Stages A and C have no consumer and are not planned work.
The stages are not a delivery order. The [roadmap](roadmap.md) is the only
delivery order, and the gates document maps each stage to its roadmap steps.

## Decision gates

Three decisions determine whether the next investment is justified. Each has a
roadmap task that records it.

The first is whether Servo can satisfy Velocetty's terminal rendering and input
requirements. A stock engine answers the cheapest form of that question before
any adapter exists.

The second is whether the investment is worth making. It weighs the measured
benefit named in the
[terms of reference](terms-of-reference.md#motivation-and-stop-criteria)
against the measured cost of carrying the patch series, after the host proof
and the carry rehearsal and before any privileged work.

The third is whether the engine and core privilege contracts can be delivered
and maintained, confirmed by rebasing the privileged series across a release.

If any fails, stop at the corresponding prototype. Do not recover apparent
compatibility by synthesizing origins, loosening capabilities, or allowing
results to follow an unrelated document.

Do not estimate total throughput, memory savings, startup improvement, or
delivery dates from architectural shape alone. Measure one, two, and ten
windows; idle and animation workloads; startup to first usable frame; invoke
latency; resource-buffer peaks; GPU memory; and teardown latency. Compare the
same fixture and build mode under Electron, Servo, and the Tauri CEF and Wry
runtimes, while recording the different process accounting. The CEF runtime is
the null hypothesis: it offers the same Tauri integration with a mature engine
and no engine patches.

If engine or privilege qualification stalls, Velocetty can ship its
browser-safe frontend and backend protocol on its existing Electron host while
keeping the inert Servo harness. This preserves product delivery and provides a
comparison workload. It retains Electron distribution costs and requires
maintaining fixtures for both hosts. A standing gate keeps that fallback
shippable: the Electron host runs the same frontend against the same service at
each product-subset and release evidence record. Reopen Servo promotion when
the failed gate has reproducible evidence on the proposed shipping
configuration.

## Open decisions and evidence required

The architecture selects the boundaries above. Implementation must resolve\
the following choices with evidence rather than implicit fallbacks.

| Decision                           | Initial position                                                                               | Evidence required to finalize it                                                                                       |
| ---------------------------------- | ---------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| Exact Servo/Tauri/host release set | Servo 0.7.0 control; patch target chosen between 0.8.0 and the 0.6 support line; Tauri alpha.6 | Build and native launch on each claimed target; dependency and ABI reconciliation.                                     |
| Published crate name               | Provisional `tauri_runtime_servo`; the registry name belongs to another project                | A publishable name, or an agreement with the existing author, recorded before any public release.                      |
| Terminal graphics profile          | Per-revision WebGL2 and IntersectionObserver settings; Canvas and DOM renderer fallbacks       | Locked xterm corpus, context-loss recovery, native input/accessibility, and measured resource budgets.                 |
| Privilege API delivery             | Public engine bootstrap, source identity, exact-document delivery, and normal-policy resources | Agreed API shape or maintained downstream patch ownership; negative tests at each boundary.                            |
| Product native-service subset      | Implement the operations selected by Velocetty's host ports                                    | Inventory every direct Electron/Node dependency and chosen Tauri plugin; reject undeclared requirements before launch. |
| Queue and performance budgets      | Bounded queues with conservative configurable defaults                                         | Actual terminal traces on representative hardware, including a slow parser and a noisy background session.             |
| Local browser/transport policy     | HTTP localhost packaged origin and authenticated loopback WebSocket                            | Actual secure-context, Origin, CSP, mixed-content, authentication, and lease-retirement tests.                         |
| Shutdown guarantee                 | Measured blocking teardown with a process-exit watchdog                                        | An explicit asynchronous engine lifecycle before promising a bounded host shutdown.                                    |
| Desktop-service transport          | Native bridge specified here; two cheaper options compared in ADR 0001                         | The native-service inventory and the origin and connection probe.                                                      |
| Toolchain scope                    | Workspace flags scoped to this project's crates, proposed in ADR 0002                          | Build time, peak memory, and pin-bump cost for both arrangements.                                                      |
| Session survival on host loss      | Backend outside the engine process; sessions destroyed on abnormal loss                        | The maintainer's choice between destruction and a bounded reattachment period; fault-injection results.                |
| Process-wide accelerated-pane cap  | None; each document's pool applies its own cap                                                 | Context-exhaustion behaviour at one, two, and ten windows.                                                             |
| Browser profile persistence        | One persistent profile                                                                         | Whether the frontend stores anything in cookies or web storage; an ephemeral profile removes the migration hazards.    |
| Engine patch-latency budget        | Undeclared                                                                                     | Timed carry rehearsals and the maintainer's declared limit.                                                            |

These decisions have named tests or ownership requirements. None requires\
inventing an application migration deadline or assuming an upstream\
interface will be accepted unchanged.

## Validation status

The November review checked source contracts at the October baselines. Tauri's
runtime handle and dispatchers require `Send + Sync`, while its native webview
type only requires `'static`. The native IPC handler receives a detached view
and an HTTP request with a string body. Servo's public webview contains
thread-local shared state and accepts JavaScript evaluation against a webview.
These interfaces support owner-thread access through a dispatcher. Privileged
completion still requires engine-side identification of the intended document
when the operation executes. [Tauri runtime][pinned-runtime-contracts],
[Tauri webview contract][tauri-webview-contract],
[Servo webview][servo-webview].

The pinned Velocetty package declares Electron and xterm version ranges. Its
lockfile supplies the actual workload versions. The Verso manifest and fork pin
establish historical integration precedent. Tauri's runtime guide describes
custom-runtime packaging. These sources support the choice of fixtures and the
need for a Servo-specific packaging inventory.
[Velocetty manifest][package-manifest],
[Velocetty lockfile][locked-dependencies], [Verso manifest][verso-manifest],
[Tauri runtime guide][runtime-guide].

A second pass on 26 November 2026 re-read every cited source at its pin and
compared Servo 0.8.0, Servo main of 25 November, and the Tauri `v3` head of 24
November. It confirmed the pinned-source claims in this document. It corrected
the 0.8.0 toolchain, the description of the experimental-feature switch, the
scope of the sandbox exclusion, and the account of Velocetty's backend script.
It added the long-term support line, per-revision preference hazards, further
interception defects, the removal of xterm's Canvas renderer upstream, the
published crate of the same name, and the state of upstream embedding
proposals. Three items rest on source reading alone and need an experiment:
secure-context and loopback WebSocket behaviour from the packaged origin, the
three inferred interception behaviours, and whether the pinned releases
reproduce the reported WebGL2 terminal failure.

The repository contains the runtime scaffold. Source inspection leaves the host
build, two-window launch, document delivery, packaging, and terminal
performance gates open. Each release claim needs a conformance result tied to
the tested revisions and target.

## Glossary

The [context document](context.md) is the glossary for this design, the
roadmap, and the conformance gates.

## References

Source links identify the inspected revisions where a source snapshot is
available. Blog and guide links supply upstream context; proposed contracts in
this document are design decisions.

### Velocetty application and migration contract

- [Application manifest](https://github.com/leynos/velocetty/blob/40949b2f4b266d8523d6ffe680f1dbc53ad9b541/package.json).

- [Locked dependencies](https://github.com/leynos/velocetty/blob/40949b2f4b266d8523d6ffe680f1dbc53ad9b541/bun.lock).

- [Backend, transport, authentication, and host design](https://github.com/leynos/velocetty/blob/40949b2f4b266d8523d6ffe680f1dbc53ad9b541/docs/velocetty-design.md).

- [Product requirements](https://github.com/leynos/velocetty/blob/40949b2f4b266d8523d6ffe680f1dbc53ad9b541/docs/velocetty-product-requirements-document.md).

- [Development roadmap](https://github.com/leynos/velocetty/blob/40949b2f4b266d8523d6ffe680f1dbc53ad9b541/docs/roadmap.md).

- [Current transport facade](https://github.com/leynos/velocetty/blob/40949b2f4b266d8523d6ffe680f1dbc53ad9b541/lib/transport/index.ts).

- [Terminal rendering and input](https://github.com/leynos/velocetty/blob/40949b2f4b266d8523d6ffe680f1dbc53ad9b541/lib/components/term.tsx).

- [Current PTY batching](https://github.com/leynos/velocetty/blob/40949b2f4b266d8523d6ffe680f1dbc53ad9b541/app/session.ts).

- [Native windows and session teardown](https://github.com/leynos/velocetty/blob/40949b2f4b266d8523d6ffe680f1dbc53ad9b541/app/ui/window.ts).

- [Synthetic frame-timing benchmark fixture](https://github.com/leynos/velocetty/blob/40949b2f4b266d8523d6ffe680f1dbc53ad9b541/scripts/run-pty-frame-timing-benchmark.mjs).

- [xterm 5.3 render service](https://github.com/xtermjs/xterm.js/blob/5.3.0/src/browser/services/RenderService.ts).

- [xterm WebGL context-loss handling](https://github.com/xtermjs/xterm.js/blob/5.3.0/addons/xterm-addon-webgl/src/WebglRenderer.ts).

### Servo release, embedding, and policy

- [Servo 0.7.0 release](https://github.com/servo/servo/releases/tag/v0.7.0).

- [Release toolchain](https://github.com/servo/servo/blob/aac43a3f31a259f04a574f5ec4e959c943ad7cc7/rust-toolchain.toml).

- [Comparison-snapshot rendering contexts](https://github.com/servo/servo/blob/69911d3671ff2c75fc0d12d2553718b16956a2b1/components/shared/paint/rendering_context.rs).

- [Public embedding facade](https://github.com/servo/servo/blob/69911d3671ff2c75fc0d12d2553718b16956a2b1/components/servo/lib.rs).

- [Early 2025 embedding improvements](https://servo.org/blog/2025/03/10/this-month-in-servo/).

- [November 2025 rendering improvements](https://servo.org/blog/2025/12/15/november-in-servo/).

- [December 2025 embedding improvements](https://servo.org/blog/2026/01/23/december-in-servo/).

- [August 2026 engine improvements](https://servo.org/blog/2026/10/09/august-in-servo/).

- [Engine ownership, globals, and teardown](https://github.com/servo/servo/blob/69911d3671ff2c75fc0d12d2553718b16956a2b1/components/servo/servo.rs).

- [Release user-script timing](https://github.com/servo/servo/blob/aac43a3f31a259f04a574f5ec4e959c943ad7cc7/components/script/dom/userscripts.rs).

- [Resource interception implementation](https://github.com/servo/servo/blob/69911d3671ff2c75fc0d12d2553718b16956a2b1/components/net/request_interceptor.rs).

- [Fetch routing implementation](https://github.com/servo/servo/blob/69911d3671ff2c75fc0d12d2553718b16956a2b1/components/net/fetch/methods.rs).

- [Custom protocol contract](https://github.com/servo/servo/blob/69911d3671ff2c75fc0d12d2553718b16956a2b1/components/net/protocols/mod.rs).

- [Webview and accessibility APIs](https://github.com/servo/servo/blob/69911d3671ff2c75fc0d12d2553718b16956a2b1/components/servo/webview.rs).

- [Site-data categories](https://github.com/servo/servo/blob/69911d3671ff2c75fc0d12d2553718b16956a2b1/components/servo/site_data_manager.rs).

- [Release feature defaults](https://github.com/servo/servo/blob/aac43a3f31a259f04a574f5ec4e959c943ad7cc7/components/config/prefs.rs).

- [Release origin trust rules](https://github.com/servo/servo/blob/aac43a3f31a259f04a574f5ec4e959c943ad7cc7/components/url/origin.rs).

- [Release rendering contexts](https://github.com/servo/servo/blob/aac43a3f31a259f04a574f5ec4e959c943ad7cc7/components/shared/paint/rendering_context.rs).

- [Release fetch routing](https://github.com/servo/servo/blob/aac43a3f31a259f04a574f5ec4e959c943ad7cc7/components/net/fetch/methods.rs).

- [Release HTTP security processing](https://github.com/servo/servo/blob/aac43a3f31a259f04a574f5ec4e959c943ad7cc7/components/net/http_loader.rs).

- [Network-state persistence](https://github.com/servo/servo/blob/aac43a3f31a259f04a574f5ec4e959c943ad7cc7/components/net/resource_thread.rs).

- [Contribution policy](https://book.servo.org/policy/ai-usage.html).

### Tauri integration and Verso prior art

- [Tauri 3 migration guide](https://v3.tauri.app/start/migrate/from-tauri-2/).

- [Tauri runtime integration guide](https://v3.tauri.app/develop/webview-runtime/).

- [Pinned Tauri runtime traits](https://github.com/tauri-apps/tauri/blob/78f49ba3993fed5a1b921235c547731006ad2a74/crates/tauri-runtime/src/lib.rs).

- [Generated invoke transport](https://github.com/tauri-apps/tauri/blob/78f49ba3993fed5a1b921235c547731006ad2a74/crates/tauri/scripts/ipc-protocol.js).

- [IPC payload serialization](https://github.com/tauri-apps/tauri/blob/78f49ba3993fed5a1b921235c547731006ad2a74/crates/tauri/scripts/process-ipc-message-fn.js).

- [Current channel delivery](https://github.com/tauri-apps/tauri/blob/78f49ba3993fed5a1b921235c547731006ad2a74/crates/tauri/src/ipc/channel.rs).

- [Runtime classification for bundling](https://github.com/tauri-apps/tauri/blob/78f49ba3993fed5a1b921235c547731006ad2a74/crates/tauri-cli/src/interface/rust/manifest.rs).

- [CEF runtime native-host dependencies](https://github.com/tauri-apps/tauri/blob/78f49ba3993fed5a1b921235c547731006ad2a74/crates/tauri-runtime-cef/Cargo.toml).

- [CEF runtime event-loop precedent](https://github.com/tauri-apps/tauri/blob/78f49ba3993fed5a1b921235c547731006ad2a74/crates/tauri-runtime-cef/src/runtime.rs).

- [tauri-runtime-verso manifest](https://github.com/versotile-org/tauri-runtime-verso/blob/dc1c1bfeadf65f3f4e38a4a3c6c960a70b749337/Cargo.toml).

- [Actual pinned Verso fork](https://github.com/tauri-apps/verso/blob/572f47ca7ead69ce3e8b0c433e5cff237f0514c9/Cargo.toml).

- [Verso runtime and request adapter](https://github.com/versotile-org/tauri-runtime-verso/blob/dc1c1bfeadf65f3f4e38a4a3c6c960a70b749337/src/runtime.rs).

- [Verso controller](https://github.com/tauri-apps/verso/blob/572f47ca7ead69ce3e8b0c433e5cff237f0514c9/verso/src/lib.rs).

- [Verso limitations documented by its maintainers](https://github.com/versotile-org/tauri-runtime-verso/blob/dc1c1bfeadf65f3f4e38a4a3c6c960a70b749337/README.md).

[package-manifest]: https://github.com/leynos/velocetty/blob/40949b2f4b266d8523d6ffe680f1dbc53ad9b541/package.json
[transport-facade]: https://github.com/leynos/velocetty/blob/40949b2f4b266d8523d6ffe680f1dbc53ad9b541/lib/transport/index.ts
[terminal-implementation]: https://github.com/leynos/velocetty/blob/40949b2f4b266d8523d6ffe680f1dbc53ad9b541/lib/components/term.tsx
[locked-dependencies]: https://github.com/leynos/velocetty/blob/40949b2f4b266d8523d6ffe680f1dbc53ad9b541/bun.lock
[current-pty-batching]: https://github.com/leynos/velocetty/blob/40949b2f4b266d8523d6ffe680f1dbc53ad9b541/app/session.ts
[tauri-migration-guide]: https://v3.tauri.app/start/migrate/from-tauri-2/
[runtime-guide]: https://v3.tauri.app/develop/webview-runtime/
[pinned-runtime-contracts]: https://github.com/tauri-apps/tauri/blob/78f49ba3993fed5a1b921235c547731006ad2a74/crates/tauri-runtime/src/lib.rs
[runtime-creation]: https://github.com/versotile-org/tauri-runtime-verso/blob/dc1c1bfeadf65f3f4e38a4a3c6c960a70b749337/src/runtime.rs
[controller]: https://github.com/tauri-apps/verso/blob/572f47ca7ead69ce3e8b0c433e5cff237f0514c9/verso/src/lib.rs
[readme]: https://github.com/versotile-org/tauri-runtime-verso/blob/dc1c1bfeadf65f3f4e38a4a3c6c960a70b749337/README.md
[s1]: https://github.com/servo/servo/releases/tag/v0.7.0
[s2]: https://github.com/servo/servo/blob/aac43a3f31a259f04a574f5ec4e959c943ad7cc7/rust-toolchain.toml
[s3]: https://github.com/servo/servo/blob/69911d3671ff2c75fc0d12d2553718b16956a2b1/components/shared/paint/rendering_context.rs
[s4]: https://github.com/servo/servo/blob/69911d3671ff2c75fc0d12d2553718b16956a2b1/components/servo/lib.rs
[s5]: https://servo.org/blog/2025/03/10/this-month-in-servo/
[s6]: https://servo.org/blog/2025/12/15/november-in-servo/
[s7]: https://servo.org/blog/2026/01/23/december-in-servo/
[s8]: https://servo.org/blog/2026/10/09/august-in-servo/
[s9]: https://github.com/servo/servo/blob/69911d3671ff2c75fc0d12d2553718b16956a2b1/components/servo/servo.rs
[s10]: https://github.com/servo/servo/blob/aac43a3f31a259f04a574f5ec4e959c943ad7cc7/components/script/dom/userscripts.rs
[s11]: https://github.com/servo/servo/blob/69911d3671ff2c75fc0d12d2553718b16956a2b1/components/net/request_interceptor.rs
[s12]: https://github.com/servo/servo/blob/69911d3671ff2c75fc0d12d2553718b16956a2b1/components/net/fetch/methods.rs
[s14]: https://github.com/servo/servo/blob/69911d3671ff2c75fc0d12d2553718b16956a2b1/components/servo/webview.rs
[s15]: https://github.com/servo/servo/blob/69911d3671ff2c75fc0d12d2553718b16956a2b1/components/servo/site_data_manager.rs
[main-manifest]: https://github.com/servo/servo/blob/69911d3671ff2c75fc0d12d2553718b16956a2b1/Cargo.toml
[main-toolchain]: https://github.com/servo/servo/blob/69911d3671ff2c75fc0d12d2553718b16956a2b1/rust-toolchain.toml
[global-options]: https://github.com/servo/servo/blob/69911d3671ff2c75fc0d12d2553718b16956a2b1/components/config/opts.rs
[http-checks]: https://github.com/servo/servo/blob/69911d3671ff2c75fc0d12d2553718b16956a2b1/components/net/http_loader.rs
[fetch-implementation]: https://github.com/servo/servo/blob/aac43a3f31a259f04a574f5ec4e959c943ad7cc7/components/net/fetch/methods.rs
[http-loader]: https://github.com/servo/servo/blob/aac43a3f31a259f04a574f5ec4e959c943ad7cc7/components/net/http_loader.rs
[manifest-detection]: https://github.com/tauri-apps/tauri/blob/78f49ba3993fed5a1b921235c547731006ad2a74/crates/tauri-cli/src/interface/rust/manifest.rs

[tauri-webview-contract]: https://github.com/tauri-apps/tauri/blob/78f49ba3993fed5a1b921235c547731006ad2a74/crates/tauri-runtime/src/webview.rs
[servo-webview]: https://github.com/servo/servo/blob/aac43a3f31a259f04a574f5ec4e959c943ad7cc7/components/servo/webview.rs
[verso-manifest]: https://github.com/versotile-org/tauri-runtime-verso/blob/dc1c1bfeadf65f3f4e38a4a3c6c960a70b749337/Cargo.toml

[velocetty-november]: https://github.com/leynos/velocetty/blob/6078e4ee51f527d248c45bc967cf65b694d71e71/package.json
[velocetty-pool]: https://github.com/leynos/velocetty/blob/40949b2f4b266d8523d6ffe680f1dbc53ad9b541/lib/utils/webgl-context-pool.ts
[servo-webgl2]: https://github.com/servo/servo/issues/41394
[servo-xterm]: https://github.com/servo/servo/issues/30822
[xterm-6]: https://github.com/xtermjs/xterm.js/releases/tag/6.0.0
[tauri-v3]: https://github.com/tauri-apps/tauri/tree/v3
[copse-runtime]: https://github.com/copse-dev/tauri-runtime-servo
[wry-servo]: https://github.com/tauri-apps/wry/pull/1797
[servo-lts]: https://book.servo.org/embedding/lts-release.html
[servo-september]: https://servo.org/blog/2026/11/05/september-in-servo/
[servo-message-ports]: https://github.com/servo/servo/pull/40513
[servo-messaging-issue]: https://github.com/servo/servo/issues/35665
