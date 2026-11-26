# Architectural decision record (ADR) 0002: Toolchain scope for the engine graph

## Status

Proposed.

## Date

2026-11-26.

## Context and problem statement

This repository builds with a dated nightly compiler, the Polonius alpha borrow
checker (`-Zpolonius=next`), a development code generator, and the mold linker.
The [Polonius policy](../polonius.md) records that contract. Its Makefile
applies at least four distinct flag sets: development, Whitaker, release, and
coverage.

Servo pins its own stable compiler for each release: Rust 1.97.1 for 0.7.0 and
Rust 1.99.1 for 0.8.0 and current main. It is a very large dependency that
includes the SpiderMonkey JavaScript engine. The runtime will also carry a
patch series against it.

If the workspace flags apply to the whole dependency graph, each flag set is a
separate full engine build, each nightly bump rebuilds the engine, and engine
code compiles under an analysis and a compiler its authors do not test. The
patch series must still build on Servo's own compiler, or it cannot be compared
with the upstream control. The combined build has never been attempted.

## Decision drivers

- The cost of a clean build, an incremental build, and a pin bump.
- Confidence that engine behaviour matches the upstream release.
- Preserving the project's borrow-centric API contract for its own code.
- Keeping the patch series buildable against the upstream control.

## Options considered

### Option A: One toolchain for the whole graph

Build Servo, Tauri, and this project with the workspace nightly and flags. It
is the simplest configuration. It multiplies full engine builds by the number
of flag sets, ties engine rebuilds to nightly bumps, and compiles the engine
with an untested compiler.

### Option B: Scope the workspace flags to this project's crates

Build the engine graph without workspace-specific flags, and apply the nightly
flags only to this project's crates. A single compiler must still build one
linked binary, so the option has two arrangements:

- B1, per-package flags: the workspace nightly builds the whole graph, with
  the project-specific flags confined by package. It needs a nightly that
  accepts the engine unchanged, and the engine is not built by Servo's release
  compiler.
- B2, prebuilt artefact: Servo's release compiler builds the engine as an
  artefact that is linked into the final build.

### Option C: Keep the adapter free of Polonius-only code

Write the engine host and runtime so that they compile under the stable borrow
checker, and build the whole graph with Servo's release compiler. The project's
other crates keep the nightly contract. It removes the conflict entirely for
the code that links the engine and gives up the Polonius contract for that code.

| Topic                             | Option A | Option B1    | Option B2    | Option C     |
| --------------------------------- | -------- | ------------ | ------------ | ------------ |
| Full engine builds per flag set   | One each | One in total | One in total | One in total |
| Engine rebuilt on a nightly bump  | Yes      | Yes          | No           | No           |
| Engine compiler matches upstream  | No       | No           | Yes          | Yes          |
| Polonius contract for the adapter | Kept     | Kept         | Kept         | Given up     |
| Build configuration complexity    | Lowest   | High         | Highest      | Moderate     |

_Table 1: Comparison of toolchain arrangements._

## Decision outcome / proposed direction

Prefer Option B2 if roadmap task 1.2.1 shows that a prebuilt engine artefact is
practical. Otherwise prefer Option B1 if per-package flags are practical and
the engine passes its upstream control under the workspace nightly. Fall back
to Option C if neither is practical. Reject Option A unless measurement shows
its build cost is acceptable and the engine passes its upstream control under
the workspace compiler.

Roadmap task 1.2.1 measures clean and incremental wall time, peak link memory,
target-directory size, and hosted-runner feasibility for each arrangement, and
the cost of one pin bump. Accept this record with those figures.

## Known risks and limitations

- Cargo applies `RUSTFLAGS` to every crate in the graph; confining flags by
  package needs configuration that may not cover every flag.
- A prebuilt engine artefact is not a stable interface across compilers, as
  the design's packaging section notes.
- Option C divides the repository's borrow-checker contract by crate, which
  the Polonius policy must then state.

## Outstanding decisions

- Whether the pinned nightly builds Servo 0.8.0 at all.
- Whether the release build of a distributed application may differ in
  compiler from the tested development build.
