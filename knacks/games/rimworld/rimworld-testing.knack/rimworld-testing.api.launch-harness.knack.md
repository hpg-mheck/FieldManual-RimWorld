# RimWorld Launch Harness

Load this companion after the
[RimWorld testing overview](rimworld-testing.overview.knack.md) when
writing the script that starts RimWorld for an automated run, isolates it
from the player's own game, waits for the runner, and turns the log into a
verdict and an exit code.

Source review: 2026-10-13, against RimWorld 1.6.4871. Evidence marks:
**[decomp]** means verified only by decompiling `Assembly-CSharp.dll` from
1.6.4871; **[observed]** means seen in game testing on 1.6.4871 but not
traced to a published source.

## Responsibilities

1. Find the game from an environment variable (`RIMWORLD_DIR`) or a
   parameter. Do not assume the variable is set in every shell, including
   an agent's; a script may fall back to probing the standard Steam
   library locations, but must say which install it chose (and its
   `Version.txt`) and fail clearly when none is found.
2. Refuse to run without a freshly built mod assembly.
3. Make the mod visible to the game: link the repository into the
   install's `Mods/` folder (a directory junction on Windows) once, and
   refuse to replace anything already there that is not that link. Compare
   mods by `packageId`, never by display name, before touching anything.
   Remember that the player's own game then loads whatever branch is
   checked out, half-finished work included; warn before switching
   branches, or point the link at a stable worktree.
4. Prepare an isolated data folder with generated configuration.
5. Launch, wait for the runner's DONE line or a timeout, and close the
   game.
6. Report the runner's PASS and FAIL lines and every logged error, and exit
   nonzero on any failure.

## Isolation

`-savedatafolder=<dir>` moves configuration and saves out of the player's
own folder (by default
`%USERPROFILE%\AppData\LocalLow\Ludeon Studios\RimWorld by Ludeon Studios`
on Windows). The game reads it as `key=value`, so write it with the equals
sign [decomp], and confirms it in the log: "Save data folder overridden to
..." [decomp]. Unity's `Player.log` stays in the default folder unless
`-logFile <path>` moves it too. With both, a test run leaves the player's
mod list, options, saves, and logs untouched, and keeps test saves out of
Steam Cloud, which syncs only the default folder [observed].

A fresh data folder needs only `Config/ModsConfig.xml` and
`Config/Prefs.xml`; the game adds the rest on first launch [observed].
Rewrite both before every run: `resetModsConfigOnCrash` defaults to true,
so a crash can reset the folder's mod list [decomp].

Before launching, check whether the player's own RimWorld is running, and
never close a game window the script did not start.

Two runs can proceed side by side with different data folders, which
samples two random maps at once, and a test can run beside a game a person
is watching [observed]. The Steam build can be started directly from its
executable with these arguments; it attaches to a running Steam client,
and the process it returns is the game's [observed]. General Steam
behavior (libraries, launch options, cloud, Workshop updates) is covered by
FieldManual's stock Steam knack, `knacks/games/steam.knack/`.

The game restarts itself, as a new process with the same arguments, when a
player confirms a mod-list change on the Mods page or in a save's
mod-mismatch dialog [decomp]. A script tracking the first process then sees
an early exit. Seeding `ModsConfig.xml` before launch means an automated
run never needs that page.

A running game keeps the assembly it loaded. After a fix, restart it; an
old instance keeps logging the old errors [observed].

### Generated `ModsConfig.xml`

```xml
<ModsConfigData>
  <version>1.6.4871 rev590</version>
  <activeMods>
    <li>ludeon.rimworld</li>
    <li>ludeon.rimworld.royalty</li>
    <li>ludeon.rimworld.ideology</li>
    <li>ludeon.rimworld.biotech</li>
    <li>ludeon.rimworld.anomaly</li>
    <li>ludeon.rimworld.odyssey</li>
    <li>exampleauthor.brassworks</li>
  </activeMods>
  <knownExpansions>
    <li>ludeon.rimworld.royalty</li>
    <!-- ... each installed DLC ... -->
  </knownExpansions>
</ModsConfigData>
```

- Read the version from `Version.txt` in the install root.
- Discover installed DLCs from `Data/*/About/About.xml` and list them in
  release order (Royalty, Ideology, Biotech, Anomaly, Odyssey); later DLCs
  inherit from earlier ones' defs, and with the order wrong the quick-test
  game failed to start [observed].
- Read the mod's own `packageId` from its `About.xml` and write it
  lowercase, as the game does.
- Run with every DLC active by default, since players do; add a reduced
  configuration when a feature has DLC-specific paths.

### Generated `Prefs.xml`

```xml
<PrefsData>
  <volumeMaster>0</volumeMaster>
  <fullscreen>False</fullscreen>
  <screenWidth>1280</screenWidth>
  <screenHeight>720</screenHeight>
  <uiScale>1</uiScale>
  <runInBackground>True</runInBackground>
  <devMode>True</devMode>
  <pauseOnError>False</pauseOnError>
  <automaticPauseMode>Never</automaticPauseMode>
  <langFolderName>English</langFolderName>
</PrefsData>
```

All of these are fields of the game's `PrefsData` [decomp]. Muted, small,
and running in the background suits unattended runs. Dev mode is
essential: the game checks defs for configuration errors only in dev mode
[decomp], and dev mode also enables dev-only gizmos and debug actions.
With `pauseOnError` and automatic pausing off, the game keeps ticking
through the events a test provokes. Runs meant for a
person use different values (see
[Showcases](../rimworld-showcases.knack/rimworld-showcases.overview.knack.md)).

## Launching

```text
RimWorldWin64.exe -savedatafolder="<dir>" -quicktest -bwsmoketest
    -logFile "<dir>\Player.log"
    -screen-fullscreen 0 -screen-width 1280 -screen-height 720
```

- `-quicktest` skips the menus and generates a game directly: the
  Crashlanded scenario, Cassandra on Rough, a fresh random world and
  starting tile, and a 250 by 250 map [decomp]. Every run therefore sees a
  different biome and terrain.
- `-logFile` and the `-screen-*` options are standard Unity player
  arguments.
- The runner flag is the mod's own; add a "keep open" flag for runs a
  person will inspect afterwards.

## Reading the Result

- Poll the log for the runner's DONE line with a timeout. Allow a grace
  period for the game to quit, then end the process.
- Extract the runner's PASS, FAIL, and NOTE lines.
- `Player.log` records an error as a plain line without a level marker
  [observed]. Match error-shaped lines (`Exception`, `Error`, `XML error`,
  `Config error`, `Could not resolve`, "needs a StaticConstructorOnStartup"),
  skip Unity's stack-trace and file-location lines, and attribute an error
  to the mod when its line **or the stack-trace lines that follow it**
  name the mod's namespace or def prefix. The runner's own scan of
  `Log.Messages` knows each message's level and is the primary check; the
  harness scan is the backstop for runs that never reach the runner and
  for messages the queue dropped.
- Exit nonzero when the DONE line is missing, any check failed (the
  runner's last check fails on any logged error), or the log shows an
  error attributed to the mod.
- Save each run's log under a unique name. Comparing many runs, for
  example counting which check fails on which kind of map, separates bugs
  from map noise.
- A clean exit ends the log with Unity's "Memory Statistics" dump; a log
  that stops earlier means the process crashed or was killed [observed].

### Errors Worth Matching, and Noise

Lines that indicate real problems include "Could not resolve
cross-reference", "Object with load ID ... is referenced ... but is not
deep-saved", "RandomElementByWeight with totalWeight=0", and "Exception
ticking <thing> (at (x, 0, z))" [observed].

Lines that a naive pattern also catches but that are routine on 1.6.4871
include "Fallback handler could not load library" for Mono's own
libraries, "Failed Allocations" in the exit memory dump, a
`ThreadAbortException` during shutdown, map generation's "could not find
cell" for cave debris, and a translation-data error count for English
[observed]. List such exceptions explicitly, by exact text, so new errors
are never hidden by a broad filter.

## Random Maps and Flakes

A quick-test map can contain hives, ruins, geysers, mountains, water, and
hostile insects. When something fails on one map only:

1. read the failure detail and the NOTE lines from that run;
2. decide whether the map made the scenario impossible (no valid site, a
   creature wandering into the test, a raid) or exposed a real bug;
3. for map noise, harden the scenario: search wider for a site and then
   skip with a logged reason, quiet the map, count every sink;
4. for a real bug, add a check that fails before the fix;
5. if it stays unexplained, make the check report more and record the
   flake in the project's known-issues list.

A site search that found nothing on one map in thirty runs (insect hives
filled the preferred area) was working as designed; the runner now widens
the search before skipping [observed].

## Review Checklist

- Does the harness find the game without hard-coded paths?
- Is every run isolated in its own data folder, with generated
  configuration?
- Are DLCs listed in release order?
- Does the verdict depend on the DONE line, every FAIL, and every mod
  error, including those found through stack traces?
- Are run logs kept for comparison?

## Further Information

- RimWorld Wiki, [Testing mods](https://rimworldwiki.com/wiki/Modding_Tutorials/Testing_mods)
  (`-quicktest` and `-savedatafolder`)
- Unity 2022.3 manual, Player command-line arguments:
  <https://docs.unity3d.com/2022.3/Documentation/Manual/PlayerCommandLineArguments.html>
- Unity 2022.3 manual, Log files:
  <https://docs.unity3d.com/2022.3/Documentation/Manual/LogFiles.html>
- Each DLC's `Data/<DLC>/About/About.xml` (`forceLoadAfter`) for the
  release order
