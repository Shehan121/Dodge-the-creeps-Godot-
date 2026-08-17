# Dodge the Creeps — Godot 4

A 2D arcade survival game built in **Godot 4.4** with **GDScript**. Enemies
spawn from the screen edges and fly across at random angles; you survive as long
as you can, scoring a point per second.

Based on the Godot engine's official *Dodge the Creeps* tutorial project.

![Godot](https://img.shields.io/badge/Godot-4.4-478CBF?logo=godotengine&logoColor=white)
![GDScript](https://img.shields.io/badge/GDScript-355570)

---

> [!IMPORTANT]
> **The project as committed is not playable without two fixes** — no main scene
> is set, and the input actions the player script depends on are not defined.
> Both are one-line changes in the Godot editor. See
> [Known issues](#known-issues).

---

## Running it

1. Open **Godot 4.4** and import
   `dodge-the-creeps-godot-main/DTC/dodge-the-creeps/project.godot`
2. Apply the two fixes in [Known issues](#known-issues)
3. Press **F5**

The project uses the **GL Compatibility** renderer, so it runs on older hardware
and exports cleanly to web and mobile targets.

---

## How it works

### Scene structure

Four scenes, each with its own script:

| Scene | Root node | Responsibility |
|---|---|---|
| `Main.tscn` | `Node` | Game loop — spawning, scoring, timers |
| `Player.tscn` | `Area2D` | Movement, animation, collision detection |
| `Mob.tscn` | `RigidBody2D` | An enemy — random type, physics-driven motion |
| `HUD.tscn` | `CanvasLayer` | Score, messages, start button |

### Signal wiring

Nodes communicate through signals rather than direct references, connected in
`Main.tscn`:

```
Player  ── hit ──────────►  Main.game_over()
HUD     ── start_game ───►  Main.new_game()
MobTimer   ── timeout ───►  Main._on_mob_timer_timeout()
ScoreTimer ── timeout ───►  Main._on_score_timer_timeout()
StartTimer ── timeout ───►  Main._on_start_timer_timeout()
```

This is why `Main._ready()` contains a commented-out `new_game()` and a `pass` —
the game deliberately does **not** start on load. It waits for the HUD's Start
button to emit `start_game`. That is the correct design, and the commented line
is a leftover from the tutorial.

### Player movement

Eight-directional movement built from four independent axis checks, then
normalised:

```gdscript
if velocity.length() > 0:
    velocity = velocity.normalized() * speed
```

The `normalized()` call is the important part — without it, holding two
directions at once would give a diagonal speed of `√2 × speed`, making diagonal
movement about 41% faster than straight movement. This is the single most
commonly missed detail in 2D movement code.

`position.clamp(Vector2.ZERO, screen_size)` keeps the player on screen, and
sprite animation switches between `walk` and `up` with `flip_h` / `flip_v`
mirroring rather than separate art for each direction.

On collision the player hides, emits `hit`, and disables its collision shape
with `set_deferred("disabled", true)` — deferred because you cannot modify a
physics body's collision state during the physics callback that reported it.
Setting it directly is a classic source of engine errors here.

### Mob spawning

Each enemy is placed on a `PathFollow2D` around the screen edge at a random
point:

```gdscript
mob_spawn_location.progress_ratio = randf()          # random point on the path
var direction = mob_spawn_location.rotation + PI / 2 # perpendicular, i.e. inward
direction += randf_range(-PI / 4, PI / 4)            # ±45° of variation
mob.linear_velocity = Vector2(randf_range(150.0, 250.0), 0.0).rotated(direction)
```

Taking the path's rotation and adding `PI / 2` aims each mob **inward** across
the screen; the `±PI/4` jitter stops them all travelling on parallel lines. Mobs
pick a random animation from their available sprite frames, so enemy variety
costs no extra code.

Cleanup is handled by `VisibleOnScreenNotifier2D`: once a mob leaves the view it
calls `queue_free()` on itself, so off-screen enemies do not accumulate.

### Game over sequence

`HUD.show_game_over()` uses `await` to sequence messages without nested timers
or state flags:

```gdscript
show_message("Game Over")
await $MessageTimer.timeout
$Message.text = "Dodge the Creeps!"
await get_tree().create_timer(1.0).timeout
$StartButton.show()
```

Reading top to bottom in the order it happens — the practical argument for
coroutines over callback chains.

---

## Known issues

Both verified against `project.godot`:

1. **No main scene is configured.** `project.godot` has no `run/main_scene`
   key, so pressing Play opens the "select a main scene" dialog instead of
   running the game.
   **Fix:** *Project → Project Settings → Application → Run → Main Scene* →
   `Main.tscn`.

2. **The input actions are not defined.** `Player.gd` reads `move_left`,
   `move_right`, `move_up` and `move_down`, but `project.godot` has no `[input]`
   section at all. These are custom action names, not Godot built-ins, so every
   `Input.is_action_pressed` call returns `false` and **the player cannot move**.
   **Fix:** *Project → Project Settings → Input Map* → add the four actions and
   bind them to the arrow keys or WASD.

3. **Directory nesting is redundant** —
   `dodge-the-creeps-godot-main/DTC/dodge-the-creeps/dodge-the-creeps/`. The
   Godot project root is the third level, with all assets in a fourth folder of
   the same name.

4. **Empty `_process` callbacks** remain in `Main.gd` and `mob.gd` (`pass`).
   They run every frame for no reason and should be deleted.

## Project structure

```
DTC/dodge-the-creeps/          ← Godot project root (project.godot)
├── project.godot
├── icon.svg
└── dodge-the-creeps/
    ├── Scenes/    Main.tscn, Player.tscn, Mob.tscn, HUD.tscn
    ├── Scripts/   Main.gd, Player.gd, mob.gd, Hud.gd
    ├── art/       sprites, House_In_a_Forest_Loop.ogg, gameover.wav
    └── fonts/
```

## Concepts demonstrated

Scene composition and instancing · signals for decoupled communication ·
`Area2D` vs `RigidBody2D` collision models · `PathFollow2D` for edge spawning ·
normalised vector movement · deferred physics state changes ·
`await` coroutines for timed sequences · automatic off-screen cleanup ·
timer-driven game loops

## Author

**Shehan Nimsara** — B.Sc. Software Design (International), TH Aschaffenburg
