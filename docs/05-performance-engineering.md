# Performance engineering

[← back to the index](../README.md)

The useful story here is not a number that got smaller. It is the time I was confident about the
cause, wrote it down, and was wrong — and what the instrument said instead.

<!-- budget:inshort max=60 -->
> **What it does.** Measures the tick, against committed baselines. **How it works.** BenchmarkDotNet
> at 100 / 1,000 / 5,000 entities, plus per-call-site allocation counters. **Result.** Move preparation
> from 2,821 ms / 450 MB to 1,409 ms / 99 MB at 5,000 entities; fog −38%; broadcast 2.0–2.3× faster.
<!-- /budget -->

**Every measurement below:** one desktop, Ryzen 7 7700X, .NET 10, BenchmarkDotNet with the
in-process toolchain, 5 warmup / 15 iterations, at 100 / 1,000 / 5,000 simulated entities. This is a
pre-alpha project with no external players. These are not production throughput figures.

---

## First: measure before optimising anything

An audit produced three candidate optimisations for the tick hot path. Rather than implement them, I
benchmarked them against a 1,000 ms tick budget:

| Candidate | Measured cost | Share of budget |
|---|--:|--:|
| A* open-set scan (`SelectBest` → `PriorityQueue`) | 16.6 µs per path, **flat** across entity counts | negligible |
| Broadcast delta dictionary rebuild | 2.29 ms @5000, 632 KB | 0.23% |
| Reservation table freeze | 4.54 ms / 3.5 MB per build, ~2 builds per tick | ~0.4% |

And the pass they all live inside:

| | @100 | @1000 | @5000 | Alloc @5000 |
|---|--:|--:|--:|--:|
| Per-tick move preparation | 4.56 ms | 67.6 ms | **216.9 ms** | **44.6 MB** |

Move preparation was two orders of magnitude more expensive than any of the three proposed fixes. So
I did not implement them, wrote down why with the numbers attached, and parked the spec. At current
scale — low hundreds of units — the entire pass costs under 5 ms and the tick is nowhere near
budget.

Not shipping three optimisations is the result I am most confident was correct.

## Then: a rewrite landed, and nobody re-baselined it

A later change replaced per-room pathfinding with the windowed-global architecture. Re-running the
same benchmark on master:

| `CrowdedMovePreparation` | @100 | @1000 | @5000 | Alloc @5000 |
|---|--:|--:|--:|--:|
| before the rewrite | 4.56 ms | 67.6 ms | 216.9 ms | 44.6 MB |
| after the rewrite | 5.0 ms | 145 ms | **2,821 ms** | **450 MB** |

Thirteen times slower, ten times the allocation, sitting on master, green tests. No test asserts a
duration. The only reason it surfaced is that a committed baseline existed to re-run — which is the
argument for writing the baseline down in the repository rather than keeping a number in your head.

### Lever one: the O(N²) hiding inside an inner loop

`BuildTrafficPenaltyMap` ran inside *every* A* search. At 5,000 entities that is roughly **11,700
searches per tick**, each rebuilding a map that is O(all reservations) ≈ O(N). An O(N²) term nobody
wrote on purpose.

The map is a pure function of a table's reservations, so it now gets built once per table, cached,
and invalidated on add. The dictionaries are pooled, so the replan loop — which mutates its table
between searches and therefore genuinely must rebuild — stays allocation-free.

**−37% time at 5,000, −20% at 1,000, byte-identical move outcomes.** 2,821 ms → 1,775 ms.

I also measured something I had planned to fix and then did not: the merged search window is built
only **4 times per tick** at 5,000 entities, not per round. The cost is per *search*, not per round,
so the round-level optimisation I had scheduled would have bought nothing. Dropped.

## The part worth reading: I was wrong, in writing

Lever one fixed time and did nothing for allocation — 450 MB → 451 MB. So I wrote down what I
believed the allocator was, in the baseline document, next to the numbers:

> The 10× allocation regression and the residual time gap are the global-frame machinery itself:
> `SpatialOccupancyService.ShiftToGlobal` allocates a `SpatialReservation` record plus a
> `FootprintBounds2D` per reservation per window build […]

Reasonable. It fits the evidence — the regression arrived with the global-frame change, and
`ShiftToGlobal` is the obvious per-reservation allocator in it.

It was wrong. Before acting on it I instrumented: throwaway
`GC.GetAllocatedBytesForCurrentThread()` deltas per call site over a single run at 5,000 entities.

| Site | Share of 450 MB |
|---|--:|
| `ShiftToGlobal` — my hypothesis | **2.8%** (12.7 MB — noise) |
| `GridAStarCore.Search` | **94.5%** |
| …of which goal-ring probing in `ResolveGoalAnchors` alone | **62.8%** |

The real cause was a type boundary. Every per-cell probe crossed from the integer grid back into
world space: `anchor.ToWorldPosition(worldY)` → `CanPlaceUnit(WorldPosition)` → `CanReserve`. That
heap-allocated a `WorldPosition` (a record *class*), a `FootprintBounds2D` (also a class), and a
candidate iterator with a lazy `HashSet` — **per probed cell**, across ~11,700 searches.

Nothing about reading the code would have told me the ratio was 2.8% against 94.5%. An afternoon of
confident refactoring would have bought 2.8%.

### Lever two: don't cross the boundary

A new probe computes footprint extents as locals and walks the cell index directly — no iterator, no
deduplication set, no world-space object. The object-based overloads delegate to the same primitive
implementations, so there is exactly one behaviour, and an equivalence test pins the new probe
against the old one.

| `CrowdedMovePreparation` | @100 | @1000 | @5000 | Alloc @5000 |
|---|--:|--:|--:|--:|
| after lever one | 5.6 ms | 116 ms | 1,775 ms | 451 MB |
| after the allocation probe | 2.8 ms | **82 ms** | **1,494 ms** | **116 MB** |

**Allocation −74%. Time −16% at 5,000, −30% at 1,000. Move outcomes byte-identical**, full suite
green including every golden and determinism test.

### Lever three: a design change justified by an attribution

With 116 MB left, the same instrument — per-type constructor counters over one run — attributed
23.5% of it to `WorldPosition` alone: 1.21 million constructor calls, ~38.6 MB. So `WorldPosition`
became a `readonly record struct`.

| Allocated | @100 | @1000 | @5000 |
|---|--:|--:|--:|
| record class | 2,264 KB | 23,750 KB | 118,987 KB |
| readonly record struct | **1,895 KB** | **19,832 KB** | **99,347 KB** |
| | −16.3% | −16.5% | **−16.5%** |

Time was flat — 1,430 → 1,409 ms, inside noise. This was a GC-pressure win and it is recorded as
one, not dressed up as a speedup.

The saving is ~19.6 MB against a 38.6 MB attribution, and the gap is explainable: a struct keeps its
payload inline where it is stored, so list backing arrays and dictionary entries still carry the
bytes. Only the object header, the indirection, and the fully transient locals disappear. Predicting
19.6 from 38.6 in advance would have been guessing; measuring both is cheap.

The same instrumentation also said `FootprintBounds2D` was 1.0%. It stayed a class. Converting it
would have been the same work for a fortieth of the benefit, against a real blast radius through
nullability, EF and DTO mapping.

## Where it actually landed, stated honestly

After three levers: **1,409 ms and 99 MB at 5,000 entities**, against 216.9 ms and 44.6 MB before
the pathfinding rewrite. The regression was reduced by half and is **not** closed. The remaining gap
is structural — the global-frame architecture does more work than per-room pathfinding did, and it
buys cross-room navigation that the old design could not do at all.

At the scale this project actually runs — low hundreds of units — move preparation costs a few
milliseconds against a 1,000 ms budget. The remaining work is written down, with its measurements,
for when entity counts make it real.

## Two more, briefly

**Per-player fog of war.** Caching the vision footprint of structures only: **6.69 ms → 4.17 ms
(−38%)** at 5,000 entities across 4 players, allocation flat. Unit footprints are deliberately *not*
cached — they move every tick, so the cache would miss more often than it hit. The reason is written
next to the code.

**Broadcast.** Serialising each player's delta once instead of once per connection: **2.0–2.3×
faster and half the allocation** (2,218 → 1,118 µs, 845 → 507 KB at 5,000 across 8 connections).
Sharing the start-of-tick world snapshot: −36% allocation, −12–17% time.

And one thing I measured specifically to decide whether to keep it: the broadcast path reloads world
state from the database each tick, which looks redundant. It costs **3.4 ms at 1,000 units — 0.34%
of the tick budget** — and it guarantees the broadcast cannot diverge from persisted truth. Measured,
then deliberately kept. An optimisation you can prove is not worth doing is as useful as one that is.

## Three optimisations I did not do

The three candidates at the top of this page are still unimplemented, on purpose, with their numbers
recorded so the decision can be re-litigated with evidence rather than re-argued from scratch. The
same applies to the [216 KB/tick of static terrain](03-persistent-worker-protocol.md#a-cost-i-measured-and-have-not-yet-paid-down)
in the worker protocol.

The rule I ended up with: an observable symptom gets an instrument, not a second hypothesis. The
first guess in this document was well-reasoned, well-informed, written down by someone who knew the
code — and off by a factor of thirty-four.

---

**Next:** [Fog of war →](06-fog-of-war.md)
