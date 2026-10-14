# RimWorld Mod Tooling

Load this knack when setting up a RimWorld 1.6 mod's build, or when an API,
field name, def name, or texture path must be verified instead of guessed.
It covers compiling against the game, decompiling it, querying it by
reflection, and finding vanilla assets.

Source review: 2026-10-13, against RimWorld 1.6.4871. Evidence marks:
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
  <LangVersion>latest</LangVersion>
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

- Take the install location from an environment variable, never a
  hard-coded path, and fail the build with a message when it is missing.
- Add Unity modules only as needed: `UnityEngine.IMGUIModule` for `GUI`
  and `Event`, `UnityEngine.TextRenderingModule` for `TextAnchor`
  [decomp]. Each with `Private` false, so nothing is copied into
  `Assemblies/`.
- Do not ship Harmony or other libraries' DLLs; depend on their mods.
- Community reference-assembly packages on NuGet (for example
  `Krafs.Rimworld.Ref`) remove the need for the game on a build machine;
  pin one that matches the build you test against.
- Link `About/`, `Defs/`, `Patches/`, `Textures/`, and `Languages/` into
  the project as non-compiled items so the whole mod is browsable in the
  IDE.
- Keep `Assemblies/`, `obj/`, and `bin/` out of version control unless
  the release process needs the built DLL committed.

A build you did not see succeed may not have happened. If a compile error
leaves the old DLL in `Assemblies/`, every later test runs old code.
Check the build's exit code, or compare the DLL's timestamp with the
source's, before trusting a test run [observed].

The game loads every `.dll` under a mod's `Assemblies/` folder,
subfolders included, and stops loading that mod's assemblies at the first
one that fails [decomp]. Do not direct a debug build into a subfolder of
`Assemblies/`: the game will try to load it beside the release build, and
whether a second copy of the same assembly loads or is ignored was not
verified.

For development, link the repository into the install's `Mods/` folder (a
directory junction on Windows) so the game loads the working tree. The
local `Mods/` folder is in the install folder, not in the save-data folder
[decomp]; Workshop mods live under Steam's
`steamapps/workshop/content/294100/<id>`.

## Packaging and Publishing

The in-game Workshop uploader hands Steam the mod's whole root folder;
RimWorld's code applies no exclusions [decomp]. Publishing from a
development folder, or from a link to the repository, would therefore
upload sources, `.git`, scratch folders, decompiled code, and logs, unless
Steam filters them, which was not verified. Publish from a staged copy
holding only what the game loads: `About/` (with `Preview.png`),
`Assemblies/`, `Defs/`, `Patches/`, `Languages/`, `Sounds/`, `Textures/`,
and any `LoadFolders.xml` folders. After the first upload the game records
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

ILSpy's command-line tool installs as a .NET tool. Pin its version in a
tool path inside an ignored folder of the project:

```text
dotnet tool install ilspycmd --version 9.1.0.7988 --tool-path .local/tools
.local/tools/ilspycmd -p -o .local/decomp \
    <install>/RimWorldWin64_Data/Managed/Assembly-CSharp.dll
.local/tools/ilspycmd -t Verse.GenSpawn <same dll>
```

- `-p -o <dir>` writes a whole project (a few minutes; about 9,000
  files for 1.6); `-t <Type>` prints one type.
- The newest `ilspycmd` may require a newer .NET SDK than the machine
  has; in practice the latest failed to install on a .NET 9 SDK and
  9.1.0.7988 worked [observed]. Pin a version that installs, and record
  it.
- Keep the decompiled code out of version control and out of the mod: it
  is Ludeon's code.
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
[observed]. `GetType(name)` works in both.

PowerShell's XML adapter has a trap for def files: on an element that has
a `Name="..."` attribute, `.Name` returns the attribute, not the element
name. Use `.LocalName`.

## Finding Vanilla Assets

- **Base-game textures** are packed into Unity assets, but their paths are
  listed in lowercase in `RimWorldWin64_Data/globalgamemanagers`. A byte
  search finds them, for example
  `grep -aoE "textures/ui/commands/[a-z0-9_]+" globalgamemanagers`
  [observed].
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
- Microsoft, .NET tools: <https://learn.microsoft.com/en-us/dotnet/core/tools/global-tools>
- `Krafs.Rimworld.Ref` on NuGet: <https://www.nuget.org/packages/Krafs.Rimworld.Ref>
