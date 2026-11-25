# Tauri Servo runtime design

**Status:** Proposed architecture; no implementation or compatibility claim.

**Date:** 29 October 2026.

**Audience:** Runtime implementers, application developers evaluating the\
backend, and reviewers of the Servo and Tauri integration boundaries.

**Download:**
[Complete Markdown specification](https://chatgpt.com/library?file_id=libfile_de66df53c11c8191aa9e43c299dcc3b5\&account_id=personal)
·
[Design bundle with proposed Rust contracts and conformance manifest](https://chatgpt.com/library?file_id=libfile_cccafbee5c708191b46bb0bbe71675f8\&account_id=personal)

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

Use Velocetty revision\
`40949b2f4b266d8523d6ffe680f1dbc53ad9b541`, from 29 October 2026, as the\
first application baseline. Its current implementation runs on Electron,\
React 19, Redux, and xterm 5.3.0. It already has a transport facade,\
visible-pane WebGL allocation, Canvas fallback, and runtime telemetry.\
The protobuf/WebSocket service, Rust PTY backend, and Tauri packaging are\
planned work. The frontend still contains Electron and Node dependencies.\
[Package manifest][package-manifest], [transport facade][transport-facade],\
[terminal implementation][terminal-implementation],
[locked dependencies][locked-dependencies],\
[roadmap](https://github.com/leynos/velocetty/blob/40949b2f4b266d8523d6ffe680f1dbc53ad9b541/docs/roadmap.md).

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
[Terminal rendering and allocation](https://github.com/leynos/velocetty/blob/40949b2f4b266d8523d6ffe680f1dbc53ad9b541/lib/components/term.tsx).

### Make the actual xterm stack the first engine gate

Velocetty probes `canvas.getContext('webgl2')`. Servo 0.7.0's defaults\
disable `dom_webgl2_enabled` and `dom_intersection_observer_enabled`,\
while enabling `dom_resize_observer_enabled`. Therefore a default engine\
build does not establish the intended accelerated terminal path.\
[Servo release preferences](https://github.com/servo/servo/blob/aac43a3f31a259f04a574f5ec4e959c943ad7cc7/components/config/prefs.rs).

Create a declared Velocetty engine profile that explicitly enables and\
tests WebGL2 and, where correct, IntersectionObserver. Avoid the blanket\
experimental-feature switch. A preference proves that an API can be\
requested; it does not prove shader, atlas, context-loss, or platform\
driver compatibility. If a target cannot pass the accelerated path,\
qualify a clearly identified Canvas mode against the actual workload\
before treating it as an acceptable product mode.

Run the exact locked xterm/addon combination before upgrading it. Test\
WebGL2 glyph atlases, fonts, fractional DPI, combining characters,\
wide characters, emoji, selection, cursor rendering, images, search,\
hyperlinks, and long scrollback. Validate cell layout and terminal\
contents as well as platform-specific screenshots. Do not require\
identical subpixel rasterization across different operating systems.

Velocetty deliberately uses Canvas for transparent terminal rendering,\
and currently enables its ligatures addon on the Canvas path. Exercise\
those paths directly. A successful opaque WebGL terminal is not evidence\
that transparent windows or ligatures work.
[Terminal implementation](https://github.com/leynos/velocetty/blob/40949b2f4b266d8523d6ffe680f1dbc53ad9b541/lib/components/term.tsx).

Hidden-pane Canvas fallback must preserve terminal state while avoiding\
unnecessary painting. xterm's hidden-render optimization depends on\
IntersectionObserver, so test the declared preference and its actual\
behaviour. If it is unreliable, add an explicit application visibility\
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
that the application expects to receive.

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
alone is not an end-to-end consumption acknowledgement.

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

The host registers a backend-side lease scoped to the backend instance,\
permitted capabilities, session namespace, and native document/view\
generation. The backend stores that mapping through a trusted control\
path; the native `DocumentToken` never enters JavaScript. A single-use\
bootstrap grant is consumed into a connection attached to this lease.\
The WebSocket handshake proves possession of that scoped bearer grant\
and supplies the browser Origin; it cannot independently recover Servo's\
native document identity.

On document retirement, the host revokes the lease over an authenticated\
supervisory channel. The backend rejects further commands and output on\
every associated connection, closes them, and destroys their PTYs under\
the existing lifecycle policy. An in-process Rust backend can use a\
direct service call; the transitional Node backend needs a private\
inherited pipe or an equivalently authenticated local control channel.\
Loss of that host channel revokes its leases. Validate lease state at\
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

Expose a configuration and selection type named\
`tauri_runtime_servo::Servo`, and a concrete implementation type named\
`ServoRuntime<T>`. The engine type remains `servo::Servo`. The intended\
application shape is:

```rust
// Proposed API shape; tauri-runtime-servo does not exist yet.
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

At this revision, Tauri owns scoped event-listener registries. The runtime\
emits the corresponding `RunEvent` values; it must not duplicate the old\
Verso listener bookkeeping. `with_webview` receives a Servo-specific\
wrapper on the owner thread. This wrapper is not a Wry object and need\
not implement `Send`.

Velocetty currently runs on Electron and does not supply an existing\
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

## Servo baseline, retained improvements, and required deltas

### Baseline and update policy

Start from **Servo 0.7.0**, released on 5 October 2026, at commit
`aac43a3f31a259f04a574f5ec4e959c943ad7cc7`. Maintain a small, reviewed patch
queue for the embedding contracts this runtime requires. The comparison
snapshot of Servo main is `69911d3671ff2c75fc0d12d2553718b16956a2b1`, dated
29 October 2026. Pin the complete dependency graph and record the patches with
the runtime release; do not follow a moving branch in application builds.
[S1][s1], [S2][s2]

| Property                          | Inspected value                                         | Design consequence                                                 |
| --------------------------------- | ------------------------------------------------------- | ------------------------------------------------------------------ |
| Release toolchain                 | Rust 1.97.1                                             | Use the release's pinned toolchain initially.                      |
| Main toolchain                    | Rust 1.99.1                                             | A main-branch improvement may also require a toolchain update.     |
| Main manifest MSRV                | Rust 1.88.0                                             | Distinguish declared MSRV from the release build toolchain.        |
| Main rendering dependencies       | Surfman 0.14.0; WebRender 0.70.1; raw-window-handle 0.6 | Integrate through raw handles and the public rendering contract.   |
| Main shell/example window library | Winit 0.30.13                                           | This does not require the runtime to use Servo's shell event loop. |

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

#### Bootstrap and local-resource requirements

Current user scripts run through `HTMLHeadElement::bind_to_tree`, which queues
a task that executes when DOM script/layout safety blockers clear. This can run
early during normal HTML parsing. It does not provide a universal
document-created hook, a once-only guarantee, frame/origin selection, or an
isolated execution world. The runtime needs a dedicated bootstrap contract
instead of relying on head insertion timing.
[S10](https://github.com/servo/servo/blob/aac43a3f31a259f04a574f5ec4e959c943ad7cc7/components/script/dom/userscripts.rs)

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

Custom protocols remain useful, but registering one does not establish a
standard tuple origin. `ProtocolHandler::is_secure` documents only
mixed-content treatment, and Servo's origin model delegates to the URL library.
Implementing comprehensive custom-scheme origin semantics would touch more
engine subsystems than a virtual HTTP(S) transport.
[S13](https://github.com/servo/servo/blob/69911d3671ff2c75fc0d12d2553718b16956a2b1/components/net/protocols/mod.rs)

### Lifecycle and platform limits

Keep one `Servo` instance on the UI thread. Its handles use `Rc`;
process-global options use a `OnceLock` that rejects a second initialization.
Preferences also remain global. Public `spin_event_loop()` returns `()`. Final
`ServoInner` drop sends shutdown and synchronously pumps until completion.
Cancel runtime work, release webview/delegate references, and preserve surface
lifetimes before allowing that final drop. Do not promise in-process recovery
by rebuilding Servo after failure.
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
Windows, ARM/aarch64, Android, iOS, RISC-V, and OHOS targets.
Supported-platform claims must reflect these limits. [S9][s9], [S14][s14],
[S15][s15]

Additional baseline references: [main manifest][main-manifest],
[main toolchain][main-toolchain], [global options][global-options],
[HTTP checks][http-checks], and
[origin model](https://github.com/servo/servo/blob/69911d3671ff2c75fc0d12d2553718b16956a2b1/components/url/origin.rs).

## Component boundaries

Start with three production crates and a test workspace. Keep components\
as modules until an independently useful API justifies another crate.

| Component                             | Owns                                                                                                                          | Must not own                                                        |
| ------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------- |
| `tauri-runtime-servo`                 | Tauri trait implementations, configuration validation, ID translation, events, extension traits, IPC adaptation               | Browser UI, duplicated Tauri authorization, private Servo internals |
| `servo-host`                          | One engine, native host loop, view registry, rendering surfaces, input conversion, document leases, resource-provider adapter | Tauri command semantics, arbitrary application business logic       |
| `tauri-runtime-servo-build`           | Compatibility metadata, resource staging, native dependency inventory, packaging validation                                   | Unpinned downloads, application signing keys, runtime policy        |
| Conformance applications and fixtures | Reproducible behavioural evidence and regression cases                                                                        | Production compatibility shims                                      |

`servo-host` is a workspace-private crate initially. It may depend on\
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

`DocumentToken` is an opaque native capability to address a document,\
not a secret that JavaScript can mint or a JSON structure that the host\
deserializes. Its implementation includes engine-instance and\
view-generation scope. Public diagnostics may expose a redacted\
correlation ID, but the transport treats that diagnostic value as\
untrusted.

Before Tauri dispatches privileged work, the runtime checks the live\
document lease and frame/origin policy. It then supplies the genuine\
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

| State or transition             | Required behaviour                                                                       |
| ------------------------------- | ---------------------------------------------------------------------------------------- |
| View allocated                  | No application IPC; all bootstrap and origin policy installed before load.               |
| Initial empty document          | No privilege by default, even when it has an inherited origin.                           |
| Authorized document created     | Mint its token and install its permitted bridge before author code.                      |
| Same-document history change    | Retain the document token; use the current engine URL when policy is path-sensitive.     |
| Cross-document navigation       | Retire the old lease and create a separate identity for the new document.                |
| Frame detached or view replaced | Cancel pending delivery, subscriptions, and payload redemption for the retired identity. |
| Closing or crashed              | Reject new work and settle native pending operations exactly once.                       |

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

An admitted native operation owns a one-shot completion. View retirement,\
timeout, shutdown, or a disconnected worker resolves it once and removes\
its registry entry. Dropping an internal request must not accidentally\
become network fallback or successful partial completion.

Differentiate a requested close from destruction. Emit the Tauri\
close-request callback before committing to closure, honour its veto,\
and resolve any supported engine unload decision. Once closure commits,\
stop admitting work, retire document leases, cancel resource requests,\
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

- Tauri workspace SHA and frontend package versions.

- Rust toolchain, target triple, native ABI baseline, and compiler/linker\
  settings that affect distributed artefacts.

- Resource and binary checksums, native dependency inventory, and notices.

- Storage-format compatibility and the conformance result identifiers.

Do not promise that a prebuilt Rust `rlib` is a stable SDK independent of\
toolchain, target, and dependency graph. Initially reuse ordinary Cargo\
caches and trusted native dependency binaries. A stable C ABI or dynamic\
engine SDK would be another project, not a packaging shortcut.

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

Track Servo security updates independently of feature work. Keep engine\
changes as a small, documented patch series with one purpose per patch\
and upstream tracking where appropriate. Rebase through the conformance\
suite. Compare a current monthly release with an LTS option when their\
required embedding contracts match; do not freeze an obsolete engine\
merely to avoid adapter maintenance.

Servo's current contribution policy excludes LLM-generated code,\
documentation, issues, and comments. Therefore upstream acceptance is a\
delivery dependency that cannot be satisfied by submitting this generated\
design or generated patches. Arrange contributions that comply with the\
project's policy, or explicitly fund and maintain a downstream patch\
series. This affects the maintenance plan, not the feasibility of an\
independent runtime.
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

## Contract adaptation notes

**Status:** specification only. `proposed-contracts.rs` names intended seams;
it does not reproduce existing public Servo or Tauri APIs, and it has not been
compiled. The inspected Tauri baseline is commit
`78f49ba3993fed5a1b921235c547731006ad2a74` on the v3 branch.

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

This section defines proposed acceptance criteria. No implementation tests have
run. A release claim requires evidence for the exact Tauri revision, Servo
revision, runtime revision, build configuration, and target platform.
Successful compilation establishes API compatibility only; it does not
establish browser policy, native integration, or lifecycle correctness.

The accompanying `conformance.toml` contains the normative checklist. The
general runtime stages inherit their predecessors. Velocetty may establish a
separately named verified subset after stages P and H plus every security,
lifecycle, native, and transport case applicable to its declared surface; this
does not claim complete stage A conformance. Conditional groups identify
applicable exercises explicitly: JSON-only invoke and native-events-only
support do not imply binary invoke or channels. Completing every Tauri channel
API must not precede the terminal proof. Stage C comprises independent future
capability gates. A successful no-op or fabricated empty result cannot
substitute for an unsupported-operation error.

### Stage P: Velocetty engine and protocol proof

Use Velocetty revision `40949b2f4b266d8523d6ffe680f1dbc53ad9b541` and its
actual locked dependencies: xterm 5.3.0, WebGL addon 0.16.0, and Canvas addon
0.5.0. Extract a browser-safe fixture with explicit host ports; do not attempt
to run Electron imports or privileged legacy plugin hooks inside Servo.
Preserve the chosen frontend assets and record any deliberate fixture
differences.

Prove WebGL2 rendering, Canvas fallback, ResizeObserver delivery, visibility
and context-pool transitions, context loss/recovery, font metrics, Unicode,
selection, keyboard modifiers, IME, and screen-reader behaviour. Exercise
several xterm panes inside one document; these panes do not require native
child webviews. Test image, search, link, and ligature addons only with their
declared support policy, reporting unsupported features explicitly.

Start with recorded output and a controlled server, then test the planned
protobuf/WebSocket service on the existing Node backend before replacing its
PTY implementation with Rust. Verify binary frames, split UTF-8 sequences,
session multiplexing, resize/close ordering, and authentication from the
production application origin under ordinary CSP and connection policy. The
local backend must authorize PTY access independently of Tauri invoke
permissions. Do not disable browser policy or treat loopback as authentication.

Test the complete backend lease lifecycle: a one-use, scoped bootstrap grant
binds a frontend connection to a backend lease through the trusted host
supervisory path. Document retirement or loss of the host control channel must
revoke grants and leases, reject further work and output, close frontend
connections, and destroy their sessions. Race revocation against grant
redemption and pending work. Restarting the backend must invalidate every grant
and lease issued by its previous instance. These obligations apply to the Node
service proof without requiring the later Rust replacement.

Bound output by byte credits and queue limits; return consumption credit after
xterm's parse/write-completion callback, not merely after WebSocket receipt.
Preserve immutable source output and its ordering as untrusted data: titles,
links, image payloads, and terminal control sequences must not acquire native
authority. Keep input and lifecycle commands responsive while another pane
floods output. Measure actual PTY-to-parse/render and input-to-backend latency
on native GPUs. Velocetty's deterministic synthetic benchmark remains a
contract fixture and cannot satisfy these measurements.

### Stage H: host proof

Build one process-owned Servo instance and two independently controlled native
windows. Load packaged fixtures without installing privileged IPC. Prove
resize, presentation, navigation, input, focus, IME, and orderly close. Close
the first-created window while the second remains usable, then create another
window from an application callback. Verify that window destruction cannot
invalidate another window's surface, renderer, or engine ownership.

Exercise minimize, occlusion, restore, zero-sized surfaces, DPI changes,
display disconnection, and surface recreation. Rendering must recover or report
a terminal failure without presenting another view's content. Input routing
must preserve physical and logical coordinates, modifiers, pointer capture, and
IME composition. The event loop must wake for work, remain idle without
continuous repainting, and preserve window responsiveness during animation and
command bursts.

Test native GPU paths on every claimed platform. Headless rendering and
software rasterization provide useful additional evidence but cannot satisfy
native presentation gates. Record GPU, driver, window system, scale factors,
and display configuration with failures.

### Stage A: complete general runtime alpha

Support two top-level windows, one browser profile, packaged content, bounded
binary custom-protocol requests, and the complete chosen invoke/channel
transport. This surface is broader than Velocetty's first verified subset.
Reject independent child-view creation, conflicting browser-profile settings,
per-view incognito, and unsupported configuration before loading a privileged
page. This restriction permits the proposed single engine-wide ephemeral
profile; it does not promise isolated per-view storage. Velocetty's terminal
profiles select shells and environments; they are not browser profiles.

**Initialization and document identity.** Prove that required initialization
executes before the first author script in every document eligible for the
bridge. Cover initial empty documents, reload, redirects, same-document history
changes, new documents, iframe creation, `srcdoc`, `about:blank`, blob
documents, and restored history entries where supported. Assert the declared
frame policy for each case: required initialization for eligible documents, and
bridge absence for ineligible documents. A frame must not acquire privilege
merely by inheriting a URL, sharing a parent, or requesting an application
asset.

Use engine-issued frame and document identities. Queue replies while
navigating, destroying frames, replacing windows, and recycling registry slots.
Servo must validate the target at actual script execution or message delivery.
A check on the UI thread before enqueueing is insufficient. Old replies,
events, and channel data must never reach a replacement document, even when it
uses the same URL and callback number.

**Authorization and protocol policy.** Attack the local-origin provider with
forged `Origin`, `Referer`, `Host`, invoke-key, and callback headers; missing
or opaque origins; remote frames; redirected requests; conflicting scheme
registrations; and canonicalization variants involving case, trailing dots,
ports, userinfo, encoded separators, and traversal. Authorization must use
trusted initiating context and Tauri's capability checks. Requests for
registered application origins must never fall through to DNS or the network.
Unprivileged requests must remain unprivileged when their destination is a
local asset.

Verify ordinary fetch policy across local and network resources: CSP, CORS and
preflight, credentials, redirects, MIME handling, caches, and any enabled
service-worker interception. Test `GET`, `HEAD`, byte ranges, and binary `POST`
bodies against the same protocol contract. Exercise every declared byte limit
at the boundary and one byte beyond it, including empty bodies and arbitrary
non-UTF-8 data. Reject overflow deterministically; never truncate a request or
encode its body into a header. Cancellation and handler failure must complete
or abandon requests according to the contract without leaving an unlimited
pending map.

**Invoke, channels, and events.** Use a fixture command with an observable
counter to detect duplicate dispatch. Force failures before dispatch, after
acceptance, during transport completion, and after the command's side effect.
An uncertain outcome must not trigger automatic resend. Return the declared
uncertain-outcome error; callers may retry only when their application contract
provides idempotency. Losing the originating document cancels delivery and
document-scoped subscriptions; it cannot promise rollback of accepted external
work.

Test success and error results, binary payloads, Unicode, malformed messages,
invalid callback IDs, channel setup and closure, cross-window routing, and
application events. Flood each bounded queue, stall consumers, close documents
with queued messages, and verify the declared overflow policy and ordering
within each channel. Enforce both message-count and byte limits where
applicable. Event listeners must unsubscribe on document destruction. Test that
high-volume delivery does not starve native input or engine updates.

**Re-entrancy and failure.** Application callbacks must query windows, create
windows, request close, and enqueue engine work without deadlock. Hold no
registry lock or mutable engine borrow across those callbacks. Same-thread
dispatch must respect the distinction between safe native queries and deferred
engine operations. Exercise handler panics, dropped response receivers, engine
shutdown, and partially created windows. Every accepted request must reach a
documented terminal state; shutdown must not wait indefinitely for a callback
that requires the stopped event loop.

### Stage B: supported desktop application candidate

Run native integration and accessibility tests for each claimed operating
system and window system. Verify macOS titlebar/control integration and
Windows/Linux frameless drag, resize, focus, and control behaviour. Check
menus, accelerators, supported tray behaviour, focus restoration after dialogs,
clipboard text/file lists, paste ordering, external-link policy, drag and drop,
native handles, and application exit vetoes. Check keyboard-only navigation,
accessible names and roles, focus announcements, text editing, selection, and
dynamic updates with the target screen reader. A missing accessibility path
prevents a supported desktop claim for that target.

Velocetty is the primary application fixture. Verify tabs/splits, terminal
buffers and input bytes, sustained output, fallback rendering, clipboard
sequencing, links, configuration changes, and `ready`, session-add,
session-exit, and update-availability flows. Compare the protocol-backed Node
implementation with the Rust replacement using the same corpus. Differences
require adjudication against the declared contract. Generic React and SolidJS
applications may extend later coverage; they cannot replace the terminal
acceptance suite.

Preserve Velocetty's existing policy that subsequent document reload and window
teardown destroy their sessions unless a separate product decision specifies
detach/reattach. Test duplicate close, late output, cancellation, process exit,
and uncertain spawn/write/restart outcomes without automatic replay. Preserve
backend-owned JSON5 configuration and plugin paths; do not require
browser-database persistence for product behaviour that does not use it. Any
claimed browser persistence, clearing, or profile locking still needs its own
tests.

Exercise packaged binaries on clean machines, including signing where claimed.
Test matched app/runtime/engine/resource installation, interrupted updates,
failed first startup, and recovery to a compatible prior package. Recovery does
not promise live-session or scrollback restoration. Release builds must disable
development endpoints; debug grants must name their exact development origin.
Record startup, idle CPU, memory, frame timing, and command latency; declare
budgets from real application measurements before promotion.

### Stage C: additional capability gates

Independent child views require bounds, clipping, z-order, focus, input,
destruction, and reparenting tests. Profile isolation requires adversarial
separation of cookies, storage, credentials, permissions, caches, and clearing
operations. An out-of-process engine requires an independently reviewed
protocol, cancellation and crash recovery, native surface ownership, version
negotiation, and demonstrated sandbox restrictions on each supported target.
Process separation alone does not satisfy the sandbox gate.

Publish a release evidence packet containing the checklist outcomes, exact
revisions, target configurations, test logs, captured failures, explicit
exclusions, and measured budgets. Run the affected conformance subset for every
engine/core update, and the complete supported profile before release. Do not
claim an untested target from another platform's success.

## Implementation order and decision gates

**Terminal engine proof:** run Velocetty's exact locked xterm/addon stack\
and a representative React shell under Servo. Verify WebGL2 with explicit\
preferences, Canvas fallback, hidden-pane behaviour, IME, and binary\
WebSocket policy before building a large Tauri adapter. Use recorded\
terminal data first; then exercise the Node backend service once\
Velocetty's planned protocol shell exists.

**Host proof:** build the selected Servo/Tauri/tauri-winit combination,\
create two native windows from one engine, exercise native handles,\
input, resizing, redraw, close veto, and teardown. Render inert test\
content without exposing application IPC. This validates the platform and\
graphics boundary before adding privilege.

**Minimum privileged bridge:** implement and test document bootstrap,\
authenticated native messages, normal-policy local resources, and\
document-bound JSON bootstrap/native-service results. Exercise the events\
and plugin operations actually needed by Velocetty, CSP, packaged assets,\
and backend lease retirement together. Keep PTY/backend traffic on its\
separate WebSocket contract. Publish the verified product subset and its\
explicit exclusions; comprehensive Tauri channels are not a prerequisite\
unless a chosen product dependency uses them.

**Velocetty desktop candidate:** complete the browser-safe frontend and\
native host ports, qualify the established backend contract under Servo,\
and only then replace Node PTY services with Rust behind that contract.\
Verify terminal lifecycle parity, JSON5 paths, plugins, menus, dialogs,\
clipboard, accessibility, native installers, and updates on each claimed\
platform. Record real performance baselines and failure behaviour. A\
platform enters the support matrix only when its evidence exists.

**General runtime alpha:** extend the tested product subset to the full\
chosen binary invoke, channel, event, and custom-protocol contracts in\
stage A. Make the broader compatibility claim only after that suite passes.

**Broader runtime support:** add independently bounded child views,\
reparenting, further plugin classes, isolation-pattern support, storage\
partitions, or isolated engine processes as separate features. Each must\
remove a declared limitation and pass a corresponding contract suite.

Do not estimate total throughput, memory savings, startup improvement, or\
delivery dates from architectural shape alone. Measure one, two, and ten\
windows; idle and animation workloads; startup to first usable frame;\
invoke latency; resource-buffer peaks; GPU memory; and teardown latency.\
Compare the same fixture and build mode under Servo, Wry, and CEF where\
practical, while recording the different process accounting.

The first product decision gate is whether Servo can satisfy Velocetty's\
terminal rendering and input requirements. The next is whether the\
engine/core privilege contracts can be delivered and maintained. If\
either fails, stop at the corresponding prototype.\
Do not recover apparent compatibility by synthesizing origins, loosening\
capabilities, or allowing results to follow an unrelated document.

## Open decisions and evidence required

The architecture selects the boundaries above. Implementation must resolve\
the following choices with evidence rather than implicit fallbacks.

| Decision                           | Initial position                                                                               | Evidence required to finalize it                                                                                       |
| ---------------------------------- | ---------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| Exact Servo/Tauri/host release set | Servo 0.7.0 plus a small patch series; pinned Tauri 3 and `tauri-winit` sources                | Build and native launch on each claimed target; dependency and ABI reconciliation.                                     |
| Terminal graphics profile          | Explicit WebGL2 and IntersectionObserver settings; tested Canvas fallback                      | Locked xterm corpus, context-loss recovery, native input/accessibility, and measured resource budgets.                 |
| Privilege API delivery             | Public engine bootstrap, source identity, exact-document delivery, and normal-policy resources | Agreed API shape or maintained downstream patch ownership; negative tests at each boundary.                            |
| Product native-service subset      | Implement the operations selected by Velocetty's host ports                                    | Inventory every direct Electron/Node dependency and chosen Tauri plugin; reject undeclared requirements before launch. |
| Queue and performance budgets      | Bounded queues with conservative configurable defaults                                         | Actual terminal traces on representative hardware, including a slow parser and a noisy background session.             |
| Local browser/transport policy     | HTTP localhost packaged origin and authenticated loopback WebSocket                            | Actual secure-context, Origin, CSP, mixed-content, authentication, and lease-retirement tests.                         |
| Shutdown guarantee                 | Measured blocking teardown for the initial alpha                                               | An explicit asynchronous engine lifecycle before promising a bounded host shutdown.                                    |

These decisions have named tests or ownership requirements. None requires\
inventing an application migration deadline or assuming an upstream\
interface will be accepted unchanged.

## Validation status

This proposal derives from current primary documentation and source\
inspection at the cited revisions. The accompanying API sketch describes\
proposed contracts; it is not a compiled implementation. The conformance\
manifest describes acceptance work, not passed tests. No Servo/Tauri build,\
application compatibility run, exploit demonstration, or performance\
benchmark formed part of this design task.

## Glossary

| Term             | Meaning in this design                                                                                     |
| ---------------- | ---------------------------------------------------------------------------------------------------------- |
| Runtime          | The adapter implementing Tauri's window, webview, event, and dispatch contracts.                           |
| Engine host      | The component owning Servo, native surfaces, input translation, and engine lifecycle.                      |
| Native window    | An operating-system window with its own lifetime and presentation surface.                                 |
| Webview          | A Servo browsing context hosted by the runtime; initially one per application window.                      |
| Frame            | A browsing context within a webview, distinct from the document currently loaded in it.                    |
| Document token   | An opaque native reference to one engine document identity; never a frontend-supplied ID.                  |
| Backend lease    | A backend-side authorization lifetime associated with a host document generation and its connections.      |
| Browser profile  | Engine storage, credentials, cookies, and related browsing state for the application.                      |
| Terminal profile | Velocetty's backend-owned shell, environment, working-directory, and terminal configuration.               |
| PTY              | A pseudo-terminal connecting a terminal frontend to a shell or another interactive process.                |
| Invoke           | A Tauri frontend-to-Rust command request, separate from Velocetty's backend service protocol.              |
| Virtual origin   | A reserved HTTP(S) origin served inside the engine's normal fetch pipeline without a network asset server. |
| Admission        | Acceptance of work into a bounded runtime queue; it does not establish successful execution.               |
| Conformance gate | A set of required behavioural results for a declared compatibility profile and target.                     |

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
