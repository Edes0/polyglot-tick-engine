# Decisions

[← back to the index](../README.md)

Eleven decisions this engine rests on, each on one screen: what forced it, what I chose instead of
what, what it cost, and what it taught me. Every card links to the page that tells it in full.

A card is here only if it had a real fork — at least one option rejected for a stated reason — a
constraint or incident that forced it, and a cost I accepted. Fixes without a fork live in the pages
themselves.

**Execution** · [1 Containers](#1-untrusted-code-goes-in-a-container) ·
[2 Workers](#2-a-long-lived-worker-per-player) · [3 Lockstep](#3-lockstep-inside-the-tick) ·
[4 Fail closed](#4-fail-closed-configuration) · [5 Compile once](#5-compile-at-upload-not-per-tick)
— **Correctness and trust** · [6 Trust boundary](#6-the-server-is-the-trust-boundary) ·
[7 Public board](#7-the-static-world-is-public) · [8 Rotations](#8-exempt-rotations-from-cycle-exclusion)
— **Performance** · [9 Instrument first](#9-instrument-before-refactoring)
— **The world** · [10 Client engine](#10-one-client-engine-kept-out-of-four) ·
[11 No seams](#11-terrain-is-a-function-of-global-coordinates)

---

### 1. Untrusted code goes in a container

<!-- budget:card max=150 -->
> In the context of running players' arbitrary code every tick, facing crashes, runaway memory and
> language sprawl, I chose a Docker sandbox per player over in-process runtimes, accepting the cost
> of a container round-trip.

| | |
|---|---|
| **Problem** | The first executor ran bots inside the server: WebAssembly, Python.NET and a V8 host. |
| **Options** | In-process runtimes (rejected: one bad bot shares the server's process) · **containers** |
| **Chosen** | One container per player, with a CPU budget, a hard kill, a memory ceiling and an output cap. |
| **Outcome** | One contract across 75 images, and failures that stay per-player. |
| **Lesson** | Isolation you get from the operating system beats isolation you maintain yourself, three runtimes over. |
<!-- /budget -->

Full story: [Sandboxed polyglot execution](02-polyglot-sandbox.md).

### 2. A long-lived worker per player

<!-- budget:card max=150 -->
> In the context of a 1,000 ms tick shared by every player, facing 50–100 ms per `docker exec`, I chose
> one persistent worker per player over a process per tick, accepting that the host now owns
> liveness, a protocol version, and hanging up on silent workers.

| | |
|---|---|
| **Problem** | Process start-up plus an HTTP round-trip per player per tick was most of the budget. |
| **Options** | Exec per tick (rejected: cost) · **a warm worker over a versioned JSON-line protocol** |
| **Chosen** | TCP by default, stdio as fallback, the snapshot captured when work is scheduled. |
| **Outcome** | ~1–5 ms per player per tick instead of 50–100 ms. |
| **Lesson** | Read state when the work is scheduled, not when it runs, or scheduling order leaks into results. |
<!-- /budget -->

Full story: [The persistent-worker protocol](03-persistent-worker-protocol.md).

### 3. Lockstep inside the tick

<!-- budget:card max=150 -->
> In the context of a world that must be deterministic, facing script results that landed after their
> tick had closed, I chose to await every script inside the tick over asynchronous calls, accepting
> that the slowest script bounds the tick.

| | |
|---|---|
| **Problem** | My first design called scripts asynchronously; intents arrived a tick late, or in an overflow buffer, or never. |
| **Options** | Async with buffers (rejected: determinism gone) · **lockstep** |
| **Chosen** | The tick waits for every script; a 200 ms hard kill caps the wait. |
| **Outcome** | Same inputs, same world, enforced by replay tests. |
| **Lesson** | A latency trade is only acceptable when something bounds the tail. |
<!-- /budget -->

Full story: [System overview](01-system-overview.md#lockstep-in-tick-and-the-design-i-reversed).

### 4. Fail-closed configuration

<!-- budget:card max=150 -->
> In the context of a server whose whole job is running scripts, facing a fresh clone that started,
> ran nothing and logged errors nobody read, I chose to refuse to start over starting degraded,
> accepting that every validation error is now fatal.

| | |
|---|---|
| **Problem** | The default configuration ran no scripts, and the validator's errors did not stop start-up. |
| **Options** | Warn and continue (rejected: silent failure) · **abort start-up** |
| **Chosen** | Persistent workers on by default; any validator error, or a missing image mapping, stops the server. |
| **Outcome** | No silent no-op servers. A language without a worker cannot run: 21 of 24 today. |
| **Lesson** | A fallback nobody tests is a second system that is already broken. |
<!-- /budget -->

Full story: [Sandboxed polyglot execution](02-polyglot-sandbox.md#one-contract-built-for-twenty-four-languages).

### 5. Compile at upload, not per tick

<!-- budget:card max=150 -->
> In the context of compiled languages inside a 1,000 ms tick, facing a recompile on every execution,
> I chose to build once at upload into a persisted artifact, accepting a build state the project must
> pass before it can run.

| | |
|---|---|
| **Problem** | Compiled bots were rebuilt inside the run container every tick, which blew the budget. |
| **Options** | Compile per tick (rejected: cost) · **build once** |
| **Chosen** | A `Built` status; activation requires it; success means a non-empty artifact, not just compiler exit 0. |
| **Outcome** | Compile cost paid once per upload. |
| **Lesson** | "Exit code 0" is a claim; the artifact is the evidence. |
<!-- /budget -->

Full story: [Sandboxed polyglot execution](02-polyglot-sandbox.md#one-contract-built-for-twenty-four-languages).

### 6. The server is the trust boundary

<!-- budget:card max=150 -->
> In the context of a game for programmers, facing private data on the wire that the client merely
> did not render, I chose per-player projection on the server over filtering in the client, accepting
> that every full-state path must go through it.

| | |
|---|---|
| **Problem** | Owner-private unit tags were broadcast to rivals; reconnecting shipped the whole world. |
| **Options** | Client-side filtering (rejected: readable with a proxy) · **server-side projection** |
| **Chosen** | Private fields removed from the wire DTO; full-state keyed on the authenticated player only. |
| **Outcome** | Regression tests assert on the serialised bytes, not on what renders. |
| **Lesson** | Client-side filtering is decoration, not a boundary. |
<!-- /budget -->

Full story: [The trust boundary](09-trust-boundary.md).

### 7. The static world is public

<!-- budget:card max=150 -->
> In the context of fog of war, facing scouts that walked into mountains forever and colonies whose
> economy depended on scouting luck, I chose a public board over fogging everything, accepting that
> exploration no longer finds resources.

| | |
|---|---|
| **Problem** | Fogged terrain made unexplored walls and open ground indistinguishable; hidden resources made spawning a dice roll. |
| **Options** | Fog everything (rejected) · **terrain and resource positions public, everything dynamic fogged** |
| **Chosen** | Generated amounts public; live amounts, units and structures stay fogged. |
| **Outcome** | Scouts stopped wedging; colonies stopped depending on luck. |
| **Lesson** | Hiding what the engine already reveals protects nothing. |
<!-- /budget -->

Full story: [The trust boundary](09-trust-boundary.md#where-the-line-is-stated-once).

### 8. Exempt rotations from cycle exclusion

<!-- budget:card max=150 -->
> In the context of move-conflict resolution, facing a closed ring of units that froze while every
> stage's tests passed, I chose to exempt tight rotations over excluding every cycle, accepting that
> the exemption does not yet cover the slowest units.

| | |
|---|---|
| **Problem** | Cycles were excluded wholesale; the two-unit "winner" was neutralised downstream. |
| **Options** | Exclude all cycles (rejected: rings freeze) · **exempt rotations** |
| **Chosen** | Rotations move, because end positions are a permutation of start positions. |
| **Outcome** | Rings rotate; the remaining limit is written down. |
| **Lesson** | A green unit test on a pipeline stage is not evidence about the pipeline. |
<!-- /budget -->

Full story: [Tick pipeline and determinism](04-tick-pipeline-and-determinism.md#total-order-or-it-is-not-deterministic).

### 9. Instrument before refactoring

<!-- budget:card max=150 -->
> In the context of a 13× regression on green tests, facing a written, plausible explanation of the
> allocation, I chose to instrument before refactoring, accepting a regression halved rather than
> closed.

| | |
|---|---|
| **Problem** | Move preparation went from 216.9 ms to 2,821 ms at 5,000 entities. |
| **Options** | Refactor what I believed was allocating (rejected: 2.8% of it) · **measure per call site** |
| **Chosen** | Three levers, each justified by an attribution; `WorldPosition` became a `readonly record struct`. |
| **Outcome** | 1,409 ms and 99 MB, down from 2,821 ms and 450 MB; three other optimisations measured and not done. |
| **Lesson** | An observable symptom gets an instrument, not a second hypothesis. |
<!-- /budget -->

Full story: [Performance engineering](05-performance-engineering.md).

### 10. One client engine kept out of four

<!-- budget:card max=150 -->
> In the context of a solo first game that needed a look and a browser build, facing engines that were
> either too limited or too heavy, I chose Unity 3D over Godot and Unreal, accepting three client
> rewrites on the way.

| | |
|---|---|
| **Problem** | Godot was a poor fit for a C# backend and 3D; Unreal was more engine than one person could learn. |
| **Options** | Godot (left after six days) · Unreal (left: no browser build, too slow to learn solo) · **Unity** |
| **Chosen** | Unity 3D with the Universal Render Pipeline. |
| **Outcome** | Each switch cost client code and backend reshaping, never the game rules. |
| **Lesson** | An authoritative server makes changing your mind about the client affordable. |
<!-- /budget -->

Full story: [The world and its look](11-world-and-visual-identity.md#four-engines-one-kept).

### 11. Terrain is a function of global coordinates

<!-- budget:card max=150 -->
> In the context of a world stored as rooms, facing discontinuities wherever a room's generation
> met its neighbour's, I chose terrain as a pure function of global coordinates over per-room
> overlays, accepting erosion run over the whole field.

| | |
|---|---|
| **Problem** | Anything generated per room had an edge at the seam. |
| **Options** | Per-room overlays (rejected: seams) · **one global function, sliced into rooms** |
| **Chosen** | No room-local overlay; erosion runs once over the whole field. |
| **Outcome** | A seamless world; byte-identical output at a single room. |
| **Lesson** | Keep the partition in the engine and out of the world. |
<!-- /budget -->

Full story: [The world and its look](11-world-and-visual-identity.md#the-maps-shape).

---

[← back to the index](../README.md)
