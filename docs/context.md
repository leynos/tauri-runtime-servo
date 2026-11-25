# Tauri Servo runtime context

Velocetty is an in-house fork of Vercel Hyper for agentic developers. Its
product vision is “VSCode for terminals”. An *agentic developer* works with
coding agents as part of software development. For example, they might follow a
coding agent's output in one terminal pane while inspecting a diff in a second
pane. The runtime must deliver input to the focused pane's window and keep that
window responsive as output arrives. Velocetty routes input to the pane and
defines what happens to its session when the pane closes.

Hyper is Velocetty's upstream lineage. Velocetty defines its own configuration
and plugin contracts. Hyper configurations and plugins carry no
backwards-compatibility guarantee.

## People and responsibilities

The df12 Productions team comprises one human maintainer and the agent
workshop. The maintainer owns product direction, integration, and release
decisions. The workshop carries out delegated engineering and gathers evidence.
Tauri and Servo maintainers decide which contributions their respective
upstream projects accept. The developer using an agent in a terminal pane is
the product user, a distinct role from developing the runtime.

## Application and host

| Term                  | Meaning and boundary                                                                                                                           |
| --------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| Reference application | The application whose locked frontend, real input, packaging, and workload supply the first product acceptance evidence. Here it is Velocetty. |
| Terminal pane         | An application view of a terminal session. Splitting a pane changes the frontend layout within its document.                                   |
| Pseudo-terminal (PTY) | The operating-system interface through which Velocetty's backend communicates with a shell or other terminal program.                          |
| Terminal profile      | Velocetty's settings for a session, such as its shell, environment, and working directory.                                                     |
| Browser profile       | Engine-managed state such as cookies, credentials, and storage. Several native windows can share it.                                           |
| Runtime               | The adapter through which Tauri operates native windows and webviews and delivers application events.                                          |
| Engine host           | The runtime component that owns Servo and coordinates engine work, native input, and presentation.                                             |

## Identity and authority

A reload can preserve the native window and its webview while replacing the
document. Treating a window label as the recipient of a command result would
let a replacement document receive a previous document's data.

| Term           | Meaning and boundary                                                                                                                           |
| -------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| Native window  | An operating-system window with a lifetime and a presentation surface.                                                                         |
| Webview        | A browsing context hosted in the application. The initial profile places one in each native window.                                            |
| Frame          | A context in which documents load. Navigation can replace the document while the frame survives.                                               |
| Document token | An opaque reference supplied by the engine that identifies one document for the proposed native bridge.                                        |
| Origin         | A browser security principal. Multiple distinct documents can share an origin.                                                                 |
| Invoke         | A request from a frontend document to a Tauri command. It carries desktop operations such as connection bootstrap.                             |
| Backend lease  | The backend's revocable authority for connections associated with a particular host document. A connection can outlive the bootstrap exchange. |
| Admission      | Acceptance of work into a bounded queue. Completion, cancellation, and delivery are later outcomes.                                            |

## Reading a support claim

A *compatibility profile* declares which application and runtime operations a
build supports. A *target* identifies the build triple and the native display
backend where applicable. A *conformance gate* identifies the observations
required to advance a specific profile on that target. For example, a
successful launch on Linux with X11 supplies evidence for that display backend.
Wayland requires its own run.

The [terms of reference](terms-of-reference.md) defines the user outcomes. The
[runtime design](tauri-runtime-servo-design.md) specifies the proposed
contracts and the [roadmap](roadmap.md) identifies the evidence needed to
qualify them.
