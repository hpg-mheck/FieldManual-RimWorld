# RimWorld Smoke Runner

Load this companion after the
[RimWorld testing overview](rimworld-testing.overview.knack.md) when
building or extending an in-game test runner: code inside the mod that, when
the game starts with a flag, builds scenarios on a quick-test map, drives
real ticks, checks results, and logs a verdict. The general design (the
log protocol, simulated time, scenario hygiene, error attribution) is in
FieldManual
`knacks/software-engineering/testing/in-host-smoke.knack/`; this
companion gives RimWorld's version of each part.

Reviewed: 2026-10-14, against RimWorld 1.6.4871. Evidence marks:
**[decomp]** means verified only by decompiling `Assembly-CSharp.dll` from
1.6.4871; **[observed]** means seen in game testing on 1.6.4871 but not
traced to a published source.

## Shape

A `GameComponent` is created for every game, the quick-test game
included, and costs nothing until a command-line flag switches it on:

```csharp
public class GameComponent_BWSmoke : GameComponent
{
    private const string Flag = "bwsmoketest";
    private const string KeepOpenFlag = "bwsmoketestkeep";
    private bool done;
    private int settledFrames;
    private int passed, failed;

    public GameComponent_BWSmoke(Game game) { }

    public override void GameComponentUpdate()
    {
        if (done || Current.ProgramState != ProgramState.Playing
            || Find.CurrentMap == null
            || !GenCommandLine.CommandLineArgPassed(Flag)
            || ++settledFrames < 120)
        {
            return;
        }
        done = true;
        try { Run(Find.CurrentMap); }
        catch (Exception e) { Check("runner finished", false, e.ToString()); }
        Log.Message($"[BW-SMOKE] DONE passed={passed} failed={failed}");
        if (!GenCommandLine.CommandLineArgPassed(KeepOpenFlag))
        {
            Root.Shutdown();
        }
    }

    private static void Tick(int ticks)
    {
        for (int i = 0; i < ticks; i++)
        {
            Find.TickManager.DoSingleTick();
        }
    }

    private void Check(string name, bool ok, string detail = null)
    {
        if (ok) { passed++; Log.Message($"[BW-SMOKE] PASS {name}"); }
        else { failed++; Log.Warning($"[BW-SMOKE] FAIL {name}: {detail}"); }
    }
}
```

- `GenCommandLine.CommandLineArgPassed` ignores case and accepts the flag
  with or without a leading dash [decomp].
- Wait a couple of seconds of frames after the map appears, so map
  generation's own follow-up work has settled.
- Catch everything around `Run`, so a crash becomes a failed check and the
  DONE line still appears for the harness to read.

## Synchronous Ticking

`Find.TickManager.DoSingleTick()` in a loop advances the game inside a
single frame. It is fast and keeps scenarios in a fixed order, but work
the engine does once per *frame* does not happen [decomp]:

- power nets are rewired in `Map.MapUpdate`; call
  `map.powerNetManager.UpdatePowerNetsAndConnections_First()` before
  reading `PowerOn`, and give a net time to switch waiting consumers on,
  which it does a few at a time (see
  [Buildings and Storage](../rimworld-modding.knack/rimworld-modding.api.buildings-and-storage.knack.md));
- rooms rebuild lazily; call
  `map.regionAndRoomUpdater.TryRebuildDirtyRegionsAndRooms()` before
  reading a room in the same frame;
- nothing is drawn while the loop runs, so the window does not respond
  until the run ends; the harness must wait on the log, not the window.

Heavy effects slow ticking far below real time: hundreds of explosive
projectiles took seconds of wall-clock time to resolve [observed].

## Reading Errors

`Log.Messages` exposes the game's own log queue: each `LogMessage` has a
`type`, its `text`, and a `repeats` count. Mark the newest error before a
scenario and compare afterwards:

```csharp
private static (LogMessage Last, int Repeats) ErrorMark()
{
    LogMessage last = Log.Messages
        .LastOrDefault(m => m.type == LogMessageType.Error);
    return (last, last?.repeats ?? 0);
}

private static bool NewErrorSince((LogMessage Last, int Repeats) mark)
{
    var now = ErrorMark();
    return now.Last != mark.Last || now.Repeats != mark.Repeats;
}
```

Queue limits [decomp]: it keeps the last 1,000 messages; an identical
message logged consecutively increments `repeats` (up to 99) instead of
adding an entry, hence the repeat count in the mark; and logging stops
entirely after 10,000 messages. A runner that logs a great deal can push
early errors out of the queue, so check marks per scenario and scan the
whole queue at the end as well. The final scan should fail on any error
and on warnings that name the mod or contain "StaticConstructorOnStartup".

Errors also have side effects. With `pauseOnError` set the game pauses on
an error, and in dev mode the log window opens [decomp]; the launch
harness turns the first off.

## Writing Checks

- Name checks as stable sentences ("the press vents to zero"), and make
  each failure detail say what was observed, as the stock runner
  companion describes. In RimWorld that means positions, counts, states,
  the job a pawn was doing, and who reserved what.
- Log NOTE lines in passing runs too (distances reached, seeds, counts).
  Comparing notes across thirty saved runs found a creature that reached
  safety and then wandered back into danger [observed].
- Judge an effect only inside its own window. An explosion's wave arrives
  over several ticks; check the target after it arrives. A pull that
  releases pawns should be judged while it runs, not after a later blast
  moves them.
- Test what was actually built. If construction skipped parts around an
  obstacle, compare with the executed plan, not the requested one
  [observed].
- Count every place a thing can go. An item count that ignores items
  carried by haulers, held in inventories, or fired as ammunition reads
  as a loss [observed].

## Staging Scenarios

- **Order.** Non-destructive scenarios first; anything that explodes
  last. Build shared fixtures once and remove them with
  `DestroyMode.Vanish` before destructive scenarios, so no death effect or
  damage-triggered behavior fires mid-test.
- **Silence the storyteller.** Clear `Find.Storyteller.storytellerComps`
  and its `incidentQueue` at the start, and note how many hostiles were
  already on the map. A raid once landed mid-run and spent the fixtures'
  ammunition [observed].
- **Quiet the map** before precise checks: destroy every `Fire`, vanish
  hostiles, and draft colonists with fire-at-will off, so nobody walks
  through the scenario.
- **Clear the stage** with the same helper the mod uses for clearing
  ground (skip pawns and indestructible things).
- **Fresh pawns.** Generate new pawns for behavior checks: adult, healthy
  (remove generated ailments), fed, and rested, placed at exact cells
  rather than a random cell "near" the target, which can land outside the
  radius under test [observed].
- **Pin randomness.** Narrow a def's random range for the duration of a
  scenario (for example a countdown of 3 to 5 seconds held at 5) and
  restore it in a `finally` block.
- **Real causes.** Fire real projectiles from a pawn rather than calling
  `TakeDamage`; start real jobs with `JobMaker.MakeJob` and
  `pawn.jobs.StartJob`.
- **Lazy state.** Check a roof-collapse buffer before the first tick,
  since the tick resolves it; snapshot roofed cells to compare afterwards
  [observed].

## Seams in the Mod

Give the runner what it needs without reflection:

- diagnostic counters on mod classes (blasts fired, items moved, hits
  absorbed);
- methods that skip waiting, such as "start the countdown now", shared
  with a dev-mode gizmo so they stay exercised;
- deterministic choices by hashing IDs and a stored seed, so the runner
  can predict each actor's outcome. A seeded scenario can then search for
  a seed giving the split it needs (for example at least four of each
  reaction) and assert every actor's result.

## Surviving Lost Fixtures

On a random map a fixture can vanish before its scenario: destroyed by a
stray event, carried off, or never placed. Guard each scenario's inputs:
if one is gone, fail one check that reports how (destroyed, held, hit
points, fires nearby), make a fresh one, and continue. A single
`NullReferenceException` once ended a run after 56 checks, skipping about
70 more [observed].

## Review Checklist

- Is the runner inert without its flag, and does it always log DONE?
- Are per-frame engine steps done by hand where synchronous ticking skips
  them?
- Does every scenario mark errors, and does the run end with a full scan?
- Do failure details and notes say what was observed?
- Are scenarios ordered, isolated, quieted, and staged on fresh fixtures?
- Can the runner survive a missing fixture and finish the run?

## Further Information

- RimWorld Wiki, [GameComponent](https://rimworldwiki.com/wiki/Modding_Tutorials/GameComponent)
  and [Testing mods](https://rimworldwiki.com/wiki/Modding_Tutorials/Testing_mods)
- Ludeon's partial source in the install:
  `Source/Verse/AI/JobDrivers/Basics/` for job driver examples to drive
  from a runner
- [Launch Harness](rimworld-testing.api.launch-harness.knack.md) for the
  script side
- FieldManual
  `knacks/software-engineering/testing/in-host-smoke.knack/`,
  especially its runner and flakiness companions
