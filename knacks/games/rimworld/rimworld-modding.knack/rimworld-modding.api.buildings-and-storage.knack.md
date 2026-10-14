# RimWorld Buildings and Storage

Load this companion after the
[RimWorld modding overview](rimworld-modding.overview.knack.md) when a
building draws or supplies power, belongs to a faction other than the
player's, stores items, takes bills, or must not attract colonists,
haulers, or raiders. Spawning and terrain are in
[Things and Maps](rimworld-modding.api.things-and-maps.knack.md).

Source review: 2026-10-13, against RimWorld 1.6.4871. Evidence marks:
**[decomp]** means verified only by decompiling `Assembly-CSharp.dll` from
1.6.4871; **[observed]** means seen in game testing on 1.6.4871 but not
traced to a published source.

## Power

- Power nets have **no faction check**: a player's generator powers
  another faction's lamps on the same net [decomp]. Only the gizmos
  (switch, reconnect, color) are limited to the player.
- A consumer connects to the nearest transmitter within its footprint
  expanded by 6 cells [decomp].
- Nets are rebuilt once per **frame** (`Map.MapUpdate`), not per tick.
  Code that spawns buildings and ticks synchronously within one frame must
  call `map.powerNetManager.UpdatePowerNetsAndConnections_First()` before
  reading `PowerOn` [decomp].
- A dynamic draw goes through `CompPowerTrader.PowerOutput` (negative for
  consumption) and must be reapplied regularly, because rewiring calls
  `SetUpPowerVars()`, which restores the def's base value. Vanilla's cooler
  and heater reapply theirs every rare tick [decomp].
- `CompFlickable.SwitchIsOn = false` sets only the actual state. The private
  `wantSwitchOn` stays true, so the toggle still shows "on" and
  `WantsFlick()` reports a pending flick [decomp]. Set both, by reflection
  if necessary.

## Ownership Has Side Effects

Choose each building's faction for what it attracts, not only for whose it
is.

- **Home area.** When auto-home is on, a player-owned building whose def
  keeps `expandHomeArea` (the default) marks the home area over its
  footprint plus 4 cells as it spawns [decomp]. Colonists fight fires,
  repair, and clean only in the home area, so a distant player building
  draws them across the map. Remove such cells from
  `map.areaManager.Home`, or give the building to another faction.
- **Gizmos.** Vanilla comps show their buttons only to the player's
  faction, and so should yours; a device the player must operate has to be
  player-owned.
- **Raids.** Raiders on an assault trash player buildings anywhere on the
  map (`JobGiver_AITrashBuildingsDistant` in the vanilla assault duties).
  A player-owned device that reacts to damage, such as one that explodes,
  becomes a raid target wherever it stands.
- **`SetFaction` versus `SetFactionDirect`.** `SetFactionDirect` sets only
  the field; `SetFaction` also updates listers and the power mesh on a
  spawned building.

## Forbidding, Doors, Fences, and Furniture

Forbidding is faction-asymmetric [decomp]:

- `IsForbidden(thing, faction)` is always false for a non-player faction:
  other factions' pawns eat, carry, and use forbidden things.
- A forbidden door of any faction locks the player's pawns out.
- A door with no faction opens for any race with `canOpenFactionlessDoors`
  (true by default), raiders included; a faction's door opens for pawns of
  any faction not hostile to it, but not for its prisoners or for wild
  animals (shipped source: `Building_Door.PawnCanOpen`; `GenAI.MachinesLike`
  [decomp]).
- Fences and fence gates block roaming animals of any faction (`FenceBlocked`
  pawns); cats and dogs do not roam and walk over fences.

Furniture checks faction unevenly [decomp]: a bed must belong to the
sleeper's faction (or host faction); joy buildings have no faction check,
so colonists walk to another faction's powered television; colonists rescue
only downed pawns of their own faction.

## Stockpiles and Zones

- **Priority beats distance.** Haulers look for a better storage cell in
  priority order (`AllGroupsListInPriorityOrder`) [decomp]. One `Critical`
  stockpile anywhere pulls every item it accepts from the whole map. Give a
  remote building's storage `SetDisallowAll()` and fill it by code if
  haulers must never feed it.
- **Zones and spawned things.** A thing that cannot overlap zones (an
  impassable building, storage, a plant grower) removes the zone cells
  under it, and the zone then keeps only the piece connected to its first
  cell: **cells cut off elsewhere are deleted** [decomp]. When clearing
  ground from code, remove zone cells only inside the cleared area and
  expect split zones to shrink.

## Storage Inside a Building

For a building that holds items haulers may take out, follow vanilla's
bookcase (`Building_Bookcase`):

- implement `IHaulSource` on the **building class**, not on a comp,
  because the game tests `thing is IHaulSource` [decomp];
- contents count as stored only when their holder is also an
  `IHaulDestination` (`StoreUtility` reads `ParentHolder`) [decomp]. Return
  false from `HaulDestinationEnabled` if nothing should be hauled in;
- notify `map.listerHaulables` when items are added, so haulers see them;
- `ThingOwner` accepts any amount; cap `GetCountCanAccept` and `TryAdd`
  yourself where capacity matters;
- like the bookcase, such a building drops its contents when despawned in
  any mode but `WillReplace`, `Vanish` included [decomp]; to remove the
  contents too, destroy them first (`ClearAndDestroyContents`);
- `ISuspendableThingHolder` freezes the contents (see
  [Lifecycle and Ticking](rimworld-modding.api.lifecycle-and-ticking.knack.md)).

## Bills and Autonomous Production

- A recipe with `formingTicks` above zero makes autonomous bills
  (`Bill_Autonomous`), which `Building_WorkTableAutonomous` runs. Both are
  base-game code with no DLC check, though only Biotech's gestator uses
  them in vanilla [decomp].
- Vanilla's gestator still needs a pawn to load it and another to finish
  the bill. A building can finish its own by overriding
  `Notify_FormingCompleted()`, spawning the product, consuming the
  ingredients, and closing the bill's iteration; a bill giver under the
  `Hauling` work type, like vanilla's crematorium, lets a hauler load it
  [observed].
- "Do until you have X" counts through the recipe's
  `workerCounterClass`. Subclass `RecipeWorkerCounter` and override
  `CanCountProducts` and `CountProducts` to count anything, such as units
  that are not items. A recipe with no products takes its icon from
  `uiIconThing`.
- `Building_WorkTable.CurrentlyUsableForBills()` is not virtual and
  refuses bills while any refuelable comp is empty [decomp]. A work table
  with a fuel hopper takes no bills until it is refilled.

## Review Checklist

- Does code that ticks synchronously rewire power before reading it?
- Is each dynamic power draw reapplied after rewiring?
- Does each remote building's faction avoid home-area, hauling, and raid
  side effects?
- Can the intended pawns open the doors, use the beds, and reach the
  items?
- Does storage priority keep remote stockpiles from draining the base?
- Does clearing remove zone cells only where it must?

## Further Information

- Ludeon's partial source in the install:
  `Source/RimWorld/Thing/Building/Various/Building_Door.cs`,
  `Source/RimWorld/Thing/Building/Storage/Building_Storage.cs`,
  `Source/RimWorld/ThingComps/CompForbiddable.cs`, and
  `Source/Verse/AI/JobDrivers/DoBill/JobDriver_DoBill.cs`
- Vanilla defs: `Data/Core/Defs/ThingDefs_Buildings/Buildings_Power.xml`,
  `Buildings_Temperature.xml`, and `Data/Core/Defs/DutyDefs/Duties_Misc.xml`
  (assault duties)
- RimWorld Wiki, [Custom Comp Classes](https://rimworldwiki.com/wiki/Modding_Tutorials/Custom_Comp_Classes)
  and [Modifying classes](https://rimworldwiki.com/wiki/Modding_Tutorials/Modifying_classes)
