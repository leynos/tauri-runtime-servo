# Architectural decision record (ADR) 0001: Desktop-service transport

## Status

Proposed.

## Date

2026-11-26.

## Context and problem statement

Velocetty's frontend needs a small set of desktop services: connection
bootstrap, clipboard, external links, dialogs, and menu events. Its terminal
traffic, filesystem access, configuration, and plugin storage already travel
over an authenticated loopback protobuf/WebSocket service, by the product's own
design.

The [runtime design](../tauri-runtime-servo-design.md) carries those desktop
services through Tauri's in-page invoke bridge. Making that bridge safe in
Servo requires four engine additions and several Tauri core changes. The source
validation of 26 November 2026 found none of them upstream, one relevant
upstream proposal idle for a year, and a Servo contribution policy that
prevents the agent workshop from submitting patches. Every engine addition is
therefore a permanent downstream carry for a team with one human maintainer.

The design review asked whether the bridge is needed at all for the product,
and identified two cheaper transports. This record compares the three so that
the choice is made before any engine patch is written.

## Decision drivers

- The size and permanence of the downstream patch series.
- Protection of pending privileged results from a successor document.
- Reuse of contracts the product already needs for its remote-browser
  frontend.
- Exposure to change in Tauri 3's pre-release IPC and in Servo's embedding
  API.
- Whether a general Tauri runtime for other applications remains a goal.

## Options considered

### Option A: Native in-page bridge

The page calls a native binding that the engine installs before author script.
The engine reports the calling document, Tauri authorizes the command, and the
reply is delivered by an evaluation that the script thread validates against
the original document. This is the option the design specifies.

It preserves the in-page Tauri API and plugins and is the only option that
leads to a general runtime. It requires a document bootstrap hook, a native
message bridge, document-targeted evaluation, and a policy-preserving local
resource provider in Servo, and document context through invoke, resolver,
channels, and listeners in Tauri core.

### Option B: Invoke on the protocol route

The page sends every invoke as a fetch to the reserved IPC origin, which is
Tauri's existing first-choice transport. The reply returns as the fetch
response, which the engine already binds to the initiating document. No message
bridge, transport-policy hook, or document-targeted evaluation is needed for
invoke. Events from Rust to the page still need targeted delivery, or are
replaced by a page-initiated long poll.

It still requires the local resource provider, extended to carry request bodies
and the initiating document's identity, and the bootstrap hook for
initialization scripts. Tauri core must take the caller from engine-supplied
context in place of the `Origin` header.

### Option C: Desktop services over the loopback service

The page receives no privileged bridge. The backend serves the frontend assets
and the WebSocket from one authenticated loopback origin, which is what the
remote-browser deployment already requires. Desktop services become messages in
the product's protocol, using the capability negotiation its requirements
already call for. The host passes a single-use grant to the first navigation. A
WebSocket closes with its document, so a successor document cannot receive a
predecessor's results.

Tauri still supplies windows, menus, tray, and bundling from Rust. The in-page
Tauri API and plugins are unavailable, and the general-runtime goal is
abandoned or deferred. A loopback HTTP listener is exposed to local processes
and is defended by the grant and the origin check. Delivering the grant at
navigation needs a mechanism that keeps it out of URLs and logs, which the
design currently forbids placing in a URL.

| Topic                               | Option A                    | Option B                     | Option C                          |
| ----------------------------------- | --------------------------- | ---------------------------- | --------------------------------- |
| Servo additions                     | Four                        | Two, one of them extended    | None expected                     |
| Tauri core changes                  | Several, in contested areas | Caller identity only         | None                              |
| Stale-reply protection              | New engine operation        | Supplied by fetch            | Supplied by the socket's lifetime |
| In-page Tauri API and plugins       | Preserved                   | Preserved for invoke         | Unavailable                       |
| General runtime for other consumers | Possible                    | Possible with event delivery | Abandoned or deferred             |
| One frontend for desktop and remote | No                          | No                           | Yes                               |
| New exposure                        | In-process native binding   | In-process provider          | Loopback HTTP listener            |

_Table 1: Comparison of desktop-service transports._

## Decision outcome / proposed direction

Decide after two cheap pieces of evidence, and before roadmap step 3.2 spikes
any engine delta.

If the native-service inventory shows that Velocetty needs no in-page Tauri
API, and the origin and connection probe shows that a loopback origin is a
secure context with working CSP and WebSocket policy in Servo, prefer Option C
for the reference application. It removes the patch series that the maintainer
would otherwise carry indefinitely, and it converges the desktop and remote
frontends.

If the inventory finds a required in-page Tauri API, prefer Option B over
Option A unless a required feature needs events that only targeted delivery can
carry.

Retain Option A's specification in the design as the contract a general runtime
would need. Building it requires a second consumer or a recorded decision that
the general runtime is itself the goal.

## Known risks and limitations

- Option C moves desktop-service design into Velocetty's protocol, which is
  owned by another programme and has not started.
- Option C's grant delivery at first navigation is unspecified.
- Option B's extended provider has the same fetch-conformance obligations as
  Option A's.
- The comparison rests on source reading. The probe may show that Servo's
  loopback behaviour differs from its source.

## Outstanding decisions

- Whether a general Tauri runtime is a goal in its own right. The
  [terms of reference](../terms-of-reference.md#motivation-and-stop-criteria)
  ask the maintainer to say.
- How Option C delivers the first-navigation grant without placing it in a
  URL.
- Whether menu and tray events reach the page through the protocol or
  through a narrow host-initiated message.
