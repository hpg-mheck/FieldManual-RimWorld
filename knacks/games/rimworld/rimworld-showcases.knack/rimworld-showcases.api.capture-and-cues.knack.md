# RimWorld Showcase Capture and Cues

Load this companion after the
[RimWorld showcases overview](rimworld-showcases.overview.knack.md) when
launching a showcase for a viewer, timing it, or capturing it for review.

**Status.** The current, hand-built approach; revise when the planned
showcase API exists.

Source review: 2026-10-13, against RimWorld 1.6.4871. Evidence marks:
**[decomp]** means verified only by decompiling `Assembly-CSharp.dll` from
1.6.4871; **[observed]** means seen in game testing on 1.6.4871 but not
traced to a published source.

## Launching for a Viewer

Use the [launch harness](../rimworld-testing.knack/rimworld-testing.api.launch-harness.knack.md)
with the showcase flag and preferences of its own. Unattended test runs
stay small and muted; a run for a person gets:

- sound (a moderate `volumeMaster`, such as 0.6);
- `uiScale` 2 for a high-resolution display. The options dialog refuses a
  scale that leaves less than 1,024 by 768 scaled pixels, but a value in
  `Prefs.xml` is applied without that check [decomp];
- a larger starting window, maximized once the game is ready (below);
- `automaticPauseMode` `Never` and `pauseOnError` false, so neither a
  threat letter nor an error freezes the demonstration;
- an isolated data folder, so the viewer's own settings, saves, and mod
  list are untouched;
- no shutdown at the end: the game stays open for the viewer.

Before launching, check whether the person's own RimWorld is running, and
never close a game window you did not start. When asked to restart a
showcase, confirm afterwards that only one instance is running, and tell
the person which window is yours [observed].

## Maximizing After the Game Is Ready

The game changes its resolution during startup, which undoes a window
maximized too early. Wait for the showcase's `READY` cue in the log, then
maximize [observed]. On Windows, PowerShell can call
`ShowWindowAsync(hWnd, 3)` (`SW_MAXIMIZE`) from `user32.dll` on the
process's main window handle:

```powershell
Add-Type -Namespace Win32 -Name Window -MemberDefinition @'
[DllImport("user32.dll")]
public static extern bool ShowWindowAsync(System.IntPtr hWnd, int nCmdShow);
'@
$proc.Refresh()
[Win32.Window]::ShowWindowAsync($proc.MainWindowHandle, 3) | Out-Null
```

## Log Cues

The showcase writes one log line per beat, with a fixed prefix, for
outside tools to wait on:

```text
[BW-SHOWCASE] READY
[BW-SHOWCASE] SITE town at (51,166)
[BW-SHOWCASE] SCENE steam-press
[BW-SHOWCASE] SKIPPED boiler-demo: no pad clears the town
[BW-SHOWCASE] SCENE finale
```

- Tools wait on cues, never on wall-clock delays. Heavy effects slow the
  game below real time; computing one large barrage alone took one to two
  seconds [observed].
- Log a cue for every skip, with its reason, as well as for every beat
  that runs.
- Include positions and names in cues, so a capture tool can find the
  subject and a reader can match a frame to a scene.
- Keep the vocabulary small and stable; a scene name, not a sentence.

## Timing Inside the Game

A timeline runs from a tick hook (`GameComponentTick`, 60 ticks a second
at normal speed) and fires each beat at a tick offset from `READY`. A
director advances on events instead: the aftermath ended, the pull
completed, the smoke settled. Either way, schedule by game ticks, not real
time, so speed changes and slowdowns keep the sequence intact.

## Capturing

- Capture the game window at each cue. A tool that tails the log and
  captures on a cue gets the right moment regardless of speed.
- For animation, capture a burst of frames, crop the area of interest,
  scale it up without smoothing, and tile the frames into one contact
  sheet; one image then shows a pulse, a spin, or a flip [observed].
- Look at the captures. They are the only way an agent sees what it drew,
  at the zoom and UI scale a viewer uses, beside vanilla's art.
- When a person reports that "something extreme happened", capture the
  screen and scan the log of that instance at once; the explanation is
  usually in both. A running instance keeps the assembly it loaded, so it
  keeps showing an old bug until restarted [observed].

## Review Checklist

- Does the viewing run have its own preferences: sound, scale, no
  automatic pause, isolated data?
- Is the window maximized only after `READY`?
- Does every beat and every skip log a cue?
- Do outside tools wait on cues rather than delays?
- Were captures taken at each cue, and actually inspected?

## Further Information

- Microsoft, `ShowWindowAsync`:
  <https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-showwindowasync>
- Unity 2022.3 manual, Player command-line arguments:
  <https://docs.unity3d.com/2022.3/Documentation/Manual/PlayerCommandLineArguments.html>
- [Showcase Staging](rimworld-showcases.api.staging.knack.md)
