# The trust boundary

[← back to the index](../README.md)

This is a game for programmers. Anything the server sends a client, a player can read with a proxy
or a debugger. So the boundary is the server, and "hidden in the UI" is not hidden.

<!-- budget:inshort max=60 -->
> **In short.** **Problem:** two paths shipped information a player was not allowed to know; the
> client merely did not show it. **Decision:** project every payload per player on the server,
> keyed on who is authenticated, and make the public/private line explicit. **Outcome:** private data
> is absent from the wire, not dimmed, with regression tests that read the serialised bytes.
<!-- /budget -->

---

## Two leaks, one lesson

Both of these shipped, and both were found by reading the wire rather than the screen.

**Owner-private tags on every visible unit.** Bots can tag their own units — `role: scout`,
`mine: 1,2,3` — and those tags were on the broadcast unit DTO. The client only rendered your own, so
nothing looked wrong. But the broadcast goes to every player who can see the unit, so any rival with
a WebSocket client could read your colony's plan off your units.

**The whole world on reconnect.** The per-tick broadcast was already projected per player. The two
full-state paths were not: the seed sent when a client connects, and the REST endpoint a client calls
to resynchronise, both shipped every entity and every cell in the world to any authenticated client.
The fog of war held for exactly as long as nobody reconnected.

The fix for the first was structural rather than a filter: the tags field was deleted from the
broadcast DTO, so there is nothing to forget to strip. A regression test serialises a tagged unit and
asserts on the JSON itself:

```csharp
// tests/Screeps2.Infrastructure.Tests/Network/UnitToDtoMappingTagsTests.cs
[Fact]
public void ToUnitDto_TaggedUnit_DoesNotLeakTagsOnBroadcastWire()
{
    var unit = NewUnit();
    unit.ReplaceTags(new Dictionary<string, string> { ["role"] = "scout", ["mine"] = "1,2,3" });
    var dto = UnitToDtoMapping.ToUnitDto(unit);
    var json = JsonSerializer.Serialize(dto, JsonOptions.Default);
    using var doc = JsonDocument.Parse(json);
    Assert.False(
        doc.RootElement.TryGetProperty("Tags", out _),
        "Broadcast UnitDto must not carry owner-private tags — they leak to any client that can see the unit.");
}
```

The fix for the second routes both full-state paths through the same per-player projection as the
broadcast. The controller takes the player from the authenticated principal and nowhere else — never
from an id the client supplies:

```csharp
// src/Screeps2.Presentation/Controllers/WorldController.cs
[HttpGet("fullstate")]
public async Task<IActionResult> GetFullState(CancellationToken ct)
{
    if (!User.TryGetPlayerId(out var playerId, out var forbidden))
        return forbidden!;

    var delta = await _playerFullState.BuildForPlayerAsync(playerId, ct);
    if (delta == null)
        return NotFound("World not found.");
    // …
}
```

The lesson I wrote down at the time: **client-side filtering is decoration, not a boundary.** A value
a player may not know must not be serialised for that player. Dimmed, collapsed, or "not rendered" all
mean *sent*.

## Where the line is, stated once

Knowing what to hide is half the design. The other half is knowing what not to bother hiding, and
writing that down so it is a decision and not an oversight.

| | Public, always answerable | Fogged |
|---|---|---|
| Terrain type | everywhere, explored or not | — |
| Basic resource nodes | position and the map's generated amount | the live amount at a node you have seen |
| Units, structures, enemies | — | only what you can currently see |

**The static world is public; the dynamic world is fogged.** Terrain went public after a scouting bug.
With terrain fogged, an unexplored wall and unexplored open ground were indistinguishable, and a
mountain's interior is *permanently* unexplored because the mountain blocks sight into it — so scouts
aimed at the nearest "unexplored" cell, which was often inside a mountain, and walked into it
forever. The engine already routed units on true terrain through fog, so hiding it protected nothing.
It only stopped a bot predicting its own movement.

Resource positions went public for the same reason and one more: with them hidden, a colony's economy
depended on what its early scouts happened to reach. Two colonies running the identical script once
knew one patch and five patches respectively, and the first one's hatchery sat idle. Spawning and
scouting became a dice roll, which is not the game.
The generated amount is public because it is a constant of the map and leaks no activity; the *live*
amount stays fogged, because a draining patch would broadcast "someone is mining over there".

What that costs is written into the decision too: exploring no longer discovers where the resources
are, and publishing the board is a one-way door — the decision record calls it exactly that. Accepted
for now, explicitly, with the reason attached.

## Two credentials, two trust levels

A human and a bot container are different principals, so they authenticate differently:

- **A human session** uses an opaque bearer token.
- **A worker container** uses a JWT that expires after 60 minutes and is re-issued with the tick
  messages its worker receives.

They are two named ASP.NET authentication schemes behind two policies, not one token type with a
flag. Merging them would let a credential that only ever needs to act for one colony's bot stand in
for a person, which is the least-privilege argument in one sentence.

Both schemes emit the same player-id claim, so every downstream check reads the player the same way
whichever door the request came in by, and the controllers above take the player from that claim
rather than from anything the request says about itself.

## A guard that was fail-open in six places

Ownership — "does this player own this unit?" — was checked by hand in eight places across Domain,
Application and Infrastructure. Six of them were written as `OwnerId != playerId`.

That looks right, and on every path a player can actually reach it behaves right. But with value
objects and records, `null == null` is `true`. For an unowned aggregate and an absent caller, the
comparison said "owned", and those six sites would have let the action through. The two sites with
an extra null check were the correct ones — and a code audit had read it the other way round, marking
those two as the drifted copies.

There is now one guard, in Domain, fail-closed on both sides:

```csharp
// src/Screeps2.Domain/Common/Aggregates/Ownership.cs
public static bool IsOwnedBy(this IOwnable owned, PlayerId? playerId)
{
    ArgumentNullException.ThrowIfNull(owned);

    return playerId is not null && owned.OwnerId is not null && owned.OwnerId == playerId;
}
```

The explicit null checks are not decoration. The id type also has an implicit conversion to `Guid`
that throws on null, so the "obvious" rewrite of the comparison trades a silent accept for an
exception. The guard's doc comment says so, next to the code.

This was a latent defect, not an exploited one. It is in this document because the interesting part
is how it was found: by refusing to trust an audit's reading of which copies were the drifted ones.

## One piece of ordinary hygiene

The committed development JWT key is a placeholder, and a startup guard makes sure it can never be
the deployed one:

```csharp
// src/Screeps2.Presentation/Configuration/JwtSecretGuard.cs
public const string PlaceholderMarker = "ChangeInProduction";

public static void Validate(string? jwtSecretKey, bool isDevelopment)
{
    if (isDevelopment) return;

    var secret = jwtSecretKey ?? string.Empty;
    if (secret.Contains(PlaceholderMarker, StringComparison.OrdinalIgnoreCase))
    {
        throw new InvalidOperationException(
            "Game:PlayerApi:JwtSecretKey is still the development placeholder. A non-Development " +
            "deployment must set a strong secret via the Game__PlayerApi__JwtSecretKey " +
            "environment variable.");
    }
}
```

A test reads the *committed* `appsettings.json` and asserts the placeholder marker is still present —
so if anyone ever swaps the dev placeholder for a real-looking secret, the test fails and explains
why. The guard and the test cover opposite directions of the same mistake.

## What is still open

One path is knowingly ungated. The endpoint that serves room terrain and the world seed to the client
is not projected per player. Terrain is public by the rule above, so this leaks nothing a bot cannot
already ask for — but fog on the *client* is a visual effect laid over a full terrain mesh, and gating
that endpoint is a separate change I have deferred rather than forgotten.

---

**Next:** [Working with AI agents →](10-working-with-agents.md)
