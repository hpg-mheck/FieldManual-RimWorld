# RimWorld Defs and XML

Load this companion after the
[RimWorld modding overview](overview.knack.md) when writing
or reviewing def XML, patches, translations, or the C# types XML loads into.

Reviewed: 2026-10-14, against RimWorld 1.6.4871. Evidence marks:
**[decomp]** means verified only by decompiling `Assembly-CSharp.dll` from
1.6.4871; **[observed]** means seen in game testing on 1.6.4871 but not
traced to a published source.

## Loading Model

Every file under `Defs/` has a `<Defs>` root. Each child element is one def,
and its element name is the def's C# type (`ThingDef`, `JobDef`, or a mod's
own `Def` subclass by name). In 1.6, a top-level element that names no type
the game can find, or a type that does not derive from `Def`, is logged as
an error at startup.

The loader runs in this order [decomp]:

1. every active mod's def files are combined into one in-memory `<Defs>`
   document, in load order;
2. every mod's patches are applied to that document, in load order;
3. `Name`/`ParentName` inheritance is resolved;
4. each element is deserialized into its type, and def references are
   resolved by name;
5. DefOfs are bound, and, in dev mode only, `ConfigErrors()` runs on
   every def.

Two consequences follow. A patch can edit any mod's XML, whatever the load
order, because all of it is present. And a patch sees the XML *before*
inheritance: a child that inherits `<comps>` from its parent has no
`<comps>` node of its own, so an XPath into the child's comps selects
nothing.

1.6 deserializes defs with a new generated-IL loader;
`-legacy-xml-deserializer` on the command line selects the old one, which
helps when bisecting a loader bug [decomp].

## Fields and Values

- An element maps to the field of the same name, public or private, on the
  type or a base type. An element matching no field is an error ("XML
  error: ... doesn't correspond to any field in type ...") [decomp].
- `[LoadAlias("oldName")]` on a field accepts an old element name, so a
  renamed field keeps old XML working. For example, the `<priority>` of a
  `StorageSettings` loads into the field `priorityInt` [decomp].
- Enum fields parse the enum member's name. Flag enums accept list items.
- Def-typed fields hold a `defName` and are resolved after loading.
- Lists use `<li>` items. Some lists have custom parsers: `costList` and
  `statBases` use the item or stat name as the element name
  (`<Steel>50</Steel>`), and thing filters list category names as strings.
- `IgnoreIfNoMatchingField="True"` silences the error for a field the game
  does not know, such as custom `About.xml` data.
- In-game text belongs on one line in XML: line breaks and indentation
  inside `description` appear literally in the game [observed]. Write `\n`
  where a break is wanted; the loader turns it into a newline [decomp], and
  vanilla descriptions rely on that.

## Inheritance

```xml
<Defs>
  <ThingDef Name="BW_MachineBase" ParentName="BuildingBase"
      Abstract="True">
    <tickerType>Rare</tickerType>
    <comps>
      <li Class="CompProperties_Flickable" />
    </comps>
  </ThingDef>

  <ThingDef ParentName="BW_MachineBase">
    <defName>BW_SteamPress</defName>
    <label>steam press</label>
    <thingClass>BrassWorks.Building_SteamPress</thingClass>
    <comps>
      <li Class="BrassWorks.CompProperties_PressureGauge">
        <maxPressure>120</maxPressure>
      </li>
    </comps>
  </ThingDef>
</Defs>
```

The resolved `BW_SteamPress` has both comps. The merge rules [decomp]:

- `Name` registers a node as a parent; `ParentName` names the parent;
  `Abstract="True"` keeps a node from becoming a def of its own.
- A child's element overwrites the parent's element of the same name,
  recursively.
- `<li>` items are appended to the parent's list, never matched by
  position. To replace a list, put `Inherit="False"` on the child's list
  element.
- An element with text content replaces the parent's content outright.
- A parent may live in another mod or DLC. Reusing a `Name` within one mod
  is an error. Prefix names as you prefix `defName`s.

## Class Attributes

`Class="..."` on a nested object or list item selects a subclass of the
field's declared type, as in the comp example above. Resolution rules
[decomp]:

- Vanilla types resolve by short name from the game's own namespaces
  (`Verse`, `RimWorld`, `Verse.AI`, `Verse.AI.Group`, `RimWorld.Planet`,
  and a few others), as do types declared in no namespace at all.
- A mod's types need their namespace-qualified name:
  `BrassWorks.CompProperties_PressureGauge`.
- With 1.6's default loader, a class that cannot be found **drops the whole
  def**: the log shows "Exception loading def from file ..." with "Could
  not find type named ..." inside it, and the def never reaches the
  database, so everything that refers to it fails to resolve.
- Under `-legacy-xml-deserializer`, the old loader instead logs "Could not
  find type named ..." and *falls back to the declared type*, so the def
  loads and silently behaves like vanilla. Treat either error as fatal in
  testing.

Def-level class fields work the same way: `thingClass`, `compClass`,
`driverClass`, `workerClass`, and `giverClass`.

## Configuration Errors

After loading, the game calls `ConfigErrors()` on every def and logs each
result as "Config error in ...", but **only when dev mode is on**; a def
with `ignoreConfigErrors` is skipped [decomp]. A player without dev mode
never sees them, so develop and test with dev mode enabled. Some rules that
catch new content [decomp]:

- a `PawnKindDef` without `combatPower` ("has no combatPower"; the default
  is -1), or a humanlike one without `initialWillRange` and
  `initialResistanceRange`;
- a `BodyDef` with no part vulnerable to frostbite;
- a race using mechanoid flesh without butcher products;
- a `ResearchProjectDef` whose `techLevel` is `Undefined`;
- a minifiable `ThingDef` with no `thingCategories`, or an unminifiable
  building that has them;
- an impassable, player-buildable building whose `fillPercent` is below 1
  ("can be shot/seen over");
- `fillPercent` 1 on a thing that is not an edifice;
- a haulable or minifiable thing without a `Mass` stat;
- a verb whose forced-miss radius does not match whether its projectile
  explodes (see Combat and Effects).

Research cost is scaled by tech level: a colony below the project's level
(capped at Industrial) pays an extra half of the listed cost for each
level of difference, and a colony at or above it pays the listed cost
[decomp].

Add rules of your own by overriding `ConfigErrors()` on your `Def`
subclasses, `CompProperties`, and `DefModExtension`s. Config errors appear
only in a running game, so the smoke test must fail on any of them.

## Mod Extensions

A `DefModExtension` attaches typed data to any def without a subclass:

```xml
<ThingDef ParentName="BW_MachineBase">
  <defName>BW_SteamHammer</defName>
  <modExtensions>
    <li Class="BrassWorks.SteamProfile">
      <hissChance>0.2</hissChance>
    </li>
  </modExtensions>
</ThingDef>
```

Read it with `def.GetModExtension<SteamProfile>()`. To add one to a vanilla
def, use `PatchOperationAddModExtension`. Extensions are a good home for
data-driven lists, such as flavor lines, that should be translatable and
testable.

## Translation-Ready Text

- Every string field and every list of strings in a def can be replaced
  through `Languages/<Language>/DefInjected/<DefType>/*.xml`, unless the
  field carries `[NoTranslate]`.
- `[MustTranslate]` marks a field whose missing translations are reported.
  Without it, the check reports only strings containing a space [decomp].
- `[TranslationCanChangeCount]` on a `List<string>` lets a translation
  supply a different number of entries [decomp].
- Objects inside lists are addressed by index, so a list on an extension is
  `BW_SteamHammer.modExtensions.0.hissLines` [decomp].
- Text built in C# belongs in `Languages/English/Keyed/*.xml` and is read
  with `"BW_KeyName".Translate()`. Save language files as UTF-8.

## Patches

A file under `Patches/` has a `<Patch>` root and `<Operation>` children:

```xml
<Patch>
  <Operation Class="PatchOperationAdd">
    <xpath>Defs/ThinkTreeDef[defName="HumanlikeConstant"]
      /thinkRoot/subNodes
      /li[@Class="ThinkNode_ConditionalCanDoConstantThinkTreeJobNow"]
      /subNodes</xpath>
    <order>Prepend</order>
    <value>
      <li Class="BrassWorks.JobGiver_FleeSteamLeak" />
    </value>
  </Operation>
</Patch>
```

- The vanilla operations are `PatchOperationAdd`, `Insert`, `Remove`,
  `Replace`, `AttributeAdd`, `AttributeSet`, `AttributeRemove`, `SetName`,
  `AddModExtension`, `Sequence`, `Conditional`, `Test`, and `FindMod`
  [decomp].
- `PatchOperationAdd` takes `<order>` `Append` (the default) or `Prepend`.
- Every operation takes `<success>`: `Normal` fails when the XPath selects
  nothing; `Always`, `Never`, and `Invert` change that. A silent miss is
  the usual patch bug, so avoid `Always` unless the miss is expected.
- A `PatchOperationSequence` reports one failure for the whole sequence;
  test each operation alone first.
- For content from another mod or DLC, use `PatchOperationFindMod` or
  `MayRequire` on items inside a sequence.
- An XPath may be split over lines between steps, as above, to keep lines
  short; the game's XPath engine accepts it [observed].
- Remember that patches see uninherited XML.

## Validating XML Against Types

The game is the final judge, but a run takes a minute or more. A
reflection-based validator checks in seconds that every element maps to a
real field, enums parse, classes resolve, def references exist, textures
exist, and each patch's XPath selects something in the merged defs. See
[Def Validation](../tooling/def-validation.knack.md).

## Sharp Edges

- A misspelled class in `Class=` drops the whole def under the default
  loader, and with the legacy loader quietly falls back to the vanilla
  base type.
- Inherited list items accumulate. A child that "redefines" `comps`
  without `Inherit="False"` gets both sets.
- Patching a child for something it inherits finds nothing.
- `MayRequire` on a def reference only silences the error; the def still
  loads if present.
- Plausible field names are often wrong; check them against `Data` or the
  type before relying on them.

## Review Checklist

- Do all mod classes in XML carry their namespace?
- Does every new `PawnKindDef`, `ResearchProjectDef`, and building pass the
  config rules above?
- Are list overrides explicit about `Inherit="False"`?
- Does each patch target uninherited XML and fail loudly on a miss?
- Are player-visible strings in translatable fields, and lists of lines
  marked `[MustTranslate]` where appropriate?
- Did a validator run, and did the in-game run log no XML or config
  errors?

## Further Information

- RimWorld Wiki, [Defs](https://rimworldwiki.com/wiki/Modding_Tutorials/Defs),
  [XML file structure](https://rimworldwiki.com/wiki/Modding_Tutorials/XML_file_structure),
  and [Def classes](https://rimworldwiki.com/wiki/Modding_Tutorials/Def_classes)
- RimWorld Wiki, [PatchOperations](https://rimworldwiki.com/wiki/Modding_Tutorials/PatchOperations)
  and [MayRequire](https://rimworldwiki.com/wiki/Modding_Tutorials/MayRequire)
- RimWorld Wiki, [DefModExtension](https://rimworldwiki.com/wiki/Modding_Tutorials/DefModExtension)
  and [ConfigErrors](https://rimworldwiki.com/wiki/Modding_Tutorials/ConfigErrors)
- RimWorld Wiki, [Localization](https://rimworldwiki.com/wiki/Modding_Tutorials/Localization)
- Ludeon Studios, `ModUpdating.txt` in the install root: the `Languages/`
  layout and `IgnoreIfNoMatchingField`
- Ludeon Studios, "Modder Primer for 1.6" (unknown top-level def elements;
  `MayRequire` with `ParentName`):
  <https://docs.google.com/document/d/e/2PACX-1vRKE9u5ZW_zG45pxzwNvy4sxvozDeqtxlxpac5jwenOeW6liQCPgmPl9bIbtcMuqL1NPIDHOLFg64M_/pub>
- The game's own `Data/<Core or DLC>/Defs` XML, for working examples of
  every def type
