# Wrecking Ball Mini-Game — Notes

Scope: the 3D Box3D prototype at `/play-3d` (`src/components/CraneGame3D.astro`).

## 1. Feel — intuitive and satisfying

- The game should be instantly playable: the player can just **hold down A or D** and the crane swings the ball.
- Ideally, holding A or D should build up to a **massive, satisfying blast into the middle of the wall** — big impact, lots of boxes knocked at once, strong feedback.
- "Satisfying" is the priority: weighty ball, good impact, visible destruction.

## 1b. Winch hoist (chain draw / pull) — chain winds onto the spool

Raise/lower (W/S) is a **winding chain winch**. The chain itself winds around a spool at the jib tip — there is no separate cable/rod.

- The chain is 18 capsule links (`TOTAL_LINKS`). The top `coil` links are **wound onto the spool** (index 0 is deepest); the rest are the **free chain** that hangs below and swings with the ball.
- The spool is **kinematic**: its rotation is set directly to `coil * DELTA` each frame, so it can never stall under the chain's load (a dynamic/motor wheel did stall — see notes). Wound links are driven onto the spool ring every frame (`driveWoundLinks`): same physical links, placed at world angle `(coil-1-i)*DELTA` around the spool, newest at the bottom fairlead where the free chain exits.
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

## 2. Wall rows — offset vs. clean

- The wall rows are currently **staggered** (brick offset).
- Open decision: either
  - **keep the row offset**, or
  - **revert to clean, aligned rows**.
- If we keep the offset: **fix the cube that falls off** at spawn — the stagger leaves an overhanging edge cube that drops on its own. Possible directions: trim the overhanging cube, shorten the stagger, or add support so nothing falls.

## 3. Future work (winch) — requested, NOT implemented yet

Two follow-ups the user asked for on 2026-09-06 ("good place to stop for now, but in the future I'd like to…"):

1. **Make the winch rotate with the crane arm.** Right now the winch/spool keeps a fixed orientation while the jib arm swings with A/D — it doesn't turn with the arm. Future: the winch should yaw/rotate with the arm so it always aligns with where the jib is pointing (reads as part of the jib).

2. **Keep the chain's exit point fixed while winding.** When retracting or lowering, the wound links currently travel up and around the winch because the spool doesn't spin to keep the tangent fixed — so the "last chain" wraps around instead of leaving from one consistent spot. Future: make the winch actually spin as it winds so the free chain always leaves from the same place — hanging straight down at the bottom, or trailing off to the far side — like a real winch whose barrel turns.

These are really one underlying improvement: the spool should **spin in sync with the arm and with the winding**, keeping the chain's exit/tangent point anchored rather than spreading wound links around a non-spinning hub.
