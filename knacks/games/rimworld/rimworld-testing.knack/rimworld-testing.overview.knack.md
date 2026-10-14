# RimWorld Mod Testing

Load this knack when deciding how to verify a RimWorld 1.6 mod, adding a
feature that needs checks, or chasing a failure that appears only in game.
It describes a layered strategy, from seconds-long unit tests to an
automated in-game run.

This fileset maps general practice onto RimWorld. Load the general
knacks for the pattern itself: FieldManual
`knacks/software-engineering/testing/in-host-smoke-testing.knack/` (the
layers, the in-host runner, the launcher, flake control) and
`knacks/AI/agent-orchestration/proving-checks-can-fail.knack/` (sabotage
runs and vacuous checks).

Reviewed: 2026-10-14, against RimWorld 1.6.4871. Evidence marks:
**[decomp]** means verified only by decompiling `Assembly-CSharp.dll` from
1.6.4871; **[observed]** means seen in game testing on 1.6.4871 but not
traced to a published source.

## Companions

- [Static Tests](rimworld-testing.api.static-tests.knack.md) — a
  RimWorld-free logic library, XML regression tests, and tests against the
  game's own data
- [Smoke Runner](rimworld-testing.api.smoke-runner.knack.md) — an in-game
  test runner: structure, synchronous ticking, checks, error capture,
  staging, and flake control
- [Launch Harness](rimworld-testing.api.launch-harness.knack.md) — the
  script that starts the game isolated, waits, and reports

Related: [def validation](../rimworld-tooling.knack/rimworld-tooling.api.def-validation.knack.md)
in the tooling fileset, and [showcases](../rimworld-showcases.knack/rimworld-showcases.overview.knack.md),
which reuse the runner to stage demonstrations for a person.

## Mental Model

Much of what can go wrong in a RimWorld mod is invisible until the game
runs it:

- XML that names a missing field logs an error and skips the element, and
  a missing class drops the whole def [decomp];
- configuration errors are computed only after every def has loaded, and
  only when dev mode is on [decomp];
- behavior depends on the map, the active DLCs, other mods, and dozens of
  vanilla systems acting at once;
- the log is the only failure channel, and much of it is noise.

No single layer catches all of that cheaply, so use several, fastest
first:

1. **Unit tests of pure logic** — no game needed; seconds. They catch
   arithmetic, geometry, and planning mistakes.
2. **XML regression tests** — the game's `Data` folder optional;
   seconds. They catch broken house rules, dangling references, and
   misplaced patches.
3. **Def validation by reflection** — the game's assemblies; under a
   minute. It catches misspelled fields, bad enum values, missing classes
   and textures, and XPaths that select nothing.
4. **An in-game smoke run** — the game itself; about a minute per run
   (35 to 75 seconds on one machine) [observed]. It catches
   configuration errors, runtime exceptions, and wrong behavior.
5. **A showcase or manual play** — the game and a person. It catches what
   checks cannot judge: visuals, feel, and balance.

## Workflow

1. Keep logic that does not need the engine in a library without a
   RimWorld reference, and unit-test it.
2. Pin data-shaped rules in XML tests, including rules learned from
   failures.
3. Validate the defs and patches against the game assembly before each
   launch.
4. Run the smoke runner on a fresh quick-test map, then run it again: the
   map is random, and a pass on one map is a sample of one.
5. Watch a showcase, or play, for what checks cannot judge.
6. Record deviations and flakes with the evidence that explains them.

Apply the stock rule of two consecutive clean runs before calling an
in-game change green, and keep every run's output so failures can be
compared across maps. A random quick-test map makes the rule matter more
here than in most hosts.

## Prove Each Check Can Fail

Before trusting a new in-game check, see it fail against code that breaks
the behavior, as FieldManual
`knacks/AI/agent-orchestration/proving-checks-can-fail.knack/` describes.
In RimWorld runs, sabotage found these vacuous passes [observed]:

- a fixture destroyed by an earlier destructive test, accepted by a null
  guard (every colonist had died in earlier blasts);
- a tautology, comparing a property with a list computed from the same
  property;
- an observation window that includes unrelated later events, such as a
  final explosion sending a pawn fleeing;
- a containment box so large that only death could fail it.

A sabotage edit that fails to compile leaves the previous assembly in
`Assemblies/`, and the run then "passes" against old code. Check the
build's exit code, or the DLL's timestamp, before reading the result
[observed].

## Error Policy

Treat every error the game logs during a run as a failure, whether or not
it names the mod: an exception can be thrown by vanilla code called with
the mod's bad data, and its first line may name only a vanilla type
("Exception ticking MinifiedThing123") while its stack trace runs through
the mod [observed]. Fail as well on warnings that name the mod or
statically held assets. Read errors from the game's own log queue rather
than only from `Player.log`; see the Smoke Runner companion.

Other mods matter too. Harmony appends the patches on a method to
exception stack traces, and one player's log showed another mod's postfix
throwing on a mod's building [observed]. Run the mod at least once with a
realistic mod list as well as with vanilla and the DLCs.

## Sharp Edges

- The quick-test map is random: biome, terrain, hives, geysers, and
  ruins change every run. A site, placement, or path that works on one map
  can fail on the next.
- The storyteller keeps running during a test; a raid can land mid-check.
- Colonists, haulers, and animals act on their own while a check waits.
- Some engine work happens per frame, not per tick, and does not happen
  when a runner ticks synchronously.
- Lazily resolved state (a roof-collapse buffer, dirty rooms) changes the
  moment a tick runs.

## Review Checklist

- Does each new feature add checks at the cheapest layer that can catch
  its failures?
- Was each new in-game check seen to fail against broken code?
- Did the build that the run tested actually compile from the current
  source?
- Did the run fail on any logged error, including vanilla-looking ones?
- Did the change pass on more than one random map?

## Further Information

- RimWorld Wiki, [Testing mods](https://rimworldwiki.com/wiki/Modding_Tutorials/Testing_mods)
  (`-quicktest`, `-savedatafolder`)
- RimWorld Wiki, [Troubleshooting](https://rimworldwiki.com/wiki/Modding_Tutorials/Troubleshooting)
  and [Finding Exceptions](https://rimworldwiki.com/wiki/Modding_Tutorials/Troubleshooting/Finding_Exceptions)
- RimWorld Wiki, [ConfigErrors](https://rimworldwiki.com/wiki/Modding_Tutorials/ConfigErrors)
- FieldManual
  `knacks/software-engineering/testing/in-host-smoke-testing.knack/` and
  `knacks/AI/agent-orchestration/proving-checks-can-fail.knack/`, in a
  FieldManual checkout:
  <https://github.com/Hard-Problems-Group-LLC/FieldManual>
