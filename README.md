# FieldManual-RimWorld

FieldManual-RimWorld is a supplemental knack pack for
[FieldManual](https://github.com/Hard-Problems-Group-LLC/FieldManual), the
reusable standards-and-practices library for AI-assisted development. It
gives human and AI collaborators the knowledge needed to build RimWorld
mods well: how the game loads and runs mod content, where its sharp edges
are, and how to test and demonstrate a mod automatically.

It is a separate project because most FieldManual projects never touch
RimWorld. A mod project loads it alongside FieldManual; nothing else
should.

The material was distilled from the development of a working RimWorld 1.6
content mod, checked against the game's own data and partial source, and
verified where necessary by decompiling the game. Each claim says which.

## Contents

The knacks live under [`knacks/games/rimworld/`](knacks/games/rimworld/README.md),
in four filesets, each an overview with API companions:

- **Modding** — mod layout and identity, defs and XML, patches before
  Harmony, the tick lifecycle, things and maps, buildings and storage,
  pawns and AI, pawn generation and factions, combat and effects, UI and
  debugging
- **Testing** — a layered strategy, static tests, an in-game smoke runner,
  and the launch harness that drives it
- **Tooling** — building against the game, packaging for the Workshop,
  decompiling, reflection, finding vanilla assets, and a reflection-based
  def validator
- **Showcases** — staging in-game demonstrations for a person to watch,
  with log cues for capture tooling

Start at the [RimWorld index](knacks/games/rimworld/README.md), which also
explains the evidence marks used throughout.

## Adding It to a Project

Add the pack as a Git submodule next to FieldManual:

```bash
git submodule add https://github.com/hpg-mheck/FieldManual-RimWorld.git \
    FieldManual-RimWorld
git submodule update --init
```

FieldManual's configuration and bootstrap do not manage supplemental
packs, so tell agents about it in the project-specific part of
`AGENTS.md`, between FieldManual's managed header and footer:

```markdown
## RimWorld

This project is a RimWorld mod. Besides FieldManual, read
`FieldManual-RimWorld/knacks/games/rimworld/README.md` and load the
knacks it lists for the work at hand. Project-owned knacks under
`knacks/` refine both.
```

Treat the submodule as read-only guidance, as FieldManual itself is.
Project-specific facts, such as the mod's own names, conventions, and
decisions, belong in the project's own `knacks/` tree and records, not in
this pack. To pick up a newer review, update the submodule deliberately:

```bash
git submodule update --remote FieldManual-RimWorld
```

## Scope

- **Game version:** RimWorld 1.6 (build 1.6.4871), with Royalty, Ideology,
  Biotech, Anomaly, and Odyssey. Earlier versions are out of scope, and
  the 1.7 beta was not reviewed.
- **Reviewed:** 2026-10-13.
- **Platform:** the game material is platform-neutral; build, launch,
  and window-handling examples assume Windows.
- **Examples** use a fictional mod, "Brass Works". No real mod, author, or
  machine is described.

Claims verified only by decompilation are marked **[decomp]**, and those
seen only in game testing are marked **[observed]**. Both describe build
1.6.4871 and should be re-checked after a game update.

## Maintenance

Review the pack when RimWorld updates: confirm the build, re-run the
checks behind every marked claim, fix what changed, and update the review
dates in each knack and index. Follow FieldManual's
`knacks/authoring-guide.md` for sizing, naming, sources, and the manual
review list.

## License and Trademarks

Released under the MIT License; see [LICENSE](LICENSE).

RimWorld is a trademark of Ludeon Studios. This pack is unofficial and is
not affiliated with or endorsed by Ludeon Studios. It reproduces no game
code beyond identifiers and short formulas, and decompiled code should
stay out of every repository.
