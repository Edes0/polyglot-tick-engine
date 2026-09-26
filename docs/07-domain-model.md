# Domain model

[← back to the index](../README.md)

Eight thousand lines that depend on nothing, a handful of decisions I wrote down before I could
forget why, and one value object that changed shape because of a measurement.

<!-- budget:inshort max=60 -->
> **In short.** **Problem:** a domain rots through stale comments and convenient APIs. **Decision:** a
> Domain layer that depends on nothing, ADRs that carry a `revisit_if`, and one API rule — hide engine
> mechanics, never do the player's programming. **Outcome:** tombstone comments that nearly bought a
> 120-file refactor were caught, and growing the world is one config change.
<!-- /budget -->

---

## The layer that depends on nothing

`Screeps2.Domain` references no EF, no MediatR, no ASP.NET, no IO. 116 files: 10 aggregate roots,
33 value objects, 11 domain events, domain services, and a `Result`-based error model rather than
exceptions for rule violations.

| Aggregate roots | |
|---|---|
| `WorldAggregate`, `RoomAggregate` | The world and its internal grid partition |
| `UnitAggregate`, `StructureAggregate` | The things in it |
| `PlayerAggregate`, `UserAggregate` | Who owns them |
| `PlayerMemoryAggregate` | The player's opaque per-tick blob |
| `CpuBucketAggregate` | The Screeps-style CPU budget |
| `PlayerVisibilityAggregate`, `PlayerLastSeenMemoryAggregate` | What each player currently sees, and what they remember seeing |

Splitting *visible now* from *last seen* into two aggregates is a domain decision, not a storage
one. A structure you saw ten ticks ago and a structure you can see now are different facts with
different lifetimes, and a hostile unit is deliberately modelled as visible-only — it drops out the
moment it is fogged and is never remembered. Keeping those in one type would have made "can this
player read this?" a runtime question instead of a structural one.

CQRS is present but unremarkable: about 30 MediatR handlers over a template-method base. The
interesting work is in the tick pipeline, not the mediator.

## A value object with an argument written into it

`WorldPosition` is 3D, gameplay is 2D, and that gap is exactly the kind of thing that decays into a
half-implemented vertical gameplay layer three years later. So the reasoning lives in the type:

```csharp
// src/Screeps2.Domain/GameWorld/ValueObjects/WorldPosition.cs
/// <summary>
/// A <c>readonly record struct</c>: value semantics, no per-instance heap allocation on the
/// move-prep hot path. Structural equality (X, Y, Z) is identical to the former record class;
/// "absent" positions are modelled as <c>WorldPosition?</c>.
/// </summary>
public readonly record struct WorldPosition
{
    public float X { get; init; }

    /// <summary>
    /// Y coordinate. <b>VISUALIZATION-ONLY — domain code MUST NOT read this field for gameplay
    /// logic.</b> Every gameplay distance check uses <see cref="HorizontalDistanceTo"/> (X/Z only);
    /// pathfinding operates on (X, Z) grid anchors. Persisted for forward compatibility but unused
    /// by any gameplay rule. The Unity client computes display Y from a terrain raycast at render
    /// time with a per-entity-type offset. If you find yourself wanting Y in a domain service, add
    /// a discrete elevation-tier field instead — overloading this float will create a half-broken
    /// vertical gameplay layer.
    /// </summary>
    public float Y { get; init; }

    public float Z { get; init; }
    // …
}
```

"If you find yourself wanting Y, add a different field instead" is the sentence that does the work.
It names the temptation, the failure it leads to, and the correct alternative — which is more than a
`// TODO: don't use this` achieves, and it is attached to the only thing that always travels with
the field.

It became a struct for a measured reason: 1.21 million constructor calls in a single benchmark run
were 23.5% of remaining allocation, and the conversion cut allocation 16.5% at every scale with time
flat. The design change and the number that justified it are both in
[performance engineering](05-performance-engineering.md#lever-three-a-design-change-justified-by-an-attribution).

## Three decisions I recorded

The project keeps ADRs with `status`, `date` and — the field that earns its place — `revisit_if`: the
concrete condition under which the decision should be reopened. A decision without one silently
becomes permanent.

### Rooms are a compute partition, never a player concept

The world is a grid of rooms. Per-room occupancy tables, per-room A*, per-room fog — that partition
is what keeps tick cost tractable. Bots never see it: no room name on any player-facing surface, no
room concept in the API, one continuous coordinate space.

This one is in the record partly because **I nearly deleted it by mistake.** Around 25 stale comments
across the codebase read "Room is obsolete, replaced by PCG Chunks". They were wrong — what had
actually been replaced was legacy hex-room *generation*, not the partition. Both a reading agent and
I concluded rooms were dormant, and a full collapse of the partition was seriously considered before
being costed at 120+ files.

Stale comments are not untidiness. These ones nearly bought a multi-week refactor of a load-bearing
abstraction. The ADR exists so the next reader gets the truth instead of the tombstones.

### World bounds are deliberately not exposed to bots

A scout that cannot distinguish "never explored" from "does not exist" paces at the world border
forever. The obvious fix is to hand bots the world radius — the extent already rides a constants DTO,
so exposing it would be nearly free.

It stays hidden. Bots discover the edge by querying cells: off-world reads as `"mountain"`, and a
cell is only frontier if it is both unseen *and* walkable terrain.

The payoff is that growing the world is one config change — no bot edit, no worker image rebuild, no
coordinated release. Had the radius been a player-visible constant, every bot would have hard-coded
it and world growth would be a breaking change to everyone's script. That is the whole argument for
minimal public surfaces, in one concrete case, at a cost of one extra call in the bot.

### Body is an ordered token array, and the old format throws

A unit's body was a count map, `{"MOVE": 2}`. Then ordering became load-bearing — the front part
dies first — so it became an ordered array, `["TOUGH", "MOVE", "WORK"]`.

The deliberate part is the absence of a compatibility shim. Deserialising the old format **throws**.
The alternative considered and rejected was returning an empty body, which silently zeroes every
unit's body and then violates the carry-capacity and hit-point invariants everywhere downstream —
a botched rollback that looks like a gameplay bug in a different subsystem a week later.

Throwing is the right call *because* the migration path is a clean database reset and there are no
real players yet. The ADR's `revisit_if` says so explicitly: reopen this when a real player
population exists and a reset is no longer acceptable. The decision is correct for now and it is
recorded as correct-for-now.

## The API design line

There is one governing rule for the player-facing surface: **hide engine mechanics, never do the
player's programming.**

That cuts in both directions and the second half is the harder one. The API deliberately does not
ship `findClosest`, or derived convenience fields, or query helpers — not because they are difficult,
but because pathfinding to the nearest thing *is the game*. An engine that hands you the answer has
converted a programming game into a configuration game.

What it does hide: rooms, world bounds, tick scheduling, container mechanics, the internal
coordinate partition. A bot sees one continuous world and its own colony.

The full reference is [published live](https://edes0.github.io/polyglot-tick-engine/player-api/) —
it is written for bot authors, including bot authors who are prompting an LLM rather than typing,
which is a real constraint on naming and shape.

---

**Next:** [Testing and CI →](08-testing-and-ci.md)
