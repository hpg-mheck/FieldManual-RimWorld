# RimWorld Combat and Effects

Load this companion after the
[RimWorld modding overview](rimworld-modding.overview.knack.md) when code
makes explosions, starts fires, launches projectiles, reacts to damage, or
works with long ranges and large radii.

Reviewed: 2026-10-14, against RimWorld 1.6.4871. Evidence marks:
**[decomp]** means verified only by decompiling `Assembly-CSharp.dll` from
1.6.4871; **[observed]** means seen in game testing on 1.6.4871 but not
traced to a published source.

## Explosions

`GenExplosion.DoExplosion(center, map, radius, damType, instigator,
damAmount: ..., chanceToStartFire: ..., propagationSpeed: ...)` spawns an
`Explosion` that expands over several ticks; it is not instantaneous. The
wave reaches a cell at distance *d* at
`startTick + (int)(d * 1.5 / propagationSpeed)` [decomp], so a 75-tile
blast at the default speed takes 112 ticks to reach its edge. Judge the
damage of a blast only after its wave has arrived; a test that checked a
target immediately judged it before it was hit [observed].

### Bomb Damage Puts Out Fires

A `Bomb` explosion destroys every `Fire` its wave touches [decomp]. Fires
lit at the moment of detonation are therefore extinguished by the blast
that should have started them, and the result looks like a bare shockwave
[observed]. Two remedies:

- light fires *behind* the wave: schedule each cell for a few ticks after
  the formula above says the wave arrives;
- use `BombSuper` for secondary blasts that must leave fires burning. It
  inherits `Bomb`'s look and is not caught by the fire check, but its
  default damage is 550 (Core's `Damages_Misc.xml`), so always pass
  `damAmount`.

A projectile's `explosionChanceToStartFire` defaults to 0, so explosive
projectiles start no fires unless told to.

### Reach Budgets

When something must stay out of a blast, budget the whole effect: the main
radius, plus every secondary blast's offset and radius, plus fires that
spread afterwards. In one design, a 75-tile blast with secondary
explosions near its edge budgeted out to about 84 tiles.

### Explosive Items

`CompExplosive` (shipped in `Source/RimWorld/ThingComps/`) is the vanilla
pattern for countdowns, wicks, and detonation. A stack's radius grows with
its size: `explosiveRadius + sqrt((stackCount - 1) *
explosiveExpandPerStackcount)`. Stacks that take enough damage chain into
each other.

## Fire

- `FireUtility.TryStartFireIn(cell, map, fireSize, instigator)` requires
  an instigator argument; there is no overload without one.
- A fire spreads only once its size exceeds 1. It then tries every 75 to
  150 ticks, depending on size: 80 percent of the time to one of the eight
  adjacent cells, otherwise as a spark to a cell two away (shipped
  `Fire.cs`). Ignition depends on the flammability of what is in the target
  cell.
- Distance alone does not stop fire. A belt three or four cells wide with
  nothing flammable in it does. Roads of `BrokenAsphalt` or `PackedDirt`
  stay bare; gravel regrows grass.
- Thousands of fires are expensive: an aftermath of about 90 secondary
  explosions and a few thousand fires cost about 9 ms per tick while it
  ran [observed].

## Sounds

A `SoundDef` is either a sustainer (`sustain` true), started with
`TrySpawnSustainer` and kept alive by its owner, or a one-shot played with
`PlayOneShot`. Using the wrong call logs an error and plays nothing
[decomp]. Similar names do not tell you which: Core's `HissSmall` is a
sustainer, `PsychicPulseGlobal` a one-shot. Check the def under
`Data/Core/Defs/SoundDefs/` before reusing it.

## Projectiles

Launch with `projectile.Launch(launcher, origin, usedTarget,
intendedTarget, hitFlags, ...)`.

**Never launch without a launcher.** Two vanilla paths dereference it:

- every projectile's impact makes noise within 12 tiles; an animal with
  no master that hears a `Bullet`-class impact may (40 percent) flee from
  the projectile's `Launcher`, which throws a `NullReferenceException`
  when it is null [decomp] (`Bullet.AnimalsFleeImpact` is true in the
  shipped source);
- a delayed explosive projectile reads `launcher.Faction` on landing
  (shipped `Projectile_Explosive.cs`).

When the true source is gone, as when a destroyed device "cooks off" its
ammunition, launch from a stand-in: an invisible thing spawned at the
origin that removes itself after a while.

Overhead projectiles and roofs [decomp]: on impact under a thick rock roof
the projectile is destroyed without effect; under a thin or constructed
roof it first drops the roof in that cell, unless a full-fill edifice
stands there.

Verbs share their def's `VerbProperties` object: every turret of a def
points at the same instance [decomp]. To give one building its own range,
give its verb a clone (`VerbProperties.MemberwiseClone()` is public) and
change the clone.

## Thrown and Forced-Miss Verbs

A verb whose `forcedMissRadius` is above zero must launch an explosive
projectile (a `Projectile_Explosive` subclass, or one with
`CompProperties_Explosive`), and an explosive one must have a forced miss;
otherwise the game reports "incorrect forcedMiss settings" [decomp]. For a
thrown item with a custom effect, subclass `Projectile_Explosive` and
override its explosion.

## Radial Limits

`GenRadial` precomputes 20,000 cells, so the largest radius it can serve is
about 79.8 tiles; asking for more logs "Not enough squares to get to
radius ..." [decomp]. Clamp every radius, and remember the callers you do
not write:

- a player turret flagged `ai_combatDangerous` adds itself to raiders'
  avoidance grid with a radius of its range plus 4 [decomp], so a turret
  with a range beyond about 75 logs that error whenever the grid is
  rebuilt for raiders' pathing. Clear the flag or clamp the range; a test
  can force `map.avoidGrid.Regenerate()` to catch it;
- `GenDraw.DrawRadiusRing` refuses a radius beyond the limit and logs an
  error once [decomp], so a long-range turret needs an outline of its own,
  including while it is being placed. Rings are costly well before the
  limit: the ring collects every cell inside it each frame it shows
  [decomp], about 17,600 cells at 75 tiles.

## Damage Hooks

- `PreApplyDamage(ref DamageInfo dinfo, out bool absorbed)` can absorb a
  hit. Explosions call it on buildings even when the building does not use
  hit points, which makes it the place to react to blasts [observed].
- Use `dinfo.Def.harmsHealth` to tell harmful damage (bullets, blasts,
  fire) from stuns, EMP, and extinguishing foam.
- Do not destroy things or start explosions inside the hook; it runs inside
  another thing's damage pass. Record the hit and act on your next tick.
- Test damage reactions with real projectiles fired at the target rather
  than calling `TakeDamage`, so a target that projectiles cannot hit fails
  the test [observed].

## Review Checklist

- Are fires lit behind any `Bomb` wave, or secondary blasts `BombSuper`
  with explicit damage?
- Does every projectile have a launcher, even after its source dies?
- Is every radius clamped below the radial limit, including turret ranges
  fed to the avoid grid?
- Do keep-away distances include secondary blasts and fire spread?
- Do damage reactions defer their effects to the next tick?

## Further Information

- Ludeon's partial source in the install:
  `Source/RimWorld/ThingComps/CompExplosive.cs`,
  `Source/RimWorld/Thing/Fire/Fire.cs`,
  `Source/RimWorld/Thing/Projectile/Bullet.cs`, `Spark.cs`, and
  `Source/Verse/Thing/Projectile_Explosive.cs`
- Vanilla defs: `Data/Core/Defs/DamageDefs/Damages_Misc.xml` (`Bomb`,
  `BombSuper`) and `Data/Core/Defs/ThingDefs_Misc/Weapons/` (projectiles)
- RimWorld Wiki, [Weapons Guns](https://rimworldwiki.com/wiki/Modding_Tutorials/Weapons_Guns),
  [Basic Ranged Weapon](https://rimworldwiki.com/wiki/Modding_Tutorials/Basic_Ranged_Weapon),
  and [Plague Gun](https://rimworldwiki.com/wiki/Modding_Tutorials/Plague_Gun)
