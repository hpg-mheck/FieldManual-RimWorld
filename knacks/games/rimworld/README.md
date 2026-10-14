# RimWorld Knacks

Reviewed: 2026-10-14, against RimWorld 1.6.4871 with all five DLCs.

These knacks cover building, testing, and demonstrating RimWorld 1.6 mods:
XML definitions, C# against the game's assembly, XML patches, in-game test
runners, and staged showcases. They target the current release only; they
do not describe 1.5 or earlier, and the 1.7 beta was not reviewed.

## Modding

- [Overview](rimworld-modding.knack/rimworld-modding.overview.knack.md) —
  mod layout, identity and load order, defs and DefOfs, patches before
  Harmony, DLC-optional content, building, and verifying APIs
- [Defs and XML](rimworld-modding.knack/rimworld-modding.api.defs-and-xml.knack.md)
  — the loader's order, fields, inheritance, `Class=`, configuration
  errors, mod extensions, translation, patches
- [Lifecycle and Ticking](rimworld-modding.knack/rimworld-modding.api.lifecycle-and-ticking.knack.md)
  — ticker types and the 1.6 variable tick rate, comps, held things,
  components, static assets, saving, randomness
- [Things and Maps](rimworld-modding.knack/rimworld-modding.api.things-and-maps.knack.md)
  — making and spawning things, footprints, indestructible things,
  clearing ground, roofs, terrain
- [Buildings and Storage](rimworld-modding.knack/rimworld-modding.api.buildings-and-storage.knack.md)
  — power, ownership side effects, doors and forbidding, zones and
  stockpiles, storage buildings, autonomous bills
- [Pawns and AI](rimworld-modding.knack/rimworld-modding.api.pawns-and-ai.knack.md)
  — main and constant think trees, job givers, lords and duties, fleeing
- [Pawn Generation and Factions](rimworld-modding.knack/rimworld-modding.api.pawn-generation-and-factions.knack.md)
  — generation requests, pawn kinds, hidden factions, needs and breaks,
  custom races
- [Combat and Effects](rimworld-modding.knack/rimworld-modding.api.combat-and-effects.knack.md)
  — explosions and fire, projectiles and launchers, forced misses, radial
  limits, damage hooks
- [UI and Debug](rimworld-modding.knack/rimworld-modding.api.ui-and-debug.knack.md)
  — gizmos, the inspect pane, messages, map drawing and GUI, debug actions

## Testing

- [Overview](rimworld-testing.knack/rimworld-testing.overview.knack.md) —
  the layered strategy, proving checks can fail, the error policy
- [Static Tests](rimworld-testing.knack/rimworld-testing.api.static-tests.knack.md)
  — a RimWorld-free logic library, XML regression tests, tests against the
  game's own data
- [Smoke Runner](rimworld-testing.knack/rimworld-testing.api.smoke-runner.knack.md)
  — an in-game runner: synchronous ticking, checks, the log queue, staging,
  seams, lost fixtures
- [Launch Harness](rimworld-testing.knack/rimworld-testing.api.launch-harness.knack.md)
  — isolated launches, generated configuration, reading the verdict,
  random maps and flakes

## Tooling

- [Overview](rimworld-tooling.knack/rimworld-tooling.overview.knack.md) —
  building against the game, packaging and publishing, decompiling,
  reflection, finding vanilla assets
- [Def Validation](rimworld-tooling.knack/rimworld-tooling.api.def-validation.knack.md)
  — a reflection-based stand-in for the XML loader, patches included

## Showcases

- [Overview](rimworld-showcases.knack/rimworld-showcases.overview.knack.md)
  — a showcase mode of the runner, from timeline to directed tour
- [Staging](rimworld-showcases.knack/rimworld-showcases.api.staging.knack.md)
  — seeds, sites and shared safety geometry, the cast, props, camera,
  commentary
- [Capture and Cues](rimworld-showcases.knack/rimworld-showcases.api.capture-and-cues.knack.md)
  — viewing preferences, maximizing, log cues, screen capture

The showcase fileset documents the current hand-built approach: a showcase
mode of the mod's own test runner, started from the command line. It will
be revised when the planned showcase API exists, which adds a Showcases
button to the main menu listing mods, their showcases, and each one's
details, with warnings about mods flagged as interfering.

## Stock Knacks This Pack Builds On

These knacks keep only what RimWorld adds. Load the general knack first
when the work goes beyond RimWorld's specifics. Paths are inside a
FieldManual checkout, written as FieldManual `knacks/<path>`:

| Fileset | Stock knacks |
|---|---|
| Modding | `knacks/dotnet/net-framework-builds.knack/` |
| Testing | `knacks/software-engineering/testing/in-host-smoke-testing.knack/`, `knacks/AI/agent-orchestration/proving-checks-can-fail.knack/`, `knacks/dotnet/netstandard-shared-library.knack/`, `knacks/games/steam.knack/`, `knacks/windows/processes-and-files.knack/` |
| Tooling | `knacks/dotnet/net-framework-builds.knack/`, `knacks/dotnet/visual-studio.knack/`, `knacks/dotnet/closed-assembly-inspection.knack/`, `knacks/windows/agent-shells.knack/`, `knacks/games/steam.knack/` |
| Showcases | `knacks/software-engineering/demonstrations/directed-showcases.knack/`, `knacks/windows/desktop-automation.knack/` |

## Load Order

For any mod work, start with the modding overview, then load the API
companion for the subsystem in hand. Load the testing overview before
adding a feature, since every feature should add checks. Load tooling when
setting up a build or verifying an API, and showcases only when staging a
demonstration.

## Evidence Marks

Claims are supported by the sources each knack lists: Ludeon's
`ModUpdating.txt` and published primers, the partial C# that ships in the
install's `Source/` folder, the game's `Data` XML, the RimWorld Wiki, and
the Harmony, Unity, and Microsoft documentation. Two marks qualify the
rest:

- **[decomp]** — verified only by decompiling `Assembly-CSharp.dll` from
  RimWorld 1.6.4871. Re-check after a game update.
- **[observed]** — seen in game testing or tool use on 1.6.4871, not
  traced to a published source.

Unmarked claims rest on those sources, or are design advice that follows
from marked facts.

Examples use a fictional mod, "Brass Works" (`ExampleAuthor.BrassWorks`,
prefix `BW_`).

## Maintenance

Re-review this area when RimWorld updates: confirm the build in
`Version.txt`, re-run the decompile checks behind every **[decomp]** claim,
and update the review date. Ludeon's change notes for each release are
linked from the RimWorld Wiki's mod-update pages.
