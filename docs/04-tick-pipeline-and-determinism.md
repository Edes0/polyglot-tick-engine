# Tick pipeline and determinism

[← back to the index](../README.md)

Twelve ordered phases, one batched flush, and a total order over every decision — because "same
inputs, same world" is the property the whole game rests on.

<!-- budget:inshort max=60 -->
> **In short.** **Problem:** "same inputs, same world" is what the game rests on, and one unordered
> tie-break quietly breaks it. **Decision:** twelve explicit phases, one batched flush, and a four-level
> total order that ends in a unique id. **Outcome:** determinism enforced by replay and golden tests,
> not by a comment — and two bugs caught that no stage-level test could see.
<!-- /budget -->

---

## The phase list is the design document

The tick used to be one long method. It is now twelve `ITickPhase` implementations running in a
fixed sequence over a shared `TickPipelineState`. A phase returning `Halt` stops the pipeline and
must have set the result.

```csharp
// src/Screeps2.Infrastructure/Services/Tick/Pipeline/ITickPhase.cs
public interface ITickPhase
{
    string Name { get; }

    /// <summary>
    /// True on the first phase whose per-action command-handler saves must be deferred into the
    /// single end-of-tick flush. The orchestrator opens a mutation batch on reaching this phase and
    /// holds it open through the rest of the pipeline, so handler SaveChanges calls stage rather
    /// than flush mid-tick — keeping the change-tracker set intact for the phases that overlay it
    /// (hatchery regen, visibility). Phases before it (first-run spawn) still flush immediately, so
    /// their writes are visible to the same tick's DB reads (player resolution).
    /// </summary>
    bool BeginsMutationBatch => false;

    Task<TickPhaseOutcome> ExecuteAsync(TickPipelineState state, CancellationToken cancellationToken);
}
```

That one flag is the whole persistence strategy. Before it, writes flush immediately because a later
phase in the same tick needs to read them back. From it onward, every handler's `SaveChanges` stages
into one batch that flushes once — which keeps the EF change-tracker set intact for the phases that
overlay it, and means a tick is one database round-trip rather than one per action.

The order lives in DI registration, spelled out rather than resolved, with the reasons attached:

```csharp
// src/Screeps2.Infrastructure/Extensions/ServiceRegistration/PlayerServiceRegistration.cs
services.AddScoped<IReadOnlyList<ITickPhase>>(sp =>
[
    sp.GetRequiredService<SetupPhase>(),
    // First-run spawn runs before player resolution so a freshly-placed hatchery is visible to
    // the spawned-player filter this same tick.
    sp.GetRequiredService<FirstRunSpawnPhase>(),
    sp.GetRequiredService<PlayerResolutionPhase>(),
    sp.GetRequiredService<ScriptExecutionPhase>(),
    sp.GetRequiredService<ActionProcessingPhase>(),
    // Economy + build phases run post-action, pre-visibility so fog/broadcast see the result.
    // They touch disjoint state, so order among them is irrelevant — a unit hatched this tick
    // is too young for the lifespan cull, and BuildProgress checks ready before decrementing,
    // so a build never hatches the tick it started.
    sp.GetRequiredService<HatcheryRegenPhase>(),
    sp.GetRequiredService<UnitLifespanPhase>(),
    // Cull removes every 0-health unit and MUST follow UnitLifespan, which only zeroes an aged
    // unit's health and leaves the removal here (one remover, not two). Before Visibility so fog
    // and the broadcast see the corpse gone this tick.
    sp.GetRequiredService<UnitCullPhase>(),
    sp.GetRequiredService<BuildProgressPhase>(),
    sp.GetRequiredService<VisibilityPhase>(),
    sp.GetRequiredService<PersistencePhase>(),
    sp.GetRequiredService<FinalizationPhase>(),
]);
```

Those comments encode bugs that already happened. "One remover, not two" is there because of a bug
that ran the other way: combat never removed dead units at all. A unit could sit on the map at zero
health for up to 1,500 ticks — counted by its owner's bot as a healthy soldier, driving enemy army
production, standing in its cell as a permanent wall. The event meant to remove it had exactly one
consumer, and that consumer wrote a debug log line. Only old age removed units.

The fix is a cull phase that is the **only** remover — the lifespan phase lost its delete — so a new
damage source cannot forget to clean up after itself. Ordering constraints that are only in
someone's head are ordering constraints that get violated during the next refactor.

## Determinism is a test, not a comment

Every engine says it is deterministic. The question is what fails if it stops being true.

Here, two runs over identical inputs must serialise byte-equal:

```csharp
// tests/Screeps2.Domain.Tests/Tick/TickReplayDeterminismTests.cs
/// <summary>
/// Replay-determinism guard: two runs of identical inputs through the conflict-resolution path
/// must produce byte-equal output. Catches `Parallel.ForEach`, `HashSet` enumeration,
/// `DateTime.UtcNow`, or any other non-deterministic slip into the tick hot path.
/// </summary>
[Fact]
public void MoveConflictResolver_TwoRunsOfIdenticalInputs_ProduceByteEqualOutput()
{
    var scenario = BuildMoveScenario();

    var firstRunJson = SerializeMoveResolved(scenario);
    var secondRunJson = SerializeMoveResolved(scenario);

    Assert.Equal(firstRunJson, secondRunJson);
}
```

Alongside it: golden tests that pin resolved target positions and the rejection set over the real
pipeline on a 50×50 grid, plus dedicated determinism suites for the pathfinder and for visibility.

The banned constructs in Domain and Application are `DateTime.Now` / `UtcNow`, unseeded `Random`,
`Parallel.ForEach` on the hot path, and hash-set iteration order in any tie-break. The procedural
terrain generator carries seven written determinism rules in its doc comment — one seeded `Random`,
advanced only at droplet spawn, no parallelism, a whitelist of float operations — because erosion is
the easiest place in the codebase to accidentally introduce ordering sensitivity.

## Total order, or it is not deterministic

Two units want the same cell. Something must break the tie, and "whichever the dictionary yielded
first" is not something.

```csharp
// src/Screeps2.Application/Tick/MoveConflictAlgorithm.cs
private static readonly Comparison<MoveAction> MoveOrderComparison = (a, b) =>
{
    var playerOrderCmp = a.PlayerOrderForTick.CompareTo(b.PlayerOrderForTick);
    if (playerOrderCmp != 0)
        return playerOrderCmp;

    var sequenceCmp = a.IntentSequence.CompareTo(b.IntentSequence);
    if (sequenceCmp != 0)
        return sequenceCmp;

    var sourceTickCmp = a.SourceTick.CompareTo(b.SourceTick);
    if (sourceTickCmp != 0)
        return sourceTickCmp;

    return a.UnitId!.Value.CompareTo(b.UnitId!.Value);
};
```

Four levels, and the last one is a unit ID — a value that is unique by construction, so the
comparison is a **total** order and can never fall through to "equal, pick either". That final
tiebreak is the difference between deterministic and almost-deterministic.

Player order itself is not insertion order either: the runnable set is sorted by player ID and then
rotated by tick, so a cap on scripts per tick cannot starve the tail of the list.

Conflict resolution runs in three steps — occupancy projection, first-claim-per-target with
stationary blocking, then cycle detection.

Cycle detection is where a green test suite hid a frozen colony. Every unit in a cycle used to be
excluded from moving, with a canonical "winner" kept for the two-unit case. Each stage's unit tests
passed. End to end, the winner was neutralised anyway — its excluded partner stayed put, on exactly
the cell the winner meant to enter — so a closed ring of workers froze solid and re-issued the same
move forever.

Tight rotations are now exempt from exclusion. That is safe for a reason that can be stated in one
line: every participant steps into the cell the next one vacates, so the end positions are a
permutation of the start positions and nothing is doubly occupied.

```csharp
// src/Screeps2.Application/Tick/MoveCycleDetector.cs
if (pathIndexByUnit.TryGetValue(curId, out var cycleStartIndex))
{
    var cycleLength = pathUnits.Count - cycleStartIndex;
    if (isExemptCycle != null && isExemptCycle(pathUnits.GetRange(cycleStartIndex, cycleLength)))
        break;
```

The exemption needs every unit in the ring to complete its step, so it does not yet fire for the
slowest units — written down as a known limit rather than left for someone to rediscover. The lesson
I kept: **a green unit test on a pipeline stage is not evidence about the pipeline.** The freeze
survived because no test crossed the seam between stages.

## Pathfinding across a world that is secretly partitioned

The world is stored as a grid of rooms. Bots never see that: they work in one continuous global
coordinate space, and the fact that a partition exists is an engine implementation detail that never
reaches the player surface.

Which means the pathfinder cannot be per-room. It builds a merged search window — the axis-aligned
bounding box of start and goal, padded by 16 cells — and runs A* over it with octile movement costs
(10 orthogonal, 14 diagonal) and a corner-cutting rule. Node expansion is capped at 20,000; the
measured worst case, corner to corner across the current world, is 14,600. When a search fails
inside the window, there is one full-extent retry, because the failure mode that matters is a unit
in a pocket whose only route out leaves the window.

Scratch state comes from an `ObjectPool<PathfindingScratch>` rather than being allocated per search,
for reasons covered in [performance engineering](05-performance-engineering.md).

Traffic runs as a fixpoint on top: units that lose a conflict get exactly **one** deterministic
replan before they are made to wait, and the loop is bounded at 16 rounds. A fixpoint loop over
mutually blocking agents without a round ceiling is an outage waiting for the right map.

---

**Next:** [Performance engineering →](05-performance-engineering.md)
