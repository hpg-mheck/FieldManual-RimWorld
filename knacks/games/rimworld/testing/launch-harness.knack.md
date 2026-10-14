# RimWorld Launch Harness

Load this companion after the
[RimWorld testing overview](overview.knack.md) when
writing the script that starts RimWorld for an automated run, isolates it
from the player's own game, waits for the runner, and turns the log into a
verdict and an exit code.

The launcher's general duties (preflight, isolation, configuration,
installing the plugin, waiting, scraping, artifacts) are in FieldManual
`knacks/software-engineering/testing/in-host-smoke/`
(launcher companion). Steam's side, such as libraries, direct launches,
cloud sync, and the player's own instance, is in FieldManual
`knacks/games/steam/`, and links and process ownership on Windows
in `knacks/windows/processes-and-files/`. This companion fills in
RimWorld's values.

Reviewed: 2026-10-14, against RimWorld 1.6.4871. Evidence marks:
**[decomp]** means verified only by decompiling `Assembly-CSharp.dll` from
1.6.4871; **[observed]** means seen in game testing on 1.6.4871 but not
traced to a published source.

## RimWorld's Values for the Stock Steps

- **Find the game** from `RIMWORLD_DIR` or a parameter. The variable is
  often unset in an agent's shell; a fallback may probe the Steam
  libraries for app 294100, but must report which install it chose and
  its `Version.txt`.
- **Preflight**: refuse a mod DLL in `Assemblies/` older than its
  sources, and note any `RimWorldWin64` process already running, which
  is the player's and must be left alone.
- **Install the mod** by linking the repository into the install's
  `Mods/` folder once. Compare mods by `packageId`, never by display
  name, before touching anything there. The player's own game then loads
  whatever branch is checked out, half-finished work included; warn
  before switching branches, or point the link at a stable worktree.
- **Configure** an isolated data folder with a generated
  `ModsConfig.xml` and `Prefs.xml` (below).
- **Wait** for the runner's DONE line or a timeout, then close only the
  game the script started.

## Isolation

`-savedatafolder=<dir>` moves configuration and saves out of the player's
own folder (by default
`%USERPROFILE%\AppData\LocalLow\Ludeon Studios\RimWorld by Ludeon Studios`
on Windows). The game reads it as `key=value`, split at every `=`, so
write it with the equals sign and keep `=` out of the path [decomp]; it
confirms the folder in the log: "Save data folder overridden to ..."
[decomp]. Have the harness compare that line with the folder it created.
Unity's `Player.log` stays in the default folder unless `-logFile <path>`
moves it too. With both, a test run leaves the player's mod list,
options, saves, and logs untouched, and keeps test saves out of Steam
Cloud, which syncs only the default folder [observed].

A fresh data folder needs only `Config/ModsConfig.xml` and
`Config/Prefs.xml`; the game adds the rest on first launch [observed].
Rewrite both before every run: `resetModsConfigOnCrash` defaults to true,
so a crash can reset the folder's mod list [decomp].

Two runs can proceed side by side with different data folders, which
samples two random maps at once, and a test can run beside a game a person
is watching [observed]. The Steam build can be started directly from
`RimWorldWin64.exe` with these arguments; it attaches to a running Steam
client, and the process it returns is the game's [observed].

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
[Showcases](../showcases/overview.knack.md)).

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

The stock launcher companion covers polling for DONE, attributing errors
by stack trace, exit codes, and keeping artifacts. RimWorld's specifics:

- `Player.log` records an error as a plain line without a level marker
  [observed]. Match error-shaped lines (`Exception`, `Error`, `XML error`,
  `Config error`, `Could not resolve`, "needs a StaticConstructorOnStartup"),
  skip Unity's stack-trace and file-location lines, and attribute an error
  to the mod when its line **or the stack-trace lines that follow it**
  name the mod's namespace or def prefix. The runner's own scan of
  `Log.Messages` knows each message's level and is the primary check; the
  harness scan is the backstop for runs that never reach the runner and
  for messages the queue dropped.
- The runner's last check fails on any logged error, so a run with no
  FAIL line also had a clean log queue.
- Keep every run's log: counting which check fails on which kind of map
  separates bugs from map noise.
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

Triage a failure seen on one map only as the stock flakiness companion
describes. On RimWorld, first ask whether the map made the scenario
impossible: a quick-test map can hold hives, hostile insects, ruins,
geysers, a monolith, mountains, and water, and a raid or a wandering
animal can walk into a check. For map noise, harden the scenario (search
wider for a site and then skip with a logged reason, quiet the map, count
every sink); for a real bug, add a check that fails before the fix.

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
- FieldManual
  `knacks/software-engineering/testing/in-host-smoke/`
  (launcher and flakiness companions), `knacks/games/steam/`, and
  `knacks/windows/processes-and-files/`
