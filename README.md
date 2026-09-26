# Screeps2 — an authoritative tick engine for untrusted bots, built for 24 languages

<!-- budget:hook max=90 -->
**Every second, the server must run every player's bot — code I have never seen — without letting
one stall the tick, see what it should not, or break determinism.** A .NET 10 backend owns the world
and sandboxes each bot in Docker, built for 24 languages; 1,603 .NET tests passed in CI on 2026-09-20.

Designed and built solo over ~10 months by Andreas Sjögren — I make the decisions and review every merge; AI coding agents implement under gates I built (see Working with agents).

Backend .NET engineer.
<!-- /budget -->

A write-up of a private project; no buildable source ([why](#why-there-is-no-source-here)).

<!-- budget:shows max=100 -->
**What this project shows**

- **I measure before I believe myself.** My written explanation of a 13× regression covered 2.8% of the allocation; per-call-site counters put 94.5% in the pathfinder. [→](docs/decisions.md#9-instrument-before-refactoring)
- **I reverse my own designs.** Async bot calls lost determinism, so the tick now waits for every script behind a 200 ms kill. [→](docs/decisions.md#3-lockstep-inside-the-tick)
- **Isolation, then speed.** Bots moved out of the server into containers, then into warm workers: 50–100 ms → 1–5 ms per player per tick. [→](docs/decisions.md#2-a-long-lived-worker-per-player)
- **The client is hostile.** Two leaks fixed by removing data from the wire, not the screen. [→](docs/decisions.md#6-the-server-is-the-trust-boundary)
<!-- /budget -->

**30 seconds:** this block · **5 minutes:** [the decisions](docs/decisions.md) · **30 minutes:** the pages below.

![The world as a bot sees it: eroded terrain, a fog-of-war clearing around the colony, a hatchery, and a procedurally-legged drone](media/04-tuned.png)

---

## What it is

Screeps2 is a programmable idle game. A player writes a bot and it runs forever on a shared world.
The player drops in to watch the colony work, tweaks the bot, and drops out again. There is no
client-side simulation: a .NET 10 backend owns the world, advances it once per second, and the Unity
client renders the deltas.

The interesting part is not the game. It is what the game forces you to build:

- **Untrusted code, on purpose.** Every tick, every player's arbitrary code executes in a Docker
  container with a CPU budget, a hard kill, a memory ceiling and an output cap. One player's
  infinite loop cannot delay another player's results, and cannot delay the tick.
- **Built for 24 languages, one contract.** bash, C, C++, C#, F#, Clojure, Go, Groovy, Haskell, Java,
  JavaScript, Kotlin, Lua, OCaml, Perl, PHP, PowerShell, Python, R, Ruby, Rust, Scala, Swift,
  TypeScript — 75 Docker images across the version matrix. The host does not know which language
  it is talking to. Python, JavaScript and TypeScript run on the live worker protocol today;
  [the rest are next](docs/02-polyglot-sandbox.md#one-contract-built-for-twenty-four-languages).
- **A 1,000 ms wall-clock budget** that has to contain container round-trips, pathfinding for every
  unit, conflict resolution, persistence, per-player visibility projection, and the broadcast.
- **Determinism as a hard invariant.** Same inputs, byte-identical outputs — enforced by replay
  tests, not by convention.

## What's worth reading

| Piece | The problem | Why you should care |
|---|---|---|
| [Decisions](docs/decisions.md) | Eleven forks this engine rests on | Each on one screen: the constraint, the rejected option, the cost accepted |
| [01 System overview](docs/01-system-overview.md) | Script results arrived after their tick closed | I reversed my own async design for lockstep, and bounded the cost with a hard kill |
| [02 Sandboxed polyglot execution](docs/02-polyglot-sandbox.md) | Code I have never seen, in any of 24 languages | Chose one language-neutral contract over per-language branches; adding a language never touches the tick |
| [03 Persistent-worker protocol](docs/03-persistent-worker-protocol.md) | A process per player per tick cost 50–100 ms | Chose a warm worker and a versioned line protocol; a scheduling race closed by construction |
| [04 Tick pipeline and determinism](docs/04-tick-pipeline-and-determinism.md) | One unordered tie-break breaks "same inputs, same world" | Chose a total order ending in a unique id — and found two bugs no stage test could see |
| [05 Performance engineering](docs/05-performance-engineering.md) | A 13× regression, and my wrong explanation of it | Chose to instrument first; halved it, and left three optimisations undone on purpose |
| [06 Fog of war](docs/06-fog-of-war.md) | A rule on the server and an effect on the client | Chose one fullscreen pass over per-shader fog, and duplicated one function deliberately |
| [07 Domain model](docs/07-domain-model.md) | Stale comments nearly bought a 120-file refactor | Chose ADRs with a `revisit_if` and an API that never does the player's programming |
| [08 Testing and CI](docs/08-testing-and-ci.md) | A green .NET build said nothing about the bots' code | Chose separate CI lanes for the in-container workers; wiring is proven on a running host |
| [09 The trust boundary](docs/09-trust-boundary.md) | Private data on the wire, merely not rendered | Chose server-side projection over client filtering, and made the public/private line explicit |
| [10 Working with AI agents](docs/10-working-with-agents.md) | Agents are confident, fast and forgetful | Chose to treat every audit as a claim; wrong audits were caught before they became refactors |
| [11 The world and its look](docs/11-world-and-visual-identity.md) | A solo first game needed a look and a seamless map | Kept one client engine out of four; hexagons gave way to one continuous world |
| [Player API reference](https://edes0.github.io/polyglot-tick-engine/player-api/) (live) | Bots need an API that helps without playing for them | Kept the public surface small on purpose; hand-written for people who write bots |

## The tick

```mermaid
flowchart LR
    A([Tick starts]) --> B[Resolve<br/>players] --> C[Snapshot<br/>at enqueue] --> D[Execute bots<br/>in containers]
    D --> E[Reconcile<br/>intents] --> F[Simulate] --> G[Project<br/>visibility] --> H[Persist<br/>one flush] --> I[Broadcast<br/>deltas]
    I --> A

    D -.->|timeout / crash| X[Isolate that player,<br/>tick continues] -.-> E

    style X stroke-dasharray: 4
```

A player's script runs **once per tick against a frozen snapshot** and returns *intents* — "move
toward here", "gather that". The engine decides what actually happens. Intents apply on the same
world tick they were computed for; there is no background queue and no drain-results-next-tick.

## By the numbers

| | |
|---|---|
| Backend | 942 C# files, ~80k LOC across Domain / Application / Infrastructure / Presentation and a small script-tester tool |
| Sandbox | Built for 24 languages: 75 Docker images, 181 files in the execution layer alone; 3 languages live on the worker protocol |
| Tests | **1,603 passing .NET test cases**, plus 222 pytest and 25 Node cases covering the in-container worker entrypoints (CI, 2026-09-20) |
| Benchmarks | 9 BenchmarkDotNet suites, 4 committed baselines dated May–July 2026 |
| Client | Unity 6 / URP, 122 scripts, six assembly-definition modules in a compile-time-enforced DAG |
| History | 379 commits, 222 merged PRs, first commit 2025-11-02, solo |

**Scale caveat, stated up front:** this is pre-alpha with no external players. Every performance
number in this repository was measured at 100 / 1,000 / 5,000 simulated entities on one desktop
(Ryzen 7 7700X, .NET 10). They are not production throughput figures and are not presented as any.
What they demonstrate is the measurement discipline — see
[Performance engineering](docs/05-performance-engineering.md).

## Stack

.NET 10 · ASP.NET Core · EF Core · MediatR · Docker · WebSockets / SignalR · xUnit ·
BenchmarkDotNet · Unity 6 (URP, HLSL) · GitHub Actions

## Why there is no source here

Keeping the source private is a decision, not an omission. The engine is a game I intend to keep
building and eventually run, and publishing it would mean publishing a working server that anyone
could stand up as their own. So this repository carries the parts that survive being read rather than
executed: architecture, protocols, decisions, measurements, and short excerpts of the real code —
quoted with the path they came from, so you can see what the code actually looks like without
receiving a copy of it.

If you are evaluating me and want to read more of it, ask — [sjogrenandreas@live.se](mailto:sjogrenandreas@live.se).

## Licence

Prose, diagrams and images: CC BY-NC-ND 4.0. Code excerpts: all rights reserved, reproduced here
for illustration only. See [LICENSE.md](LICENSE.md).

---

**Andreas Sjögren** — [sjogrenandreas@live.se](mailto:sjogrenandreas@live.se) · [github.com/Edes0](https://github.com/Edes0)
