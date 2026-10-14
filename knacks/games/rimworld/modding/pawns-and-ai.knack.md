# RimWorld Pawns and AI

Load this companion after the
[RimWorld modding overview](overview.knack.md) when pawns
must react to something new, follow a custom routine, stay in an area, or
flee. Generating pawns, factions, needs, and custom races are in
[Pawn Generation and Factions](pawn-generation-and-factions.knack.md).

Reviewed: 2026-10-14, against RimWorld 1.6.4871. Evidence marks:
**[decomp]** means verified only by decompiling `Assembly-CSharp.dll` from
1.6.4871; **[observed]** means seen in game testing on 1.6.4871 but not
traced to a published source.

## Mental Model

A pawn does one *job* at a time, run by a `JobDriver` as a sequence of
toils. Jobs come from *think trees*: trees of `ThinkNode`s whose leaves are
`ThinkNode_JobGiver`s returning a job or null. Each race names two trees in
its `RaceProperties`:

- the **main tree** (`thinkTreeMain`), consulted when the current job ends;
- the **constant tree** (`thinkTreeConstant`), consulted about every 30
  ticks; a job it returns interrupts the current one [decomp].

A non-player pawn also takes orders from its *lord*, through a *duty* that
the lord's current toil assigns and that the main tree consults.

## Main or Constant Tree

The main tree is the place for ordinary choices. It is too slow for
interruptions: a colonist hauling for ten seconds asks it nothing until the
haul ends. Reactions that must happen within a second or two, such as
fleeing a countdown, therefore belong in the constant tree.

Vanilla's `HumanlikeConstant`, `AnimalConstant`, and `InsectConstant` trees
each contain a gate node, `ThinkNode_ConditionalCanDoConstantThinkTreeJobNow`.
It passes only pawns that are awake, not drafted, not burning, not in a
mental state, and not downed unless able to crawl [decomp]. Prepend your
giver inside that gate's `subNodes` with an XML patch (see
[Defs and XML](defs-and-xml.knack.md)), and it inherits
all of those exclusions.

Do not copy an urgent giver into the main tree as well. Prepended there,
it would sit above the node vanilla's `Humanlike.xml` labels "Wait if
drafted" and send drafted colonists running whenever a job ended.

Know which trees "everyone" runs. Humanlikes, animals (dryads included),
and insects use the three trees above. Mechanoids, Odyssey's drones,
Biotech's babies, and Anomaly's entities have their own constant trees
(see `Data/*/Defs/ThinkTreeDefs`). A test that walks every race in Core and
the DLCs and checks which constant tree each uses keeps the claim honest.

The 30-tick cadence quantizes timing. A reaction meant to come two seconds
after first sight comes 2 to 2.5 seconds after it [observed].

## Job Givers

```csharp
public class JobGiver_FleeSteamLeak : ThinkNode_JobGiver
{
    protected override Job TryGiveJob(Pawn pawn)
    {
        Building_SteamPress leak = SteamLeaks.NearestVisibleTo(pawn);
        if (leak == null)
        {
            return null;
        }
        // Keep the current flee instead of replacing it every 30 ticks.
        if (pawn.CurJobDef == BW_DefOf.BW_FleeSteamLeak)
        {
            return pawn.CurJob;
        }
        return JobMaker.MakeJob(BW_DefOf.BW_FleeSteamLeak, FleeCell(pawn));
    }
}
```

- Returning the pawn's current job from a constant-tree giver keeps lower
  constant-tree nodes, such as vanilla's hostility response, from
  interrupting it [observed].
- A giver called every half second must be cheap. Prefer a bounded flood
  fill over path searches.
- If a reaction is random, decide once per pawn per threat and store the
  result on the threat (keyed by `thingIDNumber`, saved with it), or hash
  the two IDs. Rolling inside the giver re-rolls on every think
  [observed].
- In a `JobDriver`, a toil's finish actions run whenever the toil ends,
  including when the job is interrupted [decomp]. An effect that should
  happen only on success, such as arming a device, belongs in the next
  toil's `initAction`, not in `AddFinishAction`.
- Job reports are lowercase and read after the pawn's name, ending with a
  full stop ("fleeing a steam leak."). Set `alwaysShowReport` on the
  `JobDef` if other factions' pawns should show it.

## Custom Think Trees

A pawn kind of your own can name its own trees. Start the main tree with
vanilla's `Downed` subtree (a `ThinkNode_Subtree` whose `treeDef` is
`Downed`), as the Core, Ideology, and Anomaly main trees do. Without it a
downed pawn keeps receiving jobs, and the path follower logs "tried to path
while downed. This should never happen." [observed]

## Lords and Duties

A non-colonist humanlike with no lord eventually leaves: the last fallback
of vanilla's `Humanlike` main tree is `JobGiver_ExitMapBest` for anyone who
is not a colonist. To keep visitors, residents, or guards on the map, give
them a lord.

```csharp
public class LordJob_TownResident : LordJob
{
    private IntVec3 home;

    public LordJob_TownResident() { } // required for loading

    public LordJob_TownResident(IntVec3 home) => this.home = home;

    public override bool AddFleeToil => false;

    public override StateGraph CreateGraph()
    {
        var graph = new StateGraph();
        graph.AddToil(new LordToil_TownResident(home));
        return graph;
    }

    public override bool ShouldRemovePawn(Pawn p, PawnLostCondition reason)
        => reason != PawnLostCondition.Incapped;

    public override void ExposeData() =>
        Scribe_Values.Look(ref home, "home");
}
```

Create it with `LordMaker.MakeNewLord(faction, lordJob, map, pawns)`. The
rules behind the example [decomp]:

- `ShouldRemovePawn` defaults to true, and being downed reports
  `PawnLostCondition.Incapped`, so a downed member leaves its lord and
  later walks off the map. Keep incapacitated members.
- `AddFleeToil` defaults to true; for a non-player faction with
  `autoFlee`, the lord then gains vanilla's panic-flee toil.
- A lord job is saved deep, so it needs a parameterless constructor and
  must save its own fields.

A `LordToil` assigns duties in `UpdateAllDuties`:
`pawn.mindState.duty = new PawnDuty(dutyDef, focus) { wanderRadius = 8f }`.
A `DutyDef` carries its own `thinkNode`. Choose the duty for what it does:

- vanilla's `Defend` duty fights anything within 65 tiles
  (`JobGiver_AIDefendPoint`), even bare-handed;
- a wander radius above 12 logs "wanderRadius ... greater than
  Region.GridSize of 12 and will break" [decomp]; use one lord per area
  rather than one large radius;
- `ThinkNode_ForbidOutsideFlagRadius` treats every cell beyond its radius
  from the duty location as forbidden, **food included** [decomp], so a
  leashed pawn whose nearest meal lies just outside can starve; a review
  caught one such case. Stop hunger (a hediff stage with
  `hungerRateFactor` 0) or keep food inside the leash;
- `JobGiver_SeekSafeTemperature` accepts `maxRadius` but its region search
  ignores it [decomp], so a leashed pawn with heatstroke can walk across the
  map. Subclass it to search only within the leash.

## Fleeing

- Vanilla's `JobGiver_FleePotentialExplosion` reacts only to
  `pawn.mindState.knownExploder`, flees only 9 tiles, and runs only for
  humanlike intelligence [decomp]. Only
  `GenExplosion.NotifyNearbyPawnsOfDangerousExplosive` sets that field,
  which lit `CompExplosive`s and delayed explosive projectiles call
  [decomp]. A custom hazard must call it, accepting those limits, or
  provide its own giver.
- `CellFinderLoose.GetFleeDest` and `RCellFinder.TryFindDirectFleeDestination`
  do not guarantee a minimum distance, and the latter can run several path
  searches per call [observed]. For a guaranteed distance, flood-fill
  outward from the pawn over cells it can cross and accept the first cell
  beyond the distance that `CellFinderLoose.CanFleeToLocation` accepts.
- For an ordinary "run from this thing" job, vanilla's
  `FleeUtility.FleeJob(pawn, danger, fleeDistance)` builds one. The
  `PanicFlee` mental state is different: it sends the pawn off the map.
- "Can see" is usually `GenSight.LineOfSightToThing`; a threat that matters
  usually passes `!pawn.ThreatDisabled(null)`, which excludes downed,
  dormant, and invisible pawns.
- A pawn already fleeing should keep fleeing even when the threat leaves
  its sight; re-deciding every think makes pawns oscillate.

## Review Checklist

- Is each urgent reaction in a constant tree, inside its gate?
- Are drafted, downed, burning, and mentally broken pawns excluded?
- Does each custom main tree start with the `Downed` subtree?
- Do lord jobs keep incapacitated members, opt out of the flee toil when
  appropriate, and load?
- Is every leash radius 12 or less, with food and temperature handled
  inside it?
- Are random reactions decided once and saved?

## Further Information

- Vanilla think trees: `Data/Core/Defs/ThinkTreeDefs/Humanlike.xml`,
  `Animal.xml`, `Insect.xml`, and `SubTrees_Misc.xml`; duties in
  `Data/Core/Defs/DutyDefs/`
- Ludeon's partial source in the install: `Source/Verse/AI/JobDrivers/`
  (`JobDriver_Goto.cs`, `JobDriver_Wait.cs`, `Toils_General.cs`, and
  `ToilFailConditions.cs`) and `Source/Verse/Defs/DefTypes/JobDef.cs`
- RimWorld Wiki, [Writing custom code](https://rimworldwiki.com/wiki/Modding_Tutorials/Writing_custom_code)
  and [Code: Mending Job](https://rimworldwiki.com/wiki/Modding_Tutorials/Code_MendingJob)
- Ludeon Studios, "Modder Primer for 1.6" (asynchronous path requests;
  cached lords):
  <https://docs.google.com/document/d/e/2PACX-1vRKE9u5ZW_zG45pxzwNvy4sxvozDeqtxlxpac5jwenOeW6liQCPgmPl9bIbtcMuqL1NPIDHOLFg64M_/pub>
