# RimWorld Showcase Staging

Load this companion after the
[RimWorld showcases overview](rimworld-showcases.overview.knack.md) when
building the scene a showcase runs in: where each demonstration goes, what
it needs, who must be kept safe, and what the camera and messages show.

**Status.** The current, hand-built approach; revise when the planned
showcase API exists.

Source review: 2026-10-13, against RimWorld 1.6.4871. Evidence marks:
**[decomp]** means verified only by decompiling `Assembly-CSharp.dll` from
1.6.4871; **[observed]** means seen in game testing on 1.6.4871 but not
traced to a published source.

## Preparing the Game

- **Research.** Finish the projects a demonstration needs with
  `Find.ResearchManager.FinishProject(project)`.
- **Dev buttons.** Set `DebugSettings.godMode = true` so dev-only gizmos
  show (they need dev mode too, which the launch preferences enable); a
  presenter can then trigger effects on demand [decomp].
- **Quiet.** Turn off automatic pausing in the preferences and consider
  silencing the storyteller, as the smoke runner does, so a raid does not
  interrupt the tour.
- **Fog.** Clear fog over the areas the camera will visit; unfogging
  fogged rock later sends letters.

## Seeds and Sites

The quick-test map is random every launch; the scene should not be. Its
size is fixed, 250 by 250, with x growing east and z growing north, so
the center is (125, 125) [decomp].

- Generate scenery from a fixed seed, so the same seed always gives the
  same layout and a capture from one run compares with the next.
- Search for each site from a preferred spot outward. Score candidates by
  the work needed to level them (water, rock, roofs) and by distance.
- If no candidate qualifies, widen the search window, then skip the scene
  with a logged reason. One map's hives, geysers, and monolith left no
  valid site in the preferred area [observed].

## One Geometry for Every Safety Rule

Every demonstration has a reach: its main radius, secondary blasts, fire
spread, projectiles, and a margin. Budget the whole reach (see
[Combat and Effects](../rimworld-modding.knack/rimworld-modding.api.combat-and-effects.knack.md)),
then keep demonstrations out of each other's reach and away from the
scenery.

Put the rule in **one** public helper, for example
`Site.Clears(structure, point, reach)`, measured from the cells each
structure actually occupies, and call it from every filter: the site
search, the choice of demonstration pads, and any debug action. When the
site search and the pad filter measured differently (parts versus a
bounding box with its margin), one demonstration was silently dropped on
some maps [observed]. Unit-test the helper and the reach arithmetic in the
RimWorld-free library.

Indestructible features (steam geysers, craters) cannot be cleared.
Leave them in place when they do not touch a structure; they can even add
character.

## Protecting the Cast

- Before each demonstration arms, move colonists outside its reach plus a
  margin. A colonist that wandered within a 70-tile barrage died during an
  early showcase [observed].
- A demonstration that needs a pawn (a volunteer, a thrower) must handle
  that pawn being absent or dead, and the opening message must not promise
  it.
- Keep demonstrations that destroy the scenery last, or rebuild between
  them.

## Props With Side Effects

Scenery is part of a working colony and behaves like it [observed]:

- **Player-owned props** extend the home area, so colonists cross the map
  to fight fires, repair, and clean there. Give scenery to a hidden,
  neutral faction, or remove its cells from the home area.
- **Storage props** with a high priority pull matching items from the
  whole map. Disallow everything in remote storage and fill it by code.
- **Production props** drop unforbidden output that haulers carry off.
  Forbid loose items in the scenery and switch producers off.
- **Joy furniture** that is powered attracts colonists. Leave it off.
- **Player-owned devices** far from the base become raid targets.

See [Buildings and Storage](../rimworld-modding.knack/rimworld-modding.api.buildings-and-storage.knack.md)
for the engine rules behind these.

## Camera and Selection

- `Find.CameraDriver.JumpToCurrentMapLoc(cell)` centers the view;
  `Find.CameraDriver.SetRootSize(size)` zooms. The default range is about
  11 (closest) to 60 (farthest) [decomp].
- Zoom in for small details. A countdown band one cell wide was a few
  pixels at an overview zoom; about 22 made it readable [observed]. Pull
  back for anything with a large radius.
- Frame a reaction shot off-center, so the main subject stays in view.
- Select the subject (`Find.Selector.ClearSelection()`, then
  `Find.Selector.Select(thing)`) so its buttons and inspect text are on
  screen while it is explained.

## Commentary

- Explain with vanilla messages that carry look targets:
  `Messages.Message(text, new LookTargets(thing),
  MessageTypeDefOf.PositiveEvent)`. A click jumps to the subject.
- Write the opening message from the demonstrations that remain after
  staging, not from the plan; log any scene that was skipped and why.
- Keep commentary text in XML, in a def or mod extension, so it can be
  translated and tested like other player-facing text.

## Review Checklist

- Is scenery generated from a fixed seed, and are skipped scenes logged?
- Does one helper decide every clearance, measured from occupied cells?
- Is the cast moved clear before each demonstration?
- Are props owned, switched, and forbidden so they pull nobody?
- Does the camera frame each subject at a readable zoom, with the subject
  selected?
- Does the opening message match what will run?

## Further Information

- [Things and Maps](../rimworld-modding.knack/rimworld-modding.api.things-and-maps.knack.md)
  for clearing ground and indestructible things
- [Pawn Generation and Factions](../rimworld-modding.knack/rimworld-modding.api.pawn-generation-and-factions.knack.md)
  for hidden neutral factions and calm scripted pawns
- RimWorld Wiki, [DebugActions](https://rimworldwiki.com/wiki/Modding_Tutorials/DebugActions)
