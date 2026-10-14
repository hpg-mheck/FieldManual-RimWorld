# RimWorld Knacks

Reviewed: 2026-10-14, against RimWorld 1.6.4871 with all five DLCs.

These knacks cover building, testing, and demonstrating RimWorld 1.6 mods:
XML definitions, C# against the game's assembly, XML patches, in-game test
runners, and staged showcases. They target the current release only; they
do not describe 1.5 or earlier, and the 1.7 beta was not reviewed.

## Modding

- [Overview](modding/overview.knack.md) —
  mod layout, identity and load order, defs and DefOfs, patches before
  Harmony, DLC-optional content, building, and verifying APIs
- [Defs and XML](modding/defs-and-xml.knack.md)
  — the loader's order, fields, inheritance, `Class=`, configuration
  errors, mod extensions, translation, patches
- [Lifecycle and Ticking](modding/lifecycle-and-ticking.knack.md)
  — ticker types and the 1.6 variable tick rate, comps, held things,
  components, static assets, saving, randomness
- [Things and Maps](modding/things-and-maps.knack.md)
  — making and spawning things, footprints, indestructible things,
  clearing ground, roofs, terrain
- [Buildings and Storage](modding/buildings-and-storage.knack.md)
  — power, ownership side effects, doors and forbidding, zones and
  stockpiles, storage buildings, autonomous bills
- [Pawns and AI](modding/pawns-and-ai.knack.md)
  — main and constant think trees, job givers, lords and duties, fleeing
- [Pawn Generation and Factions](modding/pawn-generation-and-factions.knack.md)
  — generation requests, pawn kinds, hidden factions, needs and breaks,
  custom races
- [Combat and Effects](modding/combat-and-effects.knack.md)
  — explosions and fire, projectiles and launchers, forced misses, radial
  limits, damage hooks
- [UI and Debug](modding/ui-and-debug.knack.md)
  — gizmos, the inspect pane, messages, map drawing and GUI, debug actions

## Testing

- [Overview](testing/overview.knack.md) —
  the layered strategy, proving checks can fail, the error policy
- [Static Tests](testing/static-tests.knack.md)
  — a RimWorld-free logic library, XML regression tests, tests against the
  game's own data
- [Smoke Runner](testing/smoke-runner.knack.md)
  — an in-game runner: synchronous ticking, checks, the log queue, staging,
  seams, lost fixtures
- [Launch Harness](testing/launch-harness.knack.md)
  — isolated launches, generated configuration, reading the verdict,
  random maps and flakes

## Tooling

- [Overview](tooling/overview.knack.md) —
  building against the game, packaging and publishing, decompiling,
  reflection, finding vanilla assets
- [Def Validation](tooling/def-validation.knack.md)
  — a reflection-based stand-in for the XML loader, patches included

## Showcases

- [Overview](showcases/overview.knack.md)
  — a showcase mode of the runner, from timeline to directed tour
- [Staging](showcases/staging.knack.md)
  — seeds, sites and shared safety geometry, the cast, props, camera,
  commentary
- [Capture and Cues](showcases/capture-and-cues.knack.md)
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
| Modding | `knacks/dotnet/framework-builds/` |
| Testing | `knacks/software-engineering/testing/in-host-smoke/`, `knacks/software-engineering/testing/proving-checks/`, `knacks/dotnet/shared-library/`, `knacks/games/steam/`, `knacks/windows/processes-and-files/` |
| Tooling | `knacks/dotnet/framework-builds/`, `knacks/dotnet/visual-studio/`, `knacks/dotnet/assembly-inspection/`, `knacks/windows/shells/`, `knacks/games/steam/` |
| Showcases | `knacks/software-engineering/demonstrations/showcases/`, `knacks/windows/desktop-automation/` |

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
