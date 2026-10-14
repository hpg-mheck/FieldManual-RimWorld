# RimWorld Showcases

Load this knack when staging an in-game demonstration for a person to
watch: a tour of a mod's features, a review of visuals, footage for a
release, or a way for an agent to see what it built.

**Status.** This fileset documents the current, hand-built approach: each
mod adds a showcase mode to its own in-game test runner, started by a
command-line flag and driven by a tick-based timeline or a small
director. A showcase API is planned that adds a **Showcases** button to
the main menu, listing the mods that offer showcases, each mod's
showcases, and each showcase's details, with warnings when a loaded mod is
flagged as interfering. Revise this fileset when that API exists; until
then, treat what follows as the way to build one by hand.

General showcase practice (the director, staging, viewing settings,
capture, and the shape of a showcase framework) is in FieldManual
`knacks/software-engineering/demonstrations/directed-showcases.knack/`.
This fileset covers what RimWorld adds.

Reviewed: 2026-10-14, against RimWorld 1.6.4871. Evidence marks:
**[decomp]** means verified only by decompiling `Assembly-CSharp.dll` from
1.6.4871; **[observed]** means seen in game testing on 1.6.4871 but not
traced to a published source.

## Companions

- [Staging](rimworld-showcases.api.staging.knack.md) — seeds, sites and
  hazard geometry, protecting the cast, props with side effects, camera,
  commentary
- [Capture and Cues](rimworld-showcases.api.capture-and-cues.knack.md) —
  launching for a viewer, log cues, window handling, screen capture

Showcases reuse the [smoke runner](../rimworld-testing.knack/rimworld-testing.api.smoke-runner.knack.md)
and the [launch harness](../rimworld-testing.knack/rimworld-testing.api.launch-harness.knack.md).

## Mental Model

A showcase is a second mode of the in-game runner. The same component,
under a different command-line flag, builds a working scene on a
quick-test map and leaves the game running instead of testing and
quitting. It differs from a smoke run in purpose and therefore in almost
every setting:

| | Smoke run | Showcase |
|---|---|---|
| Audience | a script | a person |
| Outcome | PASS and FAIL lines, then quit | a running scene |
| Destruction | everything, last | staged, survivable, explained |
| Window | small, muted, 1x UI | maximized, sound, 2x UI |
| Timing | synchronous ticks | real time, at the viewer's speed |

A showcase has six parts: staging (build the scene), a timeline or
director (what happens when), the camera, commentary (messages and
captions), log cues for outside tools, and capture.

## Workflow

1. List the scenes and, for each, its reach: blast radius, secondary
   effects, fire, projectiles, and a margin.
2. Choose sites with one shared geometry helper, so every filter agrees
   on what "clear" means.
3. Build the scene from a fixed seed where it is generated, and finish
   any research the demonstration needs.
4. Log `READY`, then run the timeline, logging one cue per beat.
5. Capture the screen at the cues and look at the frames.
6. Watch the whole showcase yourself before showing it to anyone.

## From Timeline to Directed Tour

A first showcase is often a timeline: demonstrations fire at fixed ticks
and the camera moves now and then. The stock knack explains why watchers
want a directed tour instead, one scene at a time, each waiting for its
action to finish; a person watching a RimWorld showcase asked for exactly
that [observed]. In RimWorld terms:

- a scene frames its subject with the camera, explains it with a vanilla
  message, and selects it so its gizmos and inspect text show (see
  [Staging](rimworld-showcases.api.staging.knack.md));
- "finished" is a game-tick condition: an aftermath ending, a pull
  completing, smoke settling;
- the director is a small state machine driven from
  `GameComponentTick`, logging `SCENE <name>` as each scene begins.

Build toward it even if the first version is a timeline: keep each beat
self-contained and event-driven.

## Sharp Edges

The stock knack's sharp edges all apply: silent drops, fixed delays,
props with side effects, error windows, and agents that never see the
screen. RimWorld's forms of them:

- A demonstration vanished without a trace when the site search measured
  a safety rule from a structure's parts and the pad filter measured it
  from the bounding box [observed].
- Player-owned props pull colonists, haulers, and raiders across the map
  (see Staging).
- In dev mode any error opens the log window over the scene, so routine
  staging code must log no errors; clearing code that tried to destroy a
  steam geyser covered one demonstration that way [observed].
- RimWorld's automatic pausing on threats freezes an unattended
  demonstration; set `automaticPauseMode` to `Never`.
- Heavy effects slow the game below real time, so wall-clock timing
  drifts; outside tools must wait on log cues.
- The showcase finds bugs the tests miss, because it runs longer, on more
  of the map, with more going on [observed].

## Review Checklist

- Does the showcase run under its own flag and preferences, separate from
  the smoke run?
- Do all site and safety filters share one geometry helper?
- Is every skipped demonstration logged and left out of the opening
  message?
- Does each scene frame its subject and wait for its action to end?
- Were captures taken at every cue and inspected?

## Further Information

- [Smoke Runner](../rimworld-testing.knack/rimworld-testing.api.smoke-runner.knack.md)
  and [Launch Harness](../rimworld-testing.knack/rimworld-testing.api.launch-harness.knack.md)
- FieldManual
  `knacks/software-engineering/demonstrations/directed-showcases.knack/`,
  including its framework companion, which outlines the menu-driven
  shape the planned API takes
- [RimWorld modding overview](../rimworld-modding.knack/rimworld-modding.overview.knack.md),
  "Changing Vanilla": the main menu has no extension point, so a menu
  button needs a small Harmony patch
- RimWorld Wiki, [Testing mods](https://rimworldwiki.com/wiki/Modding_Tutorials/Testing_mods)
- RimWorld Wiki, [GameComponent](https://rimworldwiki.com/wiki/Modding_Tutorials/GameComponent)
