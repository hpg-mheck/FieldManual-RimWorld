# RimWorld UI and Debug

Load this companion after the
[RimWorld modding overview](rimworld-modding.overview.knack.md) when adding
buttons (gizmos), inspect-pane text, messages, labels or overlays on the
map, or developer tools.

Reviewed: 2026-10-14, against RimWorld 1.6.4871. Evidence marks:
**[decomp]** means verified only by decompiling `Assembly-CSharp.dll` from
1.6.4871; **[observed]** means seen in game testing on 1.6.4871 but not
traced to a published source.

## Gizmos

Buttons for a selected thing come from `GetGizmos()` on the thing and
`CompGetGizmosExtra()` on its comps. Yield the base gizmos first, and show
operating buttons only to the owner:

```csharp
public override IEnumerable<Gizmo> CompGetGizmosExtra()
{
    foreach (Gizmo g in base.CompGetGizmosExtra())
    {
        yield return g;
    }
    if (parent.Faction != Faction.OfPlayer)
    {
        yield break;
    }
    var vent = new Command_Action
    {
        defaultLabel = "BW_Vent".Translate(),
        defaultDesc = "BW_VentDesc".Translate(),
        icon = BW_Textures.VentIcon,
        action = () => VentNow(),
    };
    if (Pressure <= 0f)
    {
        vent.Disable("BW_NothingToVent".Translate());
    }
    yield return vent;
    if (DebugSettings.ShowDevGizmos)
    {
        yield return new Command_Action
        {
            defaultLabel = "DEV: Overpressure",
            action = () => Pressure = MaxPressure * 2f,
        };
    }
}
```

- `Command_Toggle` takes `isActive` and `toggleAction`.
- `DebugSettings.ShowDevGizmos` is true only in dev mode with god mode on
  [decomp].
- When several selected things show matching gizmos, the grid shows one,
  and a click runs `ProcessInput` on every enabled match that
  `InheritInteractionsFrom` accepts, then `ProcessGroupInput` with the
  whole group [decomp]. One button or slider
  can therefore drive the whole selection; override `GroupsWith` and
  `ProcessGroupInput` to control that.
- `Gizmo_Slider` is the vanilla slider base. Subclass it and implement
  `Target`, `ValuePercent`, `Title`, `DraggingBar`, and `GetTooltip`.
- Find vanilla icon paths in the game's asset index (see the
  [tooling overview](../rimworld-tooling.knack/rimworld-tooling.overview.knack.md))
  instead of guessing them, and load icons in a
  `[StaticConstructorOnStartup]` class.

## Inspect Pane

`GetInspectString()` on a `ThingWithComps` joins the base text and each
comp's `CompInspectStringExtra()` with line breaks. In dev mode, a comp
string that ends in whitespace logs an error once [decomp]; return lines
without a trailing newline. Use the pane for state the player cannot see
otherwise: a mode, a distance, a countdown.

Job report strings (the status under a pawn's name) are lowercase and end
with a full stop; see [Pawns and AI](rimworld-modding.api.pawns-and-ai.knack.md).

## Messages and Letters

- `Messages.Message(text, lookTargets, MessageTypeDefOf.NeutralEvent)` shows
  a short message; with look targets, a click jumps the camera to the
  subject.
- `Find.LetterStack.ReceiveLetter(label, text, LetterDefOf.NeutralEvent,
  lookTargets)` files a letter. Use letters only for events the player
  must not miss; frequent events belong in messages.

## Drawing on the Map

Two places draw on the map: `MapComponentUpdate` or a thing's draw methods
for world-space graphics (`GenDraw`, meshes), and `MapComponentOnGUI` for
screen-space text. GUI code has rules of its own [decomp]:

- `MapComponentOnGUI` runs only for `Find.CurrentMap`, and only while the
  map, not the world, is drawn.
- It runs on **every IMGUI event** (layout, mouse, keyboard), not once a
  frame. Draw only when `Event.current.type == EventType.Repaint`, as
  vanilla's thing overlays do.
- The game catches an exception from it and logs it, on every event, every
  frame. Wrap the body in try/catch, log once, and stop drawing after the
  first failure.
- Restore `Text.Font`, `Text.Anchor`, and `GUI.color` in a `finally`
  block. Text helpers such as `GenMapUI.DrawThingLabel` reset them to
  `Small`, `UpperLeft`, and white.
- Cull to `Find.CameraDriver.CurrentViewRect`, and skip fogged cells.
- The GUI types live in Unity modules a mod may not reference by default:
  `GUI` and `Event` in `UnityEngine.IMGUIModule`, `TextAnchor` in
  `UnityEngine.TextRenderingModule`.

For a label attached to something on the map, a `Thing` whose def sets
`drawGUIOverlay` and which overrides `DrawGUIOverlay()` gets culling, fog,
and event handling from vanilla [decomp].

Buildings are usually baked into the map mesh. A def whose `drawerType` is
`MapMeshOnly`, as vanilla's `BuildingBase` sets, is never registered for
per-frame drawing, so its comps' `PostDraw` never runs [decomp]. For an
animated part, set `drawerType` to `MapMeshAndRealTime`, or draw it from a
`MapComponentUpdate` with `Graphics.DrawMesh`, rotated and scaled each
frame.

Materials are pooled. `MaterialPool.MatFrom` caches one material per
distinct request, color included, and keeps it [decomp]. An effect that
animates color through it, such as a pulsing warning band, should quantize
the color to a few levels rather than create a material per frame.

## Debug Actions

Developer actions appear under the dev-mode Actions menu:

```csharp
[LudeonTK.DebugAction("Mods", "Place steam press here",
    actionType = DebugActionType.ToolMap,
    allowedGameStates = AllowedGameStates.PlayingOnMap)]
private static void PlaceSteamPress()
{
    SteamPressPlacer.PlaceAt(Find.CurrentMap, UI.MouseCell());
}
```

- The attribute is `LudeonTK.DebugActionAttribute`. The menu scans every
  type's static methods, public or not [decomp].
- Allowed shapes [decomp]: `static void M()` for `Action` and `ToolMap`;
  `static void M(Pawn p)` for `ToolMapForPawns`; or a method returning
  `List<DebugActionNode>` or `DebugActionNode` for submenus. Any other
  shape throws from `Delegate.CreateDelegate` while the menu is built.
- A `ToolMap` action is labeled "T: ...", and runs on every left click
  until a right click [decomp]. Guard against acting twice in one place.
- The `requiresBiotech` (and sibling) arguments hide an action when that
  DLC is inactive.

## Dev Mode and the Log Window

In dev mode, any logged error opens the debug log window over the game
[decomp]. An error from routine code, such as destroying an indestructible
thing while clearing ground, therefore covers a demonstration. Keep
routine paths error-free, not merely survivable.

The options dialog refuses a UI scale unless the screen is at least 1,024
by 768 scaled pixels, but a value written to `Prefs.xml` is applied without
that check [decomp]. That is how a launch script sets 2x scale on a smaller
window. Outside dev mode, though, the game resets the scale and logs
"Resolution too small" once the scaled screen is narrower than 1,024
*and* shorter than 768 [decomp].

## Review Checklist

- Do gizmos yield the base set first and appear only for the owner?
- Do disabled gizmos explain why?
- Do grouped gizmos act correctly on a multiple selection?
- Is GUI drawing limited to repaint events, culled, guarded, and
  state-restoring?
- Are animated colors quantized?
- Do debug actions have an allowed shape and guard repeated clicks?

## Further Information

- RimWorld Wiki, [DebugActions](https://rimworldwiki.com/wiki/Modding_Tutorials/DebugActions)
  and [TweakValue](https://rimworldwiki.com/wiki/Modding_Tutorials/TweakValue)
- RimWorld Wiki, [Textures](https://rimworldwiki.com/wiki/Modding_Tutorials/Textures)
- Ludeon's partial source in the install:
  `Source/RimWorld/ThingComps/CompForbiddable.cs` and `CompExplosive.cs`
  (gizmo examples)
- Unity 2022.3 manual, Immediate Mode GUI:
  <https://docs.unity3d.com/2022.3/Documentation/Manual/GUIScriptingGuide.html>
