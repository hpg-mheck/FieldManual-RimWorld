# RimWorld Mod Tooling

Load this knack when setting up a RimWorld 1.6 mod's build, or when an API,
field name, def name, or texture path must be verified instead of guessed.
It covers compiling against the game, decompiling it, querying it by
reflection, and finding vanilla assets.

The general techniques are in FieldManual's stock knacks:
`knacks/dotnet/net-framework-builds.knack/` (host references, MSBuild),
`knacks/dotnet/visual-studio.knack/` (solutions, trusting only builds you
saw succeed), `knacks/dotnet/closed-assembly-inspection.knack/` (licence,
`ilspycmd`, reflection from PowerShell, validating data files),
`knacks/windows/agent-shells.knack/` (PowerShell and Git Bash
differences), and `knacks/games/steam.knack/` (Workshop uploads). This
knack keeps what is particular to RimWorld.

Reviewed: 2026-10-14, against RimWorld 1.6.4871. Evidence marks:
**[decomp]** means verified only by decompiling `Assembly-CSharp.dll` from
1.6.4871; **[observed]** means seen on 1.6.4871 but not traced to a
published source.

## Companions

- [Def Validation](rimworld-tooling.api.def-validation.knack.md) — a static
  stand-in for the game's XML loader, built on reflection

## Building Against the Game

RimWorld mods compile to .NET Framework 4.7.2 class libraries that
reference the game's managed assemblies without copying them:

```xml
<PropertyGroup>
  <TargetFrameworkVersion>v4.7.2</TargetFrameworkVersion>
  <LangVersion>12.0</LangVersion>
  <OutputPath>..\Assemblies\</OutputPath>
  <RimWorldManaged>$(RIMWORLD_DIR)\RimWorldWin64_Data\Managed</RimWorldManaged>
</PropertyGroup>
<ItemGroup>
  <Reference Include="Assembly-CSharp">
    <HintPath>$(RimWorldManaged)\Assembly-CSharp.dll</HintPath>
    <Private>false</Private>
  </Reference>
  <Reference Include="UnityEngine.CoreModule">
    <HintPath>$(RimWorldManaged)\UnityEngine.CoreModule.dll</HintPath>
    <Private>false</Private>
  </Reference>
</ItemGroup>
<Target Name="CheckRimWorldDir" BeforeTargets="ResolveAssemblyReferences"
    Condition="!Exists('$(RimWorldManaged)\Assembly-CSharp.dll')">
  <Error Text="Set RIMWORLD_DIR to the RimWorld install folder." />
</Target>
```

FieldManual `knacks/dotnet/net-framework-builds.knack/` explains the
environment variable, the guard target, `Private=false`, and linking
non-code content; pin `LangVersion` to a number as its .NET Standard
sibling advises. What RimWorld adds:

- Add Unity modules only as needed: `UnityEngine.IMGUIModule` for `GUI`
  and `Event`, `UnityEngine.TextRenderingModule` for `TextAnchor`
  [decomp].
- Do not ship Harmony or other libraries' DLLs; depend on their mods.
- Community reference-assembly packages on NuGet (for example
  `Krafs.Rimworld.Ref`) remove the need for the game on a build machine;
  pin one that matches the build you test against.
- Link `About/`, `Defs/`, `Patches/`, `Textures/`, and `Languages/` into
  the project so the whole mod is browsable in the IDE.
- Keep `Assemblies/`, `obj/`, and `bin/` out of version control unless
  the release process needs the built DLL committed.

A failed compile leaves the old DLL in `Assemblies/`, and every later test
then runs old code [observed]; trust only builds you saw succeed (FieldManual
`knacks/dotnet/visual-studio.knack/`).

The game loads every `.dll` under a mod's `Assemblies/` folder,
subfolders included. A DLL that fails to load stops the loading of that
mod's later assemblies; one that loads but whose types fail is skipped
with an error [decomp]. Do not direct a debug build into a subfolder of
`Assemblies/`: the game will try to load it beside the release build, and
whether a second copy of the same assembly loads or is ignored was not
verified.

For development, link the repository into the install's `Mods/` folder (a
directory junction on Windows) so the game loads the working tree. The
local `Mods/` folder is in the install folder, not in the save-data folder
[decomp]; Workshop mods live under Steam's
`steamapps/workshop/content/294100/<id>`.

## Packaging and Publishing

The in-game Workshop uploader passes the mod's whole root folder to
Steam's `SetItemContent`; RimWorld's code applies no exclusions [decomp].
Publishing from a development folder, or from a link to the repository,
would therefore upload sources, `.git`, scratch folders, decompiled code,
and logs, unless Steam filters them, which was not verified. Stage a
clean copy as FieldManual `knacks/games/steam.knack/` (mods and Workshop
companion) describes, holding only what the game loads: `About/` (with
`Preview.png`), `Assemblies/`, `Defs/`, `Patches/`, `Languages/`,
`Sounds/`, `Textures/`, and any `LoadFolders.xml` folders. After the
first upload the game records
the Workshop ID in `About/PublishedFileId.txt` [decomp]; keep that file
with the mod.

## Verifying an API

Use the cheapest source that answers the question:

1. **Ludeon's partial source** in the install's `Source/` folder: about
   forty official C# files, including `CompExplosive.cs` (the countdown
   pattern), `Building_Door.cs`, `Fire.cs`, `Projectile_Explosive.cs`,
   `ThingDef.cs`, `JobDef.cs`, `Toils_General.cs`, and
   `JobDriver_DoBill.cs`. They are primary sources; cite them.
2. **Vanilla defs** in `Data/<Core or DLC>/Defs`. Grep them for field
   names, enum values, and `defName`s. A short script that extracts a
   sorted table (every building's size, stuff, and cost; every work
   giver by priority) answers design questions quickly.
3. **Reflection** over `Assembly-CSharp.dll` for exact fields, types, and
   signatures.
4. **A decompile** for behavior: what a method actually does, in what
   order, and what it logs.

Record which build a decompiled finding came from. It is evidence about
that version, not a contract.

### Decompiling

Install `ilspycmd` pinned, in an ignored folder of the project, as
FieldManual `knacks/dotnet/closed-assembly-inspection.knack/` describes;
the newest release failed to install on a .NET 9 SDK, and 9.1.0.7988
worked [observed]. For RimWorld:

```text
dotnet tool install ilspycmd --version 9.1.0.7988 --tool-path .local/tools
.local/tools/ilspycmd -p -o .local/decomp \
    <install>/RimWorldWin64_Data/Managed/Assembly-CSharp.dll
.local/tools/ilspycmd -t Verse.GenSpawn <same dll>
```

- `-p -o <dir>` writes a whole project (a minute or two; over 9,000
  files for 1.6.4871); `-t <Type>` prints one type.
- Keep the decompiled code out of version control and out of the mod: it
  is Ludeon's code, and the install's `EULA.txt` forbids sharing the
  software or putting a substantial part of its code in a mod.
- Opening the decompiled project in Visual Studio can hang on
  `BurstCompiler` attributes; delete those lines from
  `Properties/AssemblyInfo.cs` (RimWorld Wiki, 1.6 Mod Updates).
- Namespaces rarely match guesses (`DutyDef` is in `Verse.AI`;
  `MinifiedThing` is in `RimWorld`). Find a file before reading it:
  `grep -rl "class DutyDef\b" .local/decomp`.

### Reflection From PowerShell

```powershell
$asm = [Reflection.Assembly]::LoadFrom(
  "$env:RIMWORLD_DIR\RimWorldWin64_Data\Managed\Assembly-CSharp.dll")
$asm.GetType('Verse.ThingDef').GetFields() |
  Select-Object Name, FieldType
```

On 1.6.4871, Windows PowerShell 5.1 threw `ReflectionTypeLoadException`
from `GetTypes()` for a few dozen types, while PowerShell 7 loaded them
all; catch the exception and use its `Types` property, skipping nulls
[observed]. `GetType(name)` works in both. FieldManual
`knacks/dotnet/closed-assembly-inspection.knack/` (its PowerShell
reflection companion) covers the general technique and its traps, among
them `.Name` returning a def's `Name="..."` attribute instead of the
element name; use `.LocalName`.

## Finding Vanilla Assets

- **Base-game textures** are packed into Unity assets, but their paths are
  listed in lowercase in `RimWorldWin64_Data/globalgamemanagers`. A byte
  search finds them, for example
  `grep -aoE "textures/ui/commands/[a-z0-9_]+" globalgamemanagers`
  [observed]. From PowerShell or Python, read the file as Latin-1 (code
  page 28591) so every byte maps to one character.
- **DLC textures** are in asset bundles; their paths appear as text in
  `Data/<DLC>/AssetBundles/*.manifest`, under
  `Assets/Data/<DLC>/Textures/` [observed].
- A texture path in XML omits `Textures/` and the extension. A multi-sided
  graphic uses `_north`, `_east`, and `_south` suffixes; a folder graphic
  (`Graphic_Random`) names the folder.
- `Version.txt` in the install root holds the game version, but its
  revision can lag the binary: it read "1.6.4871 rev590" while the game
  logged "RimWorld 1.6.4871 rev591" at startup [observed]. Record the
  version the game logs.

## Review Checklist

- Does the build take the install path from the environment and fail
  clearly without it?
- Are game and Unity references marked not to copy?
- Was the last build confirmed to succeed before testing?
- Is the decompiler version pinned and the output kept out of the
  repository?
- Is each verified finding labelled with its source and build?

## Further Information

- RimWorld Wiki, [Setting up a solution](https://rimworldwiki.com/wiki/Modding_Tutorials/Setting_up_a_solution)
  and [Recommended software](https://rimworldwiki.com/wiki/Modding_Tutorials/Recommended_software)
- RimWorld Wiki, [Decompiling source code](https://rimworldwiki.com/wiki/Modding_Tutorials/Decompiling_source_code)
- RimWorld Wiki, [RimWorld 1.6 Mod Updates](https://rimworldwiki.com/wiki/Modding_Tutorials/RimWorld_1.6_Mod_Updates)
- ILSpy: <https://github.com/icsharpcode/ILSpy>; `ilspycmd` on NuGet:
  <https://www.nuget.org/packages/ilspycmd/>
- `Krafs.Rimworld.Ref` on NuGet: <https://www.nuget.org/packages/Krafs.Rimworld.Ref>
- Ludeon Studios, `EULA.txt` in the install root: the "Mods" section
- FieldManual `knacks/dotnet/net-framework-builds.knack/`,
  `knacks/dotnet/visual-studio.knack/`,
  `knacks/dotnet/closed-assembly-inspection.knack/`, and
  `knacks/games/steam.knack/`
