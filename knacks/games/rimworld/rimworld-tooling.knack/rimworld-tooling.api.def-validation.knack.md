# RimWorld Def Validation

Load this companion after the
[RimWorld tooling overview](rimworld-tooling.overview.knack.md) when
building or extending a validator that checks a mod's `Defs/` and
`Patches/` against the game's real types, without launching the game.
The general technique, validating a plugin's data files against a closed
host's types, is in FieldManual
`knacks/dotnet/assembly-inspection.knack/`; this companion gives
RimWorld's loader rules and shapes.

Reviewed: 2026-10-14, against RimWorld 1.6.4871. Evidence marks:
**[decomp]** means verified only by decompiling `Assembly-CSharp.dll` from
1.6.4871; **[observed]** means seen on 1.6.4871 but not traced to a
published source.

## Purpose and Limits

A launch takes a minute or more and reports XML problems among hundreds
of log lines. A validator that reflects over the game's assembly and the
mod's own assembly catches the common mistakes in seconds: a misspelled
field, an enum value that does not parse, a class that does not exist, a
reference to a missing def, a texture that is not there, a patch whose
XPath selects nothing.

It is a stand-in, not the loader. It cannot run `ConfigErrors()`, resolve
every custom parser, or see what other mods change; the in-game smoke run
remains the final check. Exit nonzero on any problem so a script or CI
step can gate on it.

## Indexes to Build First

1. **Types.** Load `Assembly-CSharp.dll` and the mod's built assembly.
   Catch `ReflectionTypeLoadException` from `GetTypes()` and keep the
   types that loaded.
2. **Known defs.** Parse every XML file under the install's
   `Data/*/Defs` and the mod's `Defs/`, and record each def as
   `"<ElementName>/<defName>"`. The element name is the def's type.
3. **Merged defs for patches.** Copy every def element from vanilla and
   the mod under one `<Defs>` root, as the game merges them before
   applying patches [decomp]. Copy only the def types the patches' XPaths
   name, when that is known, to save memory.
4. **Texture paths.** The mod's `Textures/` files, plus vanilla's
   lowercase list from `globalgamemanagers` (and DLC asset bundle
   manifests, if needed).

## Resolving Types

Mirror the loader's rules [decomp]:

- a namespace-qualified name resolves directly, mod types first;
- a short name resolves only among the game's own namespaces (`Verse`,
  `RimWorld`, `Verse.AI`, `Verse.AI.Group`, `RimWorld.Planet`, and a few
  others) and types in no namespace, so a mod class written without its
  namespace is an error;
- a def element's name is its type.

## Checking Elements Against Fields

For each def, walk its child elements against the type:

- **Find the field** by name on the type or any base type, private fields
  included (`GetField(name, Public | NonPublic | Instance)` does not
  return a base class's private fields, so walk `BaseType` yourself). If
  no field has the name, look for one carrying
  `[LoadAlias("<name>")]`. If neither exists, report the element.
- **Check the value by field type:**
  - enum: the text must parse as a member;
  - `System.Type`: the text must resolve as a class;
  - a `Def` subclass: the text must be a known def of that type;
  - `List<T>`: check each `<li>`; honor a `Class` attribute on it;
  - a nested object: honor a `Class` attribute (it must name a subclass of
    the field's type), then recurse.
- **Custom shapes** need special cases or explicit skips: `costList` uses
  def names as element names (`<Steel>50</Steel>`), `statBases` uses stat
  names the same way, and a thing filter's `<categories>` lists
  `ThingCategoryDef` names as strings.

Make every skip visible. A validator once treated `<categories>` under a
pawn kind's `backstoryFiltersOverride` as thing categories and reported
real backstory categories as missing [observed]; the honest fix is a
special case for that path, not a quieter check.

## Checking Textures

For each `texPath`, `iconPath`, `uiIconPath`, or similar field, accept the
path if the mod has `Textures/<path>.png`, a multi-sided
`<path>_north.png`, or a folder `<path>/`, or if vanilla's list has the
lowercase equivalent. A texture check fails while art is still being drawn;
expect that and say so in the output rather than disabling the check.

## Checking Patches

For each file under `Patches/`:

1. The root must be `<Patch>` and every child an `<Operation>` [decomp].
2. Its `Class` must resolve to a subclass of `Verse.PatchOperation`.
3. Each child element must be a field of that class, including private
   fields of base classes (`success` lives on `PatchOperation` itself).
   Nested operations, as in `PatchOperationSequence`, are checked
   recursively.
4. The XPath must parse and must select at least one node in the merged
   defs, unless `<success>` is `Always` or `Invert`.
5. For `PatchOperationAdd`, check what is added against the node it lands
   in; for `PatchOperationInsert`, against the target's parent. Find that
   node's type by walking down from its def: a field's type, a list's
   element type for `<li>`, or the class a `Class` attribute names. Then
   an `<li>` added to a non-list, or a class that is not a subclass of the
   list's element type, fails.

Prove the patch checks with deliberately broken patches: a wrong
`defName`, a misspelled field, a missing class, a list item added to a
non-list, and a non-`ThinkNode` added to a think tree's `subNodes` each
failed the validator [observed].

## Implementation Notes

- The PowerShell traps (`.LocalName` versus `.Name`, partial type loads
  on Windows PowerShell 5.1) are in the
  [tooling overview](rimworld-tooling.overview.knack.md) and the stock
  assembly-inspection knack.
- Report a missing class as fatal: it drops the whole def in game (see
  [Defs and XML](../rimworld-modding.knack/rimworld-modding.api.defs-and-xml.knack.md)).
- Print each problem with a path from the def down to the element, such
  as `BW_SteamPress/comps/li[CompProperties_PressureGauge]/<maxPresure>`
  (with the class's full name in practice), and a summary count last.
- Re-run the validator after every game update. New fields and renamed
  ones appear first here.

## Review Checklist

- Does the validator load both assemblies and survive partial type loads?
- Does it resolve classes with the loader's namespace rules?
- Does it walk base types and honor `[LoadAlias]` and `Class`?
- Are custom-parsed shapes handled or visibly skipped?
- Does every XPath have to match, and is added content type-checked?
- Was each check proved by a broken input?

## Further Information

- RimWorld Wiki, [XML file structure](https://rimworldwiki.com/wiki/Modding_Tutorials/XML_file_structure),
  [PatchOperations](https://rimworldwiki.com/wiki/Modding_Tutorials/PatchOperations),
  and [Compatibility with defs](https://rimworldwiki.com/wiki/Modding_Tutorials/Compatibility_with_defs)
- Microsoft, `Type.GetField` and binding flags:
  <https://learn.microsoft.com/en-us/dotnet/api/system.type.getfield>
- Microsoft, `ReflectionTypeLoadException`:
  <https://learn.microsoft.com/en-us/dotnet/api/system.reflection.reflectiontypeloadexception>
- [RimWorld Defs and XML](../rimworld-modding.knack/rimworld-modding.api.defs-and-xml.knack.md)
  for the loader rules being mirrored
- FieldManual `knacks/dotnet/assembly-inspection.knack/`, for
  validating data files against host types in general
