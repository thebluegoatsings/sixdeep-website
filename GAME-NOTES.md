# Wrecking Ball Mini-Game — Notes

Scope: the 3D Box3D prototype at `/play-3d` (`src/components/CraneGame3D.astro`).

## 1. Feel — intuitive and satisfying

- The game should be instantly playable: the player can just **hold down A or D** and the crane swings the ball.
- Ideally, holding A or D should build up to a **massive, satisfying blast into the middle of the wall** — big impact, lots of boxes knocked at once, strong feedback.
- "Satisfying" is the priority: weighty ball, good impact, visible destruction.

## 2. Wall rows — offset vs. clean

- The wall rows are currently **staggered** (brick offset).
- Open decision: either
  - **keep the row offset**, or
  - **revert to clean, aligned rows**.
- If we keep the offset: **fix the cube that falls off** at spawn — the stagger leaves an overhanging edge cube that drops on its own. Possible directions: trim the overhanging cube, shorten the stagger, or add support so nothing falls.
