# RimWorld Modding

Load this knack when creating, extending, or reviewing a RimWorld 1.6 mod:
XML definitions, C# assemblies, XML patches, or the art and text the game
loads. It gives the mental model and the working rules. Load the API
companion for the subsystem in hand before writing code against it.

Reviewed: 2026-10-14, against RimWorld 1.6.4871.

Evidence marks: **[decomp]** means verified only by decompiling
`Assembly-CSharp.dll` from RimWorld 1.6.4871. **[observed]** means seen in
game testing on 1.6.4871 but not traced to a published source. Unmarked
claims rest on the sources under Further Information or on the game's own
`Data` XML.

## Companions

- [Defs and XML](rimworld-modding.api.defs-and-xml.knack.md) — inheritance,
  `Class=`, configuration errors, mod extensions, translation, patches
- [Lifecycle and Ticking](rimworld-modding.api.lifecycle-and-ticking.knack.md)
  — tick methods, comps, held things, components, static assets, saving
- [Things and Maps](rimworld-modding.api.things-and-maps.knack.md) —
  spawning, footprints, indestructible things, clearing, roofs, terrain
- [Buildings and Storage](rimworld-modding.api.buildings-and-storage.knack.md)
  — power, ownership side effects, doors, zones, stockpiles, bills
- [Pawns and AI](rimworld-modding.api.pawns-and-ai.knack.md) — think trees,
  job givers, lords, duties, fleeing
- [Pawn Generation and Factions](rimworld-modding.api.pawn-generation-and-factions.knack.md)
  — generation requests, pawn kinds, factions, needs, breaks, custom races
- [Combat and Effects](rimworld-modding.api.combat-and-effects.knack.md) —
  explosions, fire, projectiles, radial limits, damage hooks
- [UI and Debug](rimworld-modding.api.ui-and-debug.knack.md) — gizmos, the
  inspect pane, map GUI, debug actions, messages

Verification and build setup live in the sibling
[testing](../rimworld-testing.knack/rimworld-testing.overview.knack.md) and
[tooling](../rimworld-tooling.knack/rimworld-tooling.overview.knack.md)
filesets.

## Mental Model

RimWorld is data driven. At startup the game merges the XML *defs* of Core,
each active DLC, and each active mod into typed `Def` objects, and its C#
reads those objects at run time. A mod contributes some of:

- `About/About.xml` — identity, supported versions, dependencies;
- `Defs/` — new definitions: things, jobs, research, think trees;
- `Patches/` — XPath edits to XML before it becomes defs;
- `Assemblies/` — compiled C# that the XML names through `thingClass`,
  `compClass`, `workerClass`, or a `Class="..."` attribute;
- `Textures/`, `Sounds/`, and `Languages/` — assets and translations.

`LoadFolders.xml` can select different folders per game version. A mod that
targets only the current release can keep everything at its root.

Most features are new defs plus small classes plugged into vanilla
extension points: a `ThingComp` on a building, a `JobDriver` for a new job,
a `ThinkNode` in a think tree, or a `MapComponent` or `GameComponent` for
state that belongs to no single thing. Replacing vanilla methods is the
exception.

Startup order matters for early code. `Mod` subclasses are constructed
before any XML loads; every mod's XML is then combined, patched,
inheritance-resolved, and built into cross-referenced defs; `[DefOf]`
fields are bound; `[StaticConstructorOnStartup]` classes run last
[decomp]. Defs and DefOfs
are not usable in a `Mod` constructor.

## Identity and Load Order

- A `packageId` may contain only ASCII letters, digits, and periods: at
  least one period, none leading, trailing, or doubled, at most 60
  characters, and never the word "Ludeon". Hyphens and underscores fail, so
  an account or repository name often cannot be reused as is.
  `ModUpdating.txt` documents the rule, and `ModMetaData.PackageIdFormatRegex`
  enforces it [decomp].
- Treat the `packageId` and every `defName` as frozen from the first public
  release. Saves and other mods refer to them by name. Rename before then,
  and add a test that the old prefix appears nowhere. A label can change
  freely; the `defName` behind it cannot.
- List in `<supportedVersions>` only the versions you test.
- Declare hard dependencies in `<modDependencies>` and ordering needs in
  `<loadAfter>` and `<loadBefore>`.
- Official content loads in release order: Core, Royalty, Ideology,
  Biotech, Anomaly, Odyssey. Each DLC's `About.xml` pins this with
  `forceLoadAfter` and `forceLoadBefore`. Later DLCs inherit from earlier
  DLCs' abstract defs (Odyssey's `VacskinGland` uses a Royalty parent), so
  any tool that writes `ModsConfig.xml` must keep that order. Listed
  alphabetically, the DLCs produced "XML error: Could not find parent node
  named "BodyPartBionicImperialBase"" and then a crash in map generation
  [observed].
- Prefix every `defName`, texture folder, and Harmony ID with a short mod
  tag, such as `BW_SteamPress` for a fictional "Brass Works" mod, to avoid
  collisions with vanilla and with other mods.

## Defs, DefOfs, and Comps

A `defName` is unique per def type. Reach defs from C# through a `[DefOf]`
class. Its fields are matched by name (or by `[DefAlias]`), stay null until
binding, and must never be used as default field values inside other defs;
the game warns about uninitialized DefOfs for exactly that mistake.
DLC-only fields carry `[MayRequireBiotech]` or a sibling attribute and stay
null when that DLC is inactive [decomp].

```csharp
[DefOf]
public static class BW_DefOf
{
    public static ThingDef BW_SteamPress;

    [MayRequireBiotech]
    public static GeneDef BW_BrassLungs;

    static BW_DefOf() =>
        DefOfHelper.EnsureInitializedInCtor(typeof(BW_DefOf));
}
```

Put reusable behavior in a `ThingComp` with its `CompProperties`, and
static per-def data in a `DefModExtension`. Both can be attached to vanilla
defs by patch, without subclassing.

## Changing Vanilla: XML First, Harmony Last

Prefer, in order:

1. new defs that reuse vanilla classes;
2. subclasses named from XML (`thingClass`, `Class=`);
3. comps and mod extensions;
4. `PatchOperation`s that edit vanilla XML before it becomes defs;
5. Harmony, only for behavior that no extension point reaches.

A Harmony patch affects every caller of the method, interacts with every
other mod's patches, and can break silently when the game updates. When one
is unavoidable, depend on the Harmony mod (`brrainz.harmony`) through
`<modDependencies>` instead of shipping `0Harmony.dll`; prefer postfixes;
avoid prefixes that skip the original; and record each patch with the
reason no alternative worked. A main-menu button is a fair example: vanilla
builds that list inline in `MainMenuDrawer.DoMainMenuControls`, with no def
or virtual method to extend [decomp].

## DLC-Optional Content

- `MayRequire="Ludeon.RimWorld.Biotech"` (a comma list means all of them)
  or `MayRequireAnyOf` skips a top-level def, a list entry, or an operation
  inside a `PatchOperationSequence` when that content is absent.
- On a plain def reference it only suppresses the missing-def error. If the
  def exists, it still loads.
- Since 1.6, `MayRequire` works when the `ParentName` comes from the
  required content.
- In C#, gate on `ModsConfig.BiotechActive` and its siblings.

## Building Against the Game

Target .NET Framework 4.7.2, reference `Assembly-CSharp.dll` and the Unity
modules you use from `RimWorldWin64_Data/Managed` in the install with
`Private=false`, and build into `Assemblies/`. Never hard-code the install
location: read it from an environment variable such as `RIMWORLD_DIR`, and
fail the build with a clear message when it is unset. FieldManual
`knacks/dotnet/framework-builds.knack/` explains the general pattern;
the
[tooling overview](../rimworld-tooling.knack/rimworld-tooling.overview.knack.md)
has a RimWorld project template.

## Verify, Don't Guess

The game's API is large and changes between versions, and plausible names
are often wrong. There is no `Flick` work type: flicking is a work giver
named `Flick` under `BasicWorker`, beside a job and a designation of the
same name. In 1.6, `ThingCategoryDefOf` has
no `FoodMeals` field although the def still exists (declare your own DefOf
for it), `FireUtility.TryStartFireIn` requires an instigator,
`WorkGiverDef.equivalenceGroup` is a def reference,
`JobDef.checkOverrideOnDamage` is an enum, and `DesignationDef` takes a
`texturePath` [decomp]. Check each API before writing against it, using:

- Ludeon's partial C# in the install's `Source/` folder;
- vanilla defs under `Data/<Core or DLC>/Defs`, for field names, enum
  values, and `defName`s;
- a decompile of `Assembly-CSharp.dll`; and
- reflection over that assembly, for fields and signatures.

Record what was verified by decompilation and against which build. It is
evidence about one version, not a contract.

## Sharp Edges

- `Entity.TickRare()` throws, minified buildings tick inside their crates,
  and comps keep ticking after a mid-loop kill (Lifecycle and Ticking).
- Spawning an item into a full cell shoves the occupant aside or destroys
  it (Things and Maps).
- A Bomb explosion puts out every fire its wave touches, and every
  projectile needs a launcher (Combat and Effects).
- A player-owned building far from the base pulls colonists to fight its
  fires and raiders to wreck it (Buildings and Storage).
- Interrupting reactions belong in the constant think tree, not the main
  one (Pawns and AI).
- Map-component GUI code runs on every IMGUI event (UI and Debug).
- Static textures and materials outside a `[StaticConstructorOnStartup]`
  class draw a dev-mode warning.

## Review Checklist

- Is the `packageId` valid and frozen, and are `supportedVersions` honest?
- Is every `defName`, texture path, and Harmony ID prefixed?
- Is DLC content guarded by `MayRequire` or `ModsConfig` checks?
- Is each Harmony patch justified and recorded?
- Does the build read the install path from the environment?
- Was every API checked against `Source`, `Data`, a decompile, or
  reflection, with the game build noted?
- Have the static, definition, and in-game checks of the testing fileset
  run?

## Further Information

- Ludeon Studios, `ModUpdating.txt` in the RimWorld install root: mod folder
  structure, `About.xml` fields, and the `packageId` format.
- Ludeon Studios, "Modder Primer for 1.6":
  <https://docs.google.com/document/d/e/2PACX-1vRKE9u5ZW_zG45pxzwNvy4sxvozDeqtxlxpac5jwenOeW6liQCPgmPl9bIbtcMuqL1NPIDHOLFg64M_/pub>
- Ludeon Studios, multi-version support with `LoadFolders.xml`:
  <https://docs.google.com/document/d/1_DmcLpIvHIQ5AxVLYrn9_iwkOqArlgcWcyE_6RSDG6M>
- RimWorld Wiki, Modding Tutorials:
  <https://rimworldwiki.com/wiki/Modding_Tutorials>; especially
  [Mod Folder Structure](https://rimworldwiki.com/wiki/Modding_Tutorials/Mod_Folder_Structure),
  [About.xml](https://rimworldwiki.com/wiki/Modding_Tutorials/About.xml),
  [MayRequire](https://rimworldwiki.com/wiki/Modding_Tutorials/MayRequire),
  [Linking XML and C#](https://rimworldwiki.com/wiki/Modding_Tutorials/Linking_XML_and_C),
  and [RimWorld 1.6 Mod Updates](https://rimworldwiki.com/wiki/Modding_Tutorials/RimWorld_1.6_Mod_Updates).
- RimWorld Wiki, Harmony:
  <https://rimworldwiki.com/wiki/Modding_Tutorials/Harmony>
- Harmony documentation: <https://harmony.pardeike.net/articles/intro.html>
