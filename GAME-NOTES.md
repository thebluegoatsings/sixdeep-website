# Wrecking Ball Mini-Game — Notes

Scope: the 3D Box3D prototype at `/play-3d` (`src/components/CraneGame3D.astro`).

## 1. Feel — intuitive and satisfying

- The game should be instantly playable: the player can just **hold down A or D** and the crane swings the ball.
- Ideally, holding A or D should build up to a **massive, satisfying blast into the middle of the wall** — big impact, lots of boxes knocked at once, strong feedback.
- "Satisfying" is the priority: weighty ball, good impact, visible destruction.

## 1b. Winch hoist (chain draw / pull) — chain winds onto the spool

Raise/lower (W/S) is a **winding chain winch**. The chain itself winds around a spool at the jib tip — there is no separate cable/rod.

- The chain is 18 capsule links (`TOTAL_LINKS`). The top `coil` links are **wound onto the spool** (index 0 is deepest); the rest are the **free chain** that hangs below and swings with the ball.
- The spool is **kinematic**: we set its pose directly every frame, so it can never stall under the chain's load (a dynamic/motor wheel did stall — see notes). Its body is **yaw-only** — it turns with the jib but does not spin, see §3 — while the barrel's winding spin is applied to the visual mesh. Wound links are driven onto the drum every frame (`driveWoundLinks`): same physical links, placed at angle `(coilSmooth - 1 - i) * DELTA` measured from the fairlead, inside the vertical plane that contains the jib, newest at the bottom fairlead where the free chain exits.
- Winding (W) increments `coil`: the free top link is detached from the chain and becomes a wound link, so the free chain shortens one `LINK_PITCH` and the ball rises. Letting out (S) reverses it: the outermost wound link is released back into the free chain. `coil` is the source of truth, clamped to `[MIN_COIL, MAX_COIL]`.
- The free chain top is pinned to the spool's underside (the fairlead) by one spherical joint (`boundaryJ`), so the chain below it swings freely (A/D rotates the crane → wrecking ball still smashes the wall).
- Chain links and the spool never collide (shape filter mask = 0) — only the ball collides with the wall/ground — so the coil can't fight the spool or jitter.
- Verified live: W winds `coil` 2→13 and the ball rises ~1.1 m → ~3.6 m; S unwinds 13→0 and it lowers back to the floor; swinging with the ball raised topples the wall (112+ boxes). Constants to tune: `SPOOL_R`, `LINK_EXTENT`, `TOTAL_LINKS`, `MAX_COIL`, `WIND_RATE`.

Historical context (2026-09-06): the demo ("Gear Lift", box3d `samples/sample_joint.cpp`) lifts a rail-guided gate with chains offset at a fixed gear's rim — that free-swing geometry doesn't produce a wrap in-engine. Earlier attempts here (chain pinned at a rotating wheel, welded growing coil, etc.) failed live because a free ball drapes slack instead of wrapping. The winding model above is the controlled realization of the user's "glue the last contact link, chain below swings free" idea: each link is taken onto the spool as the coil grows, and the chain below genuinely swings.

## 1c. Chain-link stretch / visible gaps

- Complaint (2026-09-06): links "stretch apart" with visible gaps, worst at full retract. Causes: the ball is heavy (density 90 ≈ 500 kg) and hangs on a short free chain via point joints that are soft at the default stiffness, so the joints stretch; plus each winding step re-anchors the free chain one pitch shorter, which tugs the ball (transient whip).
- Fixes applied (conservative, keep wrecking): ball density 90 → 85; chain-joint `constraintHertz` 120 → 200; `LINK_TORQUE` 5 → 6; `WIND_RATE` 5 → 4 (gentler winding); solver iterations 24 → 28.
- Verified on a healthy page: with a lighter/stiffer setup the fully-retracted ball settles at the geometrically-correct height (~4.2 m at coil 13) instead of sagging to ~3.5 m — the sag was the stretched/gapped look.
- Tuning knobs if needed: too many gaps → raise `constraintHertz` (up to ~240) or lower ball `density`; wrecking too weak → raise `density` back toward 90 (heavy ball stretches the chain more); winding too snappy → lower `WIND_RATE`.
- **Superseded — see 1d below.** Raising `constraintHertz` turned out to be a dead end; the real cause was link mass.

## 1d. Chain stretch — root cause found: link mass, not joint stiffness (2026-09-14)

Re-reported as "the chain is a little too springy, it would be nice if it was a little tighter". The
2026-09-06 fixes (1c) treated the symptom.

The cause is the **mass ratio**. The ball is ~474 kg; at `LINK_DENSITY = 16` each link massed ~0.04 kg —
about **12,000:1**, which a solver cannot resolve. The free chain stretched regardless of how the joints
were configured. Measured at full retract (coil 13, free chain = 5 joints) by reading
`b3Joint_GetLinearSeparation` on every joint in the free chain:

| setting | chain stretch | per joint | raised ball height | ball bob at rest |
| --- | --- | --- | --- | --- |
| `LINK_DENSITY 16` (old) | 0.60 m | ~120 mm | 3.54 m | 297 mm |
| `LINK_DENSITY 100` (new) | 0.095 m | ~19 mm | 4.31 m | 0 mm |

The geometric ball height at coil 13 is ~4.4 m. Adjacent links **overlap by ~100 mm at rest**, so
anything under ~100 mm per joint is invisible — 120 mm is just past that line, which is exactly why the
gaps were showing.

### Two traps in this area, both of which cost me time

- **`constraintHertz = 0` looks like the fix and is a trap.** Stretch reads ~0.03 m and the bob reads
  zero, but hertz *is* the constraint's position-correction bias: at 0 the chain stops correcting error
  and simply **sinks**, so the ball quietly drops to the floor (4.4 m → 1.1 m, i.e. resting on the
  ground at full retract). When tuning this, always verify the ball's **height**, not just the stretch.
- **Higher hertz stops helping.** Measured 0 / 200 / 800 / 1600 Hz. 800 is the stiffest that stays
  clean; 1600 adds jitter for no extra rigidity. Once the conditioning is fixed, stiffness is not the
  lever — it stays at 200.

### The trap that comes *with* a tight chain: sleeping

With the chain rigid and the ball hanging still, the ball qualifies for **sleep** within seconds — and a
sleeping body **ignores the crane entirely**, so A/D silently stops working. Measured: zero displacement
over a 6-second swing. The old springy chain kept micro-jittering, which incidentally kept the body awake
and masked the problem. This is very likely why an earlier attempt at a tighter chain got reverted.

Fixed with `enableSleep = false` on the ball and the chain links **only** — the 196 wall boxes still
sleep normally.

### A note on the measurements

Knock counts in this file are noisy. The page's own animation loop interleaves with scripted stepping,
and a 15-second wall collapse is chaotic, so a single scripted sweep varies by roughly ±25% run to run.
Tune on the deterministic measurements (stretch, ball height) and then play it. For reference, a real
3-second swing on the shipped build knocks ~93 of 196 boxes.

If the wrecking ever feels weak, remember the old stretch was acting as a whip. The levers are
`ballDef.linearDamping` (0.2) for how long the ball keeps swinging, and ball `density` (85) for raw
impact energy. For chain tightness, `LINK_DENSITY` is the one that matters.

## 2. Wall rows — offset vs. clean

- The wall rows are **staggered** (brick offset), and that stays for now. Reverting to clean,
  aligned rows is still an open decision — it is a one-line change either way (`rowStagger = 0`).
- **Fixed (2026-09-14): the cube that fell off at spawn.** The offset was half a box
  (`BOX_SPACING / 2`), which put the outer box of every offset row's centre of mass exactly on
  the edge of the box below it, so it toppled by itself and the score read 1/196 before the
  player touched anything. Shortened to `BOX_SPACING / 4`: still reads as a brick offset, but
  the centre of mass now sits well inside the support. Verified by stepping the simulation for
  4 s with no input — 0 boxes knocked.

## 3. Winch on the jib — both requests implemented (2026-09-14)

The two follow-ups from 2026-09-06 are in. They really were one underlying change.

1. **The winch turns with the arm.** The drum carries a yaw of `-craneAngle`, so its spin axis stays
   horizontal and perpendicular to the jib, and the coil wraps in the vertical plane that contains the
   jib. Previously the wrap plane was locked to world X-Y, so the coil only looked right when the jib
   happened to point along X.
2. **The chain always leaves from the same place, and the drum visibly turns.** The trick is that the
   barrel's *spin* is applied to the visual mesh only — the physical spool body is yaw-only. The free
   chain's top link is pinned to a frame on that body, so a spinning body would have dragged the
   chain's exit point up and around the drum (this was the old "last chain wraps around" look).
   Keeping the body unspun holds the fairlead bolted to the bottom of the drum, and the mesh still
   reads as a turning barrel.

Also in this pass:

- `coilSmooth` — an eased copy of `coil` drives the wound-link placement and the drum's visual spin, so
  the coil glides between winding steps instead of snapping a whole link around four times a second.
  `coil` itself stays a whole number of links, so the ball height stays discrete.
- **Fixed: the demo loaded with twice as many links wound as Reset produced.** `coil` was initialised to
  `START_COIL` *and* the pre-wind loop wound `START_COIL` on, so a fresh load started at coil 4 while
  Reset returned to 2. Now `coil` starts at `MIN_COIL`, so both agree at `START_COIL`.
  (If you preferred the deeper coil at rest, set `START_COIL = 4` — that is the only knob.)
- The render loop is split into `step(dt)` (simulation + render) and `loop(now)` (timing), so the
  simulation can be driven deterministically in tests and tooling.
- **Winch mount redrawn.** The drum used to hang off a single thin post (a 14 cm rod, 1.6 m long) under
  the jib tip, which read as disconnected from the arm. It now sits in a bracket: a bridge plate under
  the beam, two slim legs down to a pair of bearing blocks, and the axle with accent collars on its
  ends. Everything is parented to `jibGroup`, so it yaws with the arm for free. The first attempt used
  two full-height side plates, but from the default camera the near plate covered the drum's face and
  hid the winding — slim legs plus compact blocks keep the drum readable.
  This pass is **visual only** — no joints, bodies or tuning constants changed, so the ball's reach and
  the winding feel are exactly as verified above.
  Knobs if the 1.6 m drop still looks long: lower `JIB_Y` (shortens the drop with **zero** gameplay
  impact, because the chain hangs from `SPOOL_Y`), or raise `SPOOL_Y` and add ~3 to `TOTAL_LINKS` so the
  ball still reaches the floor (that one does touch tuned constants — re-verify afterwards).

Verified numerically by driving `step()` directly (240 steps = 4 s of simulated time per case):

| coil wound | ball Y | drift of the chain's exit point from below the drum | drum spin |
| ---------- | ------ | --------------------------------------------------- | --------- |
| 2          | 1.10 m | 0.000 m                                             | 49.7°     |
| 5          | 1.10 m | 0.000 m                                             | 124.1°    |
| 7          | 1.49 m | 0.000 m                                             | 173.8°    |
| 11         | 2.90 m | 0.000 m                                             | 273.1°    |

- The exit point drift is exactly zero at every coil depth — that is request 2.
- The drum spins 24.8° per link (`DELTA`), matching the coil, so the barrel turns with the winding.
- The spool body's quaternion stays a pure yaw at every coil depth, which is what keeps the anchor fixed.
- Wound links sit exactly in the jib's vertical plane and their long axes lie in that plane (worst
  deviation 0.000) at every arm angle — that is request 1.

Still true after the change: winding `coil` 2→13 lifts the ball, `S` lowers it to the floor (1.1 m),
and a scripted swing still topples 81/196 boxes.
