# RimWorld Pawn Generation and Factions

Load this companion after the
[RimWorld modding overview](rimworld-modding.overview.knack.md) when code
generates pawns, defines a pawn kind or a new race, places non-player pawns
that must stay calm and harmless, or relies on faction relations. Think
trees, lords, and fleeing are in
[Pawns and AI](rimworld-modding.api.pawns-and-ai.knack.md).

Source review: 2026-10-13, against RimWorld 1.6.4871. Evidence marks:
**[decomp]** means verified only by decompiling `Assembly-CSharp.dll` from
1.6.4871; **[observed]** means seen in game testing on 1.6.4871 but not
traced to a published source.

## Generating Pawns

`PawnGenerator.GeneratePawn(new PawnGenerationRequest(...))` takes a long
list of optional, named arguments. Useful ones for scripted pawns, all
present in 1.6.4871 [decomp]:

```csharp
var request = new PawnGenerationRequest(
    BW_DefOf.BW_TownResident, faction,
    forceGenerateNewPawn: true,       // never reuse a world pawn
    canGeneratePawnRelations: false,  // no surprise relatives
    allowAddictions: false,
    forbidAnyTitle: true,             // no royal titles
    forcedXenotype: ModsConfig.BiotechActive
        ? XenotypeDefOf.Baseliner : null,
    developmentalStages: DevelopmentalStage.Adult,
    dontGiveWeapon: true);
Pawn pawn = PawnGenerator.GeneratePawn(request);
pawn.Name = new NameTriple("Ada", "Gauge", "Pressley");
```

- `NameTriple` trims its first and last names, so a null either way throws
  [decomp]. The map label shows the nickname. Only the player's rename
  dialog limits length (first 12, nickname 16, last 12 characters)
  [decomp]; keep generated names within those limits so the player can
  edit them.
- `XenotypeDefOf.Baseliner` is a Biotech field; guard it as above.

## Pawn Kinds

A custom `PawnKindDef` avoids inheriting a faction's quirks. Its
configuration rules [decomp]:

- set `combatPower`; the default of -1 is a config error;
- a humanlike kind needs `initialWillRange` and `initialResistanceRange`;
- a faction's apparel stuff filter applies unless
  `ignoreFactionApparelStuffRequirements` is set; the hidden Ancients
  faction, for one, allows only a few rare fabrics and plasteel;
- `backstoryFiltersOverride` replaces the faction's backstory categories;
- `fleeHealthThresholdRange` (default -0.4 to 0.4) controls panic on damage
  (below).

A `[DefOf]` field whose name matches a def of another type needs
`[DefAlias("...")]` to bind to the right one.

## Factions

- `Faction.OfAncients` is the hidden, neutral faction every world has
  (`requiredCountAtGameStart` 1, `hidden`, `autoFlee` false in Core's
  `Factions_Hidden.xml`). `AncientsHostile` is its permanently hostile twin.
- A hidden faction has no goodwill (`HasGoodwill` is false), so harming,
  killing, or stripping its members changes no relation and sends no
  goodwill message [decomp]. That suits test fixtures, set dressing, and
  scripted townsfolk.
- Death letters are sent only for pawns of the player's faction or hosted
  by it [decomp].
- Ancients have `rescueesCanJoin`, so a rescued member may join the
  colony. Code that later removes "its" pawns must check the faction at
  removal time and leave prisoners, slaves, and recruits alone [observed].
- Raiders are hostile to the Ancients and will attack them.

## Needs, Moods, and Breaks

- Needs marked `colonistsOnly` are removed from non-player pawns, and
  `colonistAndPrisonersOnly` ones from anyone neither the player's nor its
  prisoner [decomp]. A visitor keeps food, rest, and mood, but no joy,
  beauty, or comfort.
- Mental breaks are not limited to colonists: random breaks depend on the
  life stage, not the faction, so any spawned adult humanlike can break
  [decomp], and a berserk resident attacks whoever is near.
- A non-player humanlike hit while its health is below its kind's flee
  threshold enters the `PanicFlee` mental state [decomp], whose think node
  exits the map (`SubTrees_Misc.xml`).
- To keep scripted pawns calm, give them a hediff whose stage sets
  `blocksMentalBreaks`, `pctAllThoughtNullification` 1, a fixed
  `overrideMoodBase`, and, if they must not need feeding,
  `hungerRateFactor` 0. Blocked breaks also block that panic flee, as do
  `mindState.mentalStateHandler.neverFleeIndividual` and a flee threshold of
  -1 to -1 [decomp].

## Custom Races

- `RaceProperties.IsMechanoid` compares the race's flesh type with
  vanilla's `Mechanoid` [decomp]. A robot race with its own `FleshTypeDef`
  is therefore not a mechanoid, and Biotech's mechanitor, bandwidth, and
  overseer rules never apply to it. A race that does use mechanoid flesh
  must set butcher products, or it raises a config error [decomp].
- `ToolUser` intelligence, as vanilla's mechanoids use, filters most needs
  out by intelligence, food type, or rest.
- Every body needs at least one part with `frostbiteVulnerability` above
  zero; otherwise "no parts vulnerable to frostbite" is a config error
  [decomp].
- Damage to a part overflows into its parent. Damage to a wheel large
  enough to remove it can destroy the chassis and kill the pawn outright
  [observed]. To test the loss of a part, add a missing-part hediff rather
  than dealing damage.
- To take a live pawn off the map for good, as a pawn becoming a building
  does, despawn and discard it the way vanilla's dryad returning to its
  tree does; `Pawn.Destroy` accepts only `Vanish` and `KillFinalize`
  [decomp].
- Vanilla offers the allowed-area menu only to colonists, animals, and
  mechanoids, although every player pawn has `playerSettings`. A custom
  player pawn needs its own gizmo, which can call vanilla's area menu
  (`AreaUtility.MakeAllowedAreaListFloatMenu`) for the whole selection
  [observed].
- `Pawn.IsColonist` means a humanlike of the player's faction, including
  secure slaves, that is not subhuman [decomp]. A robot of the player's
  faction is not a colonist.

## Review Checklist

- Does every generation request pin what matters (new pawn, no relations,
  no titles, life stage, xenotype under a DLC guard)?
- Does each custom pawn kind pass the config rules above?
- Is the faction chosen for its goodwill, letter, and recruitment
  behavior?
- Are scripted pawns protected from breaks, panic, hunger, and leaving?
- Does a custom race avoid mechanoid rules deliberately, and does its body
  pass the frostbite rule?

## Further Information

- RimWorld Wiki, [Pawn Generation Process](https://rimworldwiki.com/wiki/Modding_Tutorials/Pawn_Generation_Process)
  and [Xenotype template](https://rimworldwiki.com/wiki/Modding_Tutorials/Xenotype_template)
- Ludeon's partial source in the install:
  `Source/RimWorld/Defs/DefTypes/FactionDef.cs`
- Vanilla defs: `Data/Core/Defs/FactionDefs/Factions_Hidden.xml`,
  `Data/Core/Defs/PawnKinds/`, `Data/Core/Defs/Bodies/`, and the mechanoid
  races in `Data/Core/Defs/ThingDefs_Races/`
- Ludeon Studios, "Modder Primer for 1.6" (mutants and `IsSubhuman`):
  <https://docs.google.com/document/d/e/2PACX-1vRKE9u5ZW_zG45pxzwNvy4sxvozDeqtxlxpac5jwenOeW6liQCPgmPl9bIbtcMuqL1NPIDHOLFg64M_/pub>
