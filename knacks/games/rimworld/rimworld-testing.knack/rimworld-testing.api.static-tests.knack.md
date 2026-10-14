# RimWorld Static Tests

Load this companion after the
[RimWorld testing overview](rimworld-testing.overview.knack.md) when
setting up tests that run without launching the game: unit tests of mod
logic and regression tests over the mod's XML and the game's own data.

Reviewed: 2026-10-14, against RimWorld 1.6.4871. Evidence marks:
**[decomp]** means verified only by decompiling `Assembly-CSharp.dll` from
1.6.4871; **[observed]** means seen in game testing on 1.6.4871 but not
traced to a published source.

## A RimWorld-Free Logic Library

Engine types cannot be constructed outside a running game, so code that
touches `Thing`, `Map`, or `Pawn` is hard to unit-test. Move everything that
does not need them into a separate class library with **no** reference to
the game: geometry, timing schedules, damage and repair arithmetic, layout
plans, random-split rules, text formatting. The mod assembly references the
library and ships its DLL in `Assemblies/` beside its own.

```text
Source/
  BrassWorks.csproj            net472; references the game and the core
  BrassWorks.Core/             netstandard2.0; references nothing
Tests/
  BrassWorks.Tests/            test framework; references the core only
```

FieldManual `knacks/dotnet/shared-library.knack/` covers the
general pattern and the `netstandard2.0` language traps: the C# 7.3
default, a pinned numeric `LangVersion`, `IsExternalInit` for records,
and the missing index and range types. What RimWorld adds:

- Unity's `Mathf` and the game's `Rand` are unavailable in the library;
  use `System.Math`, and pass random numbers in as arguments so tests
  control them;
- the game loaded the `netstandard2.0` library from `Assemblies/` with no
  facade assemblies shipped beside it [observed]; the install's `Managed`
  folder already holds `netstandard.dll`.

Design the library so the game half only *executes* its results: a layout
plan lists every cell, wall, and placement in local coordinates; the game
code spawns them. Then the plan's invariants (nothing overlaps, every room
is reachable, every door is clear) are fast unit tests, and an in-game
check can compare what was built with what the plan predicted. Expected
values computed by the same model, such as the repair time for a given
injury, catch drift between model and game when an in-game check compares
them [observed].

Pure logic also lets a reviewer settle a dispute by running the code
instead of arguing about it.

## XML Regression Tests

Parse the mod's `Defs/`, `Patches/`, `About/`, and `Textures/` with
`System.Xml` or LINQ to XML in the test project. Useful families:

- **House rules**: every `defName` and texture path carries the mod
  prefix; every texture path the defs name exists; every player-visible
  description is one line; the `packageId` matches its required format.
- **Rules learned from failures**: once a config error or flake is fixed,
  pin the rule. Examples: impassable buildable buildings have
  `fillPercent` 1; minifiable buildings have a thing category; a
  long-range turret is not `ai_combatDangerous`; a removed stopgap def
  stays removed.
- **Cross-references**: every reference to a mod def resolves to a def the
  mod declares, and every vanilla `defName` used exists in the installed
  `Data` folder.
- **Text that must match a specification**: if a design document lists
  twenty status lines, assert that the XML holds exactly those, unique and
  in order. A `DefModExtension` holding the lines keeps them translatable
  and testable at once.

## Tests Against the Game's Own Data

Some facts can be checked only against vanilla XML: which think trees each
race uses, whether a patch lands where intended, whether an inherited field
exists. Read them from the installed game:

- locate it from `RIMWORLD_DIR`, then the default Steam library path;
- if neither exists, return early with a message that says the test was
  skipped, so the suite still passes on a machine without the game
  [observed];
- resolve `Name`/`ParentName` inheritance yourself when a fact may be
  inherited;
- note the game build (`Version.txt`) the expectations were written
  against.

**Apply a patch to the real tree.** Load the vanilla think-tree XML, apply
the mod's `PatchOperationAdd` with `System.Xml` (select the XPath, insert
the value with the given order), and assert the result: the new giver is
first inside the gate node of each target tree, ahead of vanilla's own flee
givers. A wrong XPath or a changed vanilla tree then fails a unit test
instead of a game launch [observed].

**Check coverage across races.** Walk every `ThingDef` with a `race` in
Core and the installed DLCs, resolve its `thinkTreeConstant`, and assert
that every race expected to react runs one of the patched trees; list the
ones that do not (mechanoids, drones, babies, entities) explicitly, so a
new DLC race shows up as a test failure.

## Review Checklist

- Does logic that needs no engine live in the RimWorld-free library, with
  tests?
- Is every lesson from a config error or flake pinned by a test?
- Do tests that need the game's data skip cleanly without it?
- Are patches tested against the real vanilla XML?
- Do tests record the game build their expectations came from?

## Further Information

- Ludeon Studios, `ModUpdating.txt` in the install root (folder layout)
- RimWorld Wiki, [Setting up a solution](https://rimworldwiki.com/wiki/Modding_Tutorials/Setting_up_a_solution)
- FieldManual `knacks/dotnet/shared-library.knack/`, for the
  split, the polyfills, and the language-version decision
- Microsoft, .NET Standard versions and API coverage:
  <https://learn.microsoft.com/en-us/dotnet/standard/net-standard>
