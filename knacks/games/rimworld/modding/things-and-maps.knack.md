# RimWorld Things and Maps

Load this companion after the
[RimWorld modding overview](overview.knack.md) when code
creates, moves, or removes things, changes terrain or roofs, or clears and
builds ground from code. Power, ownership, doors, zones, and storage are in
[Buildings and Storage](buildings-and-storage.knack.md).

Reviewed: 2026-10-14, against RimWorld 1.6.4871. Evidence marks:
**[decomp]** means verified only by decompiling `Assembly-CSharp.dll` from
1.6.4871; **[observed]** means seen in game testing on 1.6.4871 but not
traced to a published source.

## Making Things

`ThingMaker.MakeThing(def, stuff)` creates an unspawned thing. It logs an
error and substitutes a default when stuff is missing for a stuff-made def,
or present for one that is not; it does **not** check that the stuff suits
the def [decomp]. A dining chair made from stone blocks comes out invalid.
Check `stuff.stuffProps.CanMake(def)` first, or take
`GenStuff.DefaultStuffFor(def)`.

Some vanilla defs are generated rather than written: carpets, for example,
are a template crossed with color defs (`CarpetMustard`) and have no DefOf
field. Look them up with `DefDatabase<TerrainDef>.GetNamedSilentFail`.

## Spawning

- `GenSpawn.Spawn(thing, cell, map, wipeMode)` places it **facing north**:
  that overload passes `Rot4.North`. Use the overload that takes a `Rot4`.
  A def with `randomizeRotationOnSpawn` ignores the rotation given
  [decomp].
- `GenSpawn` never checks terrain affordance, and it never moves pawns
  out of the way [decomp]. Fix the ground and clear pawns first.
- For loose items, prefer `GenPlace.TryPlaceThing(item, cell, map,
  ThingPlaceMode.Near)`, which finds room and merges stacks.

### Full Cells Shove or Destroy

During play, spawning an item into a cell already holding its maximum
number of items (one on the ground, more on a shelf) despawns one of the
existing items and places it nearby; if there is no room, **that item is
destroyed** [decomp]. Code that moves items one cell at a time therefore
churns: each displaced item lands in the next cell the search tries, which
looks like a line of items marching away, and stock is lost as the area
fills [observed].

Before spawning an item into a cell, check
`cell.GetItemCount(map) < cell.GetMaxItemsAllowedInCell(map)`, or merge
into a matching stack there with `TryAbsorbStack`; otherwise leave the item
where it is. Plan capacity in cells: only nine ground cells lie within 1.5
tiles of a point, eight if a building holds the center.

### Wipe Modes

Spawning a building removes what it displaces. `WipeMode.Vanish` (the
default) destroys it, `FullRefund` refunds it, and `VanishOrMoveAside`
moves items aside first. The rules, from `GenSpawn.SpawningWipes` [decomp]:

- an edifice wipes other edifices, natural rock included;
- an impassable building wipes items unless it has a surface, as a shelf
  or table does; a pass-through one leaves them;
- anything that carries power replaces a conduit in its cell;
- a thing whose def is not `destroyable` is never wiped;
- pawns are never wiped, so a wall spawned on a pawn traps it.

## Footprints and Rotation

A multi-cell thing occupies `GenAdj.OccupiedRect(position, rotation,
def.size)`. The position is the center cell; for even sizes the rectangle
shifts by one cell depending on rotation, and east or west swaps width and
depth [decomp]. Never work the rectangle out by hand. Code that plans
layouts outside the game can mirror the rule in a pure function and test it
against known cases.

## Things That Cannot Be Destroyed

Some vanilla things have `destroyable` false: steam geysers and craters in
Core, and further features in the DLCs. Calling `Destroy()` on one logs
"Tried to destroy non-destroyable thing" as an error [decomp], and in dev
mode any error opens the debug log window over the game. Skip them:

```csharp
foreach (Thing t in cell.GetThingList(map).ToList())
{
    if (t is Pawn || t.Destroyed || !t.def.destroyable)
    {
        continue;
    }
    t.Destroy(DestroyMode.Vanish); // no leavings, no explosions
}
```

`Thing.allowDestroyNonDestroyable` exists for map generation; leave it
alone. Build around such features instead. A structure is blocked only
where a feature overlaps or crowds one of its parts, so measure clearances
from the cells really occupied, never from a bounding rectangle: a geyser
in an empty corner of the rectangle should not reject a site.

## Clearing Ground Safely

Clear only the cells a structure occupies, grown by any margin it needs. A
bounding rectangle can be many times larger than the structure; a review
of one sprawling layout found a rectangle of nearly 10,000 cells that
could have swallowed an unrelated base. Then, in order:

1. unfog the area plus one cell, so breaking into a fogged cave sends no
   "area revealed" letter;
2. move pawns out to the nearest standable cell outside the area;
3. remove roofs before removing rock or walls;
4. destroy every destroyable non-pawn with `DestroyMode.Vanish`;
5. clear any roof collapse already marked inside the area;
6. remove player zone cells from the cleared ground only;
7. fix terrain, then build.

Watch for ancient cryptosleep caskets. `Vanish` discards their occupants,
and damage from an instigator with a faction ejects the casket's whole
group, whose hostile occupants form an assault [decomp].

## Roofs

- `map.roofGrid.SetRoof(cell, def)` sets or clears a roof with no support
  check.
- A roof must be within 6.9 tiles of a roof holder (a wall, door, or
  column), measured through roofed cells.
- Support is rechecked only when a roof holder despawns during play. The
  unsupported cells go into `map.roofCollapseBuffer` and fall during the
  next tick's `MapPreTick`, crushing what is under them [decomp].
- Marks already in the buffer drop *new* roofs too: the next tick drops
  whatever roof stands in a marked cell [decomp]. A builder that roofs
  cells where rock was removed must clear stale marks there first, or the
  fresh roof falls on the first tick. For bulk removal,
  `RoofCollapseCellsFinder.RemoveBulkCollapsingRoofs` handles a set of
  removed holders in one call [decomp].
- Only thick rock roofs leave `CollapsedRocks` behind; a fallen thin or
  constructed roof leaves only rubble filth, so count roofed cells before
  and after instead.
- A room touching the map edge, or with at least a quarter of its cells
  unroofed, uses the outdoor temperature, and heat pushed into it does
  nothing [decomp]. Leave a hot machine's yard open.
- Rooms rebuild lazily. Code that builds walls and reads a room in the
  same frame should first call
  `map.regionAndRoomUpdater.TryRebuildDirtyRegionsAndRooms()`.

## Terrain

- Buildings need a terrain affordance (`Light`, `Medium`, or `Heavy`). For
  stuff-made walls it depends on the stuff: wood needs `Light`, metals
  `Medium`, and stone blocks `Heavy`. Water, marsh, mud, and soft sand
  lack `Heavy`.
- Construction checks such as `GenConstruct.CanBuildOnTerrain` enforce
  affordance; spawning from code does not [decomp].
- Test `terrain.affordances.Contains(TerrainAffordanceDefOf.Heavy)` rather
  than matching names; DLC biomes add terrain.
- `map.terrainGrid.SetTerrain(cell, def)` updates plants, snow, and paths,
  and resets the cell's paint, so paint afterwards with `SetTerrainColor`
  [decomp]. `SetTerrainColor` accepts any terrain and has no DLC check;
  only the player's paint designator requires `isPaintable` [decomp]. A
  neighbor's terrain fades onto a cell when the two differ in def or
  paint, the neighbor is not hard-edged, and the neighbor's render
  precedence is at least the cell's [decomp]. Painted lines stay crisp
  when both colors are painted on a hard-edged floor such as concrete.
- 1.6 adds foundation and temporary terrain layers, and terrain may glow or
  push heat.

## Saving Generated Structures

Code that generates structures from a seed should save the seed and the
obstacles it built around, not the result, and rebuild the plan on load.
After a load the plan must be the one actually built: a plan rebuilt
without its obstacles brings back parts that were dropped around them,
which a review caught in one such generator.

## Review Checklist

- Is stuff checked with `CanMake`, and is rotation passed explicitly?
- Does code that moves items respect cell capacity?
- Does clearing touch only occupied cells, skip pawns and indestructible
  things, and use `Vanish`?
- Are roofs removed before their holders, and stale collapse marks cleared
  before new roofs?
- Is terrain made suitable, and painted after it is set?
- Does a generated structure reload as built?

## Further Information

- RimWorld Wiki, [ThingDef](https://rimworldwiki.com/wiki/Modding_Tutorials/ThingDef)
  and [BigAssListOfUsefulClasses](https://rimworldwiki.com/wiki/Modding_Tutorials/BigAssListOfUsefulClasses)
- Ludeon Studios, "Modder Primer for 1.6" (terrain layers; clear-rect
  helpers in `MapGenUtility`):
  <https://docs.google.com/document/d/e/2PACX-1vRKE9u5ZW_zG45pxzwNvy4sxvozDeqtxlxpac5jwenOeW6liQCPgmPl9bIbtcMuqL1NPIDHOLFg64M_/pub>
- Ludeon's partial source in the install:
  `Source/Verse/Thing/Building/Building.cs` and
  `Source/Verse/Defs/DefTypes/RoofDef.cs`
- Vanilla defs: `Data/Core/Defs/ThingDefs_Buildings/`,
  `Data/Core/Defs/ThingDefs_Items/Items_Resource_Stuff.xml` (wood and
  metal affordances), and `Data/Core/Defs/ThingDefs_Misc/Various_Stone.xml`
  (stone blocks)
