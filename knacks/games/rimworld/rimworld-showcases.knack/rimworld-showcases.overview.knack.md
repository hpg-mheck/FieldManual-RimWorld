# RimWorld Showcases

Load this knack when staging an in-game demonstration for a person to
watch: a tour of a mod's features, a review of visuals, footage for a
release, or a way for an agent to see what it built.

**Status.** This describes the current, hand-built approach: a showcase
mode of the mod's in-game test runner, driven by a tick-based timeline. A
dedicated showcase API is planned; revise this fileset when it exists.

Source review: 2026-10-13, against RimWorld 1.6.4871. Evidence marks:
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

A timer-driven sequence, in which demonstrations fire at fixed ticks and
the camera moves now and then, shows that features work. It was not
enough for the person watching, who asked for a directed tour instead
[observed]:

- one feature at a time, each framed by the camera and introduced in
  words before it runs;
- each scene waits for its action to finish (an aftermath ending, a pull
  completing, smoke settling) rather than for a fixed delay;
- captions that orient the viewer ("2 of 8: The Steam Press");
- the subject selected, so its buttons and inspect text show;
- a way to skip the tour and keep the running scene;
- optionally, a story that ties the scenes together.

A director is a small state machine over scenes. Each scene sets up,
frames, explains, runs, and reports done; the director logs
`SCENE <name>` as each begins. Build toward it even if the first version
is a timeline: keep each beat self-contained and event-driven.

## Sharp Edges

- A demonstration can vanish without a trace when two places apply the
  same safety rule with different geometry. One filter measured from a
  structure's parts, the other from its bounding box, and a demo was
  dropped with nothing logged [observed].
- Props have gameplay side effects: they pull colonists, haulers, and
  raiders across the map (see Staging).
- The opening message must describe what will actually run. A demo
  dropped at build time, announced anyway, misleads the viewer.
- In dev mode any error opens the log window over the scene; routine
  staging code must log no errors.
- Automatic pausing on threats freezes an unattended demonstration.
- Heavy effects slow the game below real time, so wall-clock timing
  drifts; outside tools must wait on log cues.
- Visuals nobody looked at are unverified. Agents that built sprites,
  overlays, and camera framing had seen only rendered previews until
  captures were taken at each cue [observed].
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
- RimWorld Wiki, [Testing mods](https://rimworldwiki.com/wiki/Modding_Tutorials/Testing_mods)
- RimWorld Wiki, [GameComponent](https://rimworldwiki.com/wiki/Modding_Tutorials/GameComponent)
