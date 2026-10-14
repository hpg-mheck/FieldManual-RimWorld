# RimWorld Lifecycle and Ticking

Load this companion after the
[RimWorld modding overview](overview.knack.md) when writing
a `Thing`, `ThingComp`, `MapComponent`, or `GameComponent` subclass, saving
state, or holding textures and materials in C#.

Reviewed: 2026-10-14, against RimWorld 1.6.4871. Evidence marks:
**[decomp]** means verified only by decompiling `Assembly-CSharp.dll` from
1.6.4871; **[observed]** means seen in game testing on 1.6.4871 but not
traced to a published source.

## Ticks and Ticker Types

The game advances in ticks, 60 per second at normal speed. A `ThingDef`'s
`tickerType` decides which method the engine calls:

| `tickerType` | Method called | Cadence |
|---|---|---|
| `Never` | none | |
| `Normal` | `Tick()`, and `TickInterval(int delta)` | every tick; see below |
| `Rare` | `TickRare()` | every 250 ticks |
| `Long` | `TickLong()` | every 2,000 ticks |

Comps follow their parent: `CompTick`, `CompTickInterval(delta)`,
`CompTickRare`, and `CompTickLong`. A comp ticks only if its parent's ticker
type calls the matching method.

### Variable Tick Rate (1.6)

1.6 added `TickInterval(int delta)` to things, comps, and jobs; `delta` is
the number of ticks since the last call. Its interval follows the camera
[decomp]: a thing in view runs it every tick at the closest zoom and up to
every five ticks zoomed out, and a thing out of view or on another map
every 15 ticks. Animals always use 15. Scale time-based logic by
`delta`, and override `MinTickIntervalRate`, `MaxTickIntervalRate` (15 by
default), or `UpdateRateTicks` when a thing needs a different rate.
`Tick()` still runs every tick for work that cannot be batched, such as
path following.

`Tick()` is `protected` in 1.6, so an override written for an older
version may fail to compile [decomp].

## Things That Throw or Keep Ticking

**`Entity.TickRare()` and `Entity.TickLong()` throw
`NotImplementedException`** [decomp]. `ThingWithComps` overrides both to
tick its comps, so `base.TickRare()` is safe there. A direct subclass of
`Thing` with `tickerType` `Rare` must override `TickRare()` and must not
call the base.

**Comps keep ticking after a mid-loop kill.** `ThingWithComps.Tick()` walks
its comps by index without checking `Destroyed` between them [decomp]. If
one comp destroys its parent, the comps after it in the def's `<comps>`
list still tick on a destroyed thing. Put a destructive comp last, or check
`parent.Destroyed` at the top of each comp tick.

**Held things tick.** In 1.6, a ticking `IThingHolder` ticks its contents
by default. A minified building ticks inside its crate (the vanilla
`MinifiedThing` def uses `tickerType` `Normal`), and so does anything in a
container. Its `Map` is null there, so a tick that touches the map throws
every time it runs; uninstalling such a building logged an exception every
250 ticks [observed]. Guard every tick:

```csharp
public override void CompTickRare()
{
    if (!parent.Spawned)
    {
        return; // packed in a crate or held in a container
    }
    // ...
}
```

A holder can stop ticking its contents by implementing
`IThingHolderTickable` and returning false from `ShouldTickContents`, or
by implementing `ISuspendableThingHolder` with `IsContentsSuspended` true,
which stops the contents ticking at all, as a cryptosleep casket does: food
inside no longer rots [decomp].

## Spawning, Damage, and Destruction

- `PostMake()` runs once, at creation. `SpawnSetup(map,
  respawningAfterLoad)` runs on every spawn, including after a load; set up
  caches there and use the flag to skip first-spawn effects.
- `DeSpawn(mode)` and `Destroy(mode)` run on removal. `Kill()` destroys
  with `DestroyMode.KillFinalize`, the mode that drops leavings and lets
  comps react to a death; vanilla's `CompExplosive` detonates on it when
  `explodeOnKilled` is set (see the shipped `CompExplosive.cs`).
  `DestroyMode.Vanish` removes a thing without those effects, which is what
  test teardown and ground clearing want.
- `Pawn.Destroy()` accepts only `Vanish` and `KillFinalize`, and logs an
  error for any other mode [decomp].
- `PreApplyDamage(ref DamageInfo dinfo, out bool absorbed)` can absorb a
  hit; `PostApplyDamage` reacts after it. Both run inside another thing's
  damage pass, such as an explosion's cell loop, so record what should
  happen and act on your own next tick rather than destroying things
  there.

## Components

`GameComponent`, `MapComponent`, and `WorldComponent` subclasses are found
by reflection and created for every game, map, or world. Each needs a
public constructor taking `Game`, `Map`, or `World` respectively; the game
creates them with `Activator.CreateInstance` and logs an error if that
fails [decomp]. Every map gets one of every `MapComponent` type, except
subclasses of `CustomMapComponent`, which are added only on request
[decomp]. An idle component must therefore cost nothing.

Run a long effect that should outlive the thing that started it (an
aftermath of fires and secondary blasts, say) in a `MapComponent` that is
saved with the map. It then survives the source's destruction and a save
made halfway through.

Per-frame hooks (`GameComponentUpdate`, `MapComponentUpdate`) run every
frame, paused or not; per-tick hooks (`GameComponentTick`,
`MapComponentTick`) run only as the game advances [decomp].

## Static Assets

Unity assets must load on the main thread. Keep every static field of type
`Texture`, `Material`, `Shader`, `Graphic`, `GameObject`, or
`MaterialPropertyBlock` in a class marked `[StaticConstructorOnStartup]`;
the game runs those static constructors after defs load. In dev mode it
warns for any other class holding one: "Type ... probably needs a
StaticConstructorOnStartup attribute" [decomp].

```csharp
[StaticConstructorOnStartup]
public static class BW_Textures
{
    public static readonly Texture2D VentIcon =
        ContentFinder<Texture2D>.Get("UI/Commands/BW_Vent");
}
```

`ContentFinder.Get` logs an error for a missing path unless called with
`reportFailure: false`.

## Saving and Loading

State survives a save only if `ExposeData()` writes it. The `Scribe`
helpers each run in several passes (`LoadSaveMode.Saving`, `LoadingVars`,
`ResolvingCrossRefs`, and `PostLoadInit`):

- `Scribe_Values.Look(ref x, "x", defaultValue)` for primitives and simple
  structs. A value equal to the default is not written, and a missing node
  loads the default, so pass the same default as the field initializer
  [decomp]. Defs need `Scribe_Defs`; a def through `Scribe_Values` is an
  error [decomp].
- `Scribe_Deep.Look(ref obj, "obj")` for objects this one owns. The type
  needs a parameterless constructor (it may be private) [decomp].
- `Scribe_References.Look(ref thing, "thing")` for things saved elsewhere.
  A reference to a destroyed thing is saved as null unless
  `saveDestroyedThings` is true [decomp].
- `Scribe_Collections.Look(ref list, "list", LookMode.Deep)` and the other
  `LookMode`s for lists and dictionaries.

Do fix-ups in `PostLoadInit`: prune null entries, rebuild caches, and give
fields added since an older save a sane value.

```csharp
public override void ExposeData()
{
    base.ExposeData();
    Scribe_Values.Look(ref pressure, "pressure", 0f);
    Scribe_References.Look(ref operatorPawn, "operatorPawn");
    Scribe_Collections.Look(ref vents, "vents", LookMode.Deep);
    if (Scribe.mode == LoadSaveMode.PostLoadInit)
    {
        vents ??= new List<Vent>();
        vents.RemoveAll(v => v == null);
    }
}
```

## Randomness and Stable Choices

- `Rand.Value` is in [0, 1] *inclusive*; it can return exactly 1
  [decomp]. Clamp before indexing a weighted table that rejects 1.
- `Rand.Range(int, int)` excludes its maximum; `Rand.RangeInclusive` and
  `IntRange.RandomInRange` include it [decomp].
- For a choice that must stay the same across saves and restarted jobs
  without being saved, hash stable IDs:
  `Gen.HashCombineInt(pawn.thingIDNumber, thing.thingIDNumber)`. Tests can
  then predict the outcome.

## Review Checklist

- Does each tick override match the def's `tickerType`, and does a
  `Rare` or `Long` `Thing` subclass avoid calling the base?
- Does every tick return early when the thing is not spawned?
- Is any destructive comp last in `<comps>`, or does each later comp check
  `Destroyed`?
- Is damage handling deferred to the thing's own tick?
- Do components have the right constructor and cost nothing while idle?
- Are all static Unity assets in `[StaticConstructorOnStartup]` classes?
- Is every field that matters saved, with matching defaults and
  `PostLoadInit` repair?

## Further Information

- Ludeon Studios, "Modder Primer for 1.6" (variable tick rate; holders
  ticking their contents):
  <https://docs.google.com/document/d/e/2PACX-1vRKE9u5ZW_zG45pxzwNvy4sxvozDeqtxlxpac5jwenOeW6liQCPgmPl9bIbtcMuqL1NPIDHOLFg64M_/pub>
- RimWorld Wiki, [ThingComp](https://rimworldwiki.com/wiki/Modding_Tutorials/ThingComp),
  [GameComponent](https://rimworldwiki.com/wiki/Modding_Tutorials/GameComponent),
  and [ExposeData](https://rimworldwiki.com/wiki/Modding_Tutorials/ExposeData)
- RimWorld Wiki, [Application Startup](https://rimworldwiki.com/wiki/Modding_Tutorials/Application_Startup)
- Ludeon's partial source in the install: `Source/RimWorld/ThingComps/`
  (for example `CompExplosive.cs`) and
  `Source/Verse/Thing/Building/Building.cs`
- The vanilla `MinifiedThing` def in
  `Data/Core/Defs/ThingDefs_Items/Items_Unfinished.xml`
