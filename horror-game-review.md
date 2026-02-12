# Horror Maze Prototype Review

Overall: strong atmosphere and compact implementation. The main improvements I'd make are around **stability**, **performance**, and **game feel**.

## What works well
- Clear minimalist visual style with CRT-like neon-on-black readability.
- Nice tension loop: battery drain + sound awareness + entity chase.
- Save/load for run persistence is a good touch.

## Changes I'd make first

1. **Fix movement speed (currently frame-rate dependent)**
   - Holding a key moves every frame, which can be too fast.
   - Add movement cooldown (e.g. 100–150ms per step).

2. **Clamp gameplay stats**
   - `awareness` and `sanity` can run out of intended ranges.
   - Clamp values each update:
     - `battery: 0..100`
     - `sanity: 0..100`
     - `awareness: 0..100`

3. **Defensive save parsing**
   - `JSON.parse(localStorage.getItem("horrorSave"))` can throw on malformed data.
   - Wrap parsing in a guarded helper and validate types.

4. **Throttle expensive/annoying alerts**
   - Multiple ending checks can trigger close together while loop continues.
   - Add a `gameOver` boolean and return early once any ending is reached.

5. **Microphone fallback UX**
   - If mic permission is denied, game silently continues.
   - Show UI note like `Mic: unavailable` and use a baseline awareness drift.

6. **Maze playability guarantee**
   - Current random fill can create unreachable exits.
   - Use DFS/backtracking maze generation or path-check and regenerate until solvable.

7. **Resize handling**
   - Canvas and grid sizes are fixed at boot.
   - On `resize`, recalc dimensions and regenerate/reproject positions.

## Quick code sketch for highest-impact fixes

```js
let gameOver = false;
let lastMoveAt = 0;
const MOVE_DELAY_MS = 120;

const clamp = (v, min, max) => Math.max(min, Math.min(max, v));

function safeLoad() {
  try {
    const raw = localStorage.getItem("horrorSave");
    if (!raw) return;
    const save = JSON.parse(raw);
    battery = clamp(Number(save.battery) || 100, 0, 100);
    sanity = clamp(Number(save.sanity) || 100, 0, 100);
    awareness = clamp(Number(save.awareness) || 0, 0, 100);
  } catch {
    localStorage.removeItem("horrorSave");
  }
}

function triggerEnding(message) {
  if (gameOver) return;
  gameOver = true;
  alert(message);
  localStorage.removeItem("horrorSave");
  location.reload();
}

function update(now = performance.now()) {
  if (gameOver) return;

  if (now - lastMoveAt >= MOVE_DELAY_MS) {
    if (keys.w) move(0, -1);
    else if (keys.s) move(0, 1);
    else if (keys.a) move(-1, 0);
    else if (keys.d) move(1, 0);
    lastMoveAt = now;
  }

  if (awareness > 25) sanity -= 0.2;

  battery = clamp(battery, 0, 100);
  sanity = clamp(sanity, 0, 100);
  awareness = clamp(awareness, 0, 100);

  if (battery <= 0) triggerEnding("Your light dies. It steps into view.\nEnding: DARKNESS.");
}
```

## Nice-to-have polish
- Add footsteps/audio cues tied to awareness.
- Add pause menu and explicit reset button.
- Add subtle entity telegraph (flicker/red vignette) before contact.
