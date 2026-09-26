# Sandboxed polyglot execution

[← back to the index](../README.md)

Every tick, code I have never seen runs on my server, and the sandbox is built to take it in
twenty-four languages. It gets one CPU budget, one hard kill, one memory ceiling, and no way to take
the tick down with it.

<!-- budget:inshort max=60 -->
> **In short.** **Problem:** code I have never seen, in any of the 24 languages the sandbox is built
> for, must never touch the tick or another player. **Decision:** one language-neutral contract — a language is a descriptor plus
> a command builder — and the host owns every timeout. **Outcome:** failures stay per-player, and adding
> a language never touches the tick.
<!-- /budget -->

---

## The shape of the problem

A player uploads a bot. The sandbox has to assume it might be Haskell. It might allocate until the box
swaps, fork until the process table is full, block on a socket forever, or return 400 MB of stdout.
The engine has to call it once per second, take whatever it produced, and be ready to do it again —
while another player's bot is doing something else wrong in parallel.

Three things fall out of that:

1. **The host must not know the language.** Twenty-four special cases in the tick loop is not a
   design, it is a backlog.
2. **The host must own every timeout.** A worker that reports its own runtime is a worker that lies
   when it hangs.
3. **Failure must be per-player.** Not per-tick, not per-server.

## One contract, built for twenty-four languages

bash · C · C++ · C# · F# · Clojure · Go · Groovy · Haskell · Java · JavaScript · Kotlin · Lua ·
OCaml · Perl · PHP · PowerShell · Python · R · Ruby · Rust · Scala · Swift · TypeScript

75 Docker images across the version matrix — Python alone spans 3.8 to 3.13, Rust four versions,
Java four. A language is a **descriptor plus a command builder**, registered into DI. Nothing in the
tick path branches on which one it is.

Python, JavaScript and TypeScript run on the live persistent-worker protocol today. The other 21 have
images and build pipelines; each is one worker entrypoint away — that port is the work in progress.

```csharp
// src/Screeps2.Infrastructure/Sandbox/ILanguageRuntime.cs
public interface ILanguageRuntime
{
    string LanguageName { get; }
    string[] FileExtensions { get; }
    string[] SupportedVersions { get; }
    string DefaultVersion { get; }

    LanguageCapabilities Capabilities { get; }
    LanguageMetadata GetMetadata();
    ResourceLimits GetDefaultLimits();

    IPlayerExecutor CreateExecutor(
        string scriptPath,
        string? version,
        IServiceProvider serviceProvider,
        string? buildArtifactPath = null);   // prebuilt artifact, or null -> compile per tick

    bool CanExecute(string filePath);
}
```

The second seam absorbs the compiled/interpreted split. A language provides up to four command
strings; which ones come back non-null *is* the classification:

```csharp
// src/Screeps2.Infrastructure/Sandbox/Docker/Commands/ILanguageCommandBuilder.cs
public interface ILanguageCommandBuilder
{
    string LanguageName { get; }

    /// Compile step for compiled languages. Null for interpreted languages.
    string? BuildCompileCommand(string scriptFileName, string scriptPath);

    /// Build-once compile writing a persisted artifact, for the upload-time pipeline -
    /// distinct from the per-tick temp-dir compile above. Null where there is no recipe yet.
    string? BuildArtifactCompileCommand(string scriptFileName, string artifactPath);

    /// Run step after compilation. Null for interpreted languages.
    string? BuildExecuteCommand(string scriptFileName, string scriptNameWithoutExt);

    /// Entry point for interpreted languages; may combine compile+execute for some compiled ones.
    string BuildEntryPointCommand(string scriptFileName, string scriptNameWithoutExt, string scriptPath);
}
```

Two orchestrators consume that — `InterpretedLanguageOrchestrator` and
`CompiledLanguageOrchestrator`. Compiled bots are built once at upload time into a persisted
artifact rather than recompiled every tick, because a Haskell compile does not fit inside a 200 ms
budget.

Adding a language means: an image, a runtime descriptor, a command builder, an environment builder,
and a `/opt/screeps2/worker` launcher inside the image. It does not mean touching the tick.

## The limits

Canonical, and enforced per execution:

| Limit | Value | Why |
|---|---|---|
| CPU allocation per tick | 150 ms | The per-player budget; unused time accrues into a bucket |
| Hard kill | 200 ms | The ceiling the soft timeout can never exceed |
| Real-time overhead allowance | 50 ms | Container round-trip is not the player's CPU |
| Memory | 256 MiB | Per execution |
| Max output | 1 MiB | Truncated, not buffered to death |
| CPU bucket cap | 10,000 ms | Burst allowance for pathfinding and planning spikes |
| Max open files | 65,536 | |
| Max processes (per-process `nproc`) | 65,535 for most images; 512 for TypeScript | Deliberately high — see below |
| Processes per container (`PidsLimit`) | 512 | The cap that actually holds |

The soft timeout is derived, not configured: `min(bucketCurrentCpu, hardKillMs)`. A player who has
banked CPU gets more of it; nobody gets past the hard kill.

The CPU bucket is Screeps' idea and it is a good one. Unused CPU accumulates, so a bot that idles
for ten ticks can afford one expensive planning tick. It is a soft roof that nudges players toward
better code, not a punishment.

### The `MaxProcesses` number is a scar

65,535 looks like someone gave up on limiting processes. What actually happened: the TypeScript
image runs `tsc` inside the container, `tsc` forks aggressively, and a sane process cap made it die
with `vfork: Resource temporarily unavailable` — intermittently, under load, which is the worst way
to find out. Limits are per-language overridable for exactly this reason, and TypeScript carries its
own raised process and file-handle values.

The per-process number is not what contains a fork bomb. The container's own PID limit — 512 — caps
every process in it, and the memory ceiling and the hard kill hold regardless. The limit that looks
alarming is the one that does not matter, and the one that matters is a line further down.

I left the number where it is and wrote down why. That is worth more than a tidier-looking config.

## When it goes wrong

A failed execution is classified before anything is retried, so the recovery action is named rather
than inferred at the call site:

```csharp
// src/Screeps2.Infrastructure/Sandbox/Docker/Execution/ContainerFaultClassifier.cs
public static ContainerFaultDecision Classify(ExecutionOrchestrationResult result)
{
    var errorMessage = result.ErrorMessage ?? result.Exception?.Message ?? string.Empty;

    var isContainerNotRunning = result.Exception is InvalidOperationException invalidOpEx &&
        invalidOpEx.Message.Contains("is not running", StringComparison.OrdinalIgnoreCase);

    // vfork/resource exhaustion during compile or exec: recycle so the next tick gets a
    // fresh process table.
    var isResourceExhaustion =
        errorMessage.Contains("Resource temporarily unavailable", StringComparison.OrdinalIgnoreCase) ||
        errorMessage.Contains("vfork", StringComparison.OrdinalIgnoreCase);

    return new ContainerFaultDecision(
        ShouldRecycle: isContainerNotRunning || isResourceExhaustion,
        IsResourceExhaustion: isResourceExhaustion);
}
```

String-matching a Docker error message is not elegant. It is what the platform gives you, and the
honest move is to isolate it in one classifier with unit tests over the real message shapes rather
than scatter `Contains("vfork")` through the executor. The return type —
`readonly record struct ContainerFaultDecision(bool ShouldRecycle, bool IsResourceExhaustion)` —
keeps the two independent facts explicit instead of encoding one in a bool and the other in a
comment.

Above that sits the escalation ladder:

- **Timeout** — the late response is discarded, the channel torn down, the worker marked unhealthy.
  The tick does not wait.
- **Crash** — exit code mapped to a player-facing error, written to that player's console, tick
  continues.
- **Repeated failure** — a sliding window over timeout count and failure rate flips the script to
  `DisabledDueToInstability`. The player is told. The engine stops paying for it.
- **Container lost out of band** — detected and recreated on the next execution.

That last rung exists because of a run that died. Around tick 300 both worker containers vanished
underneath the engine, every skipped tick was counted as the player's runtime error, and both scripts
crossed the instability threshold — 52% failures — and were disabled for good. A Docker blip had been
charged to the players. Skips caused by a missing channel now record their own status,
`InfrastructureUnavailable`, which the auto-disable window does not count as a failure. Code that
starts and then errors or times out is still the player's.

## Why not Judge0 or Piston

Both exist, both are good, and I read them closely — the container lifecycle and resource-limit
handling here borrow their patterns, and the code says so where it does. What they are built for is a
stateless job: submit code, get stdout. This is a function the engine calls once per tick against a
frozen snapshot, whose output is intents merged into a deterministic simulation, where one player's
timeout must never hold up the others. That is a different shape of problem, so it got its own
executor rather than an adapter around someone else's.

The sandbox's security hygiene — tokens, auth schemes, what the server is allowed to send — is in
[The trust boundary](09-trust-boundary.md).

---

**Next:** [The persistent-worker protocol →](03-persistent-worker-protocol.md)
