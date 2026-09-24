# Milestone 1 — Foundation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Stand up the Godot 4 project with one controllable cat (placeholder art) that moves, collides with the environment, and is followed by a camera clamped to the level bounds, inside the first playable map (Casa A).

**Architecture:** A `CatCharacter` scene (`CharacterBody2D` root) holds movement/input logic in its own script, with a `Camera2D` child configured for loose-follow with a deadzone and explicit limits. `CasaA` is a `Node2D` scene containing a `TileMapLayer` for the floor/walls (placeholder tileset) and an instance of the cat scene. A `GameState` autoload singleton is registered now (empty of real state) so later milestones extend it instead of introducing it mid-project — this matches the spec's folder structure, which already reserves `scripts/autoload/game_state.gd`.

**Tech Stack:** Godot 4 (GDScript), no external plugins for this milestone.

**Spec:** `docs/superpowers/specs/2026-09-23-alice-beyond-the-gate-design.md`

## Global Constraints

- Engine: Godot 4 with GDScript (spec: "Engine e stack").
- Folder structure must match the spec's layout exactly: `scenes/`, `scripts/`, `resources/`, `assets/` at the project root (spec: "Estrutura de pastas").
- Camera follows loosely with a deadzone and is clamped to level bounds — an unclamped camera showing past the edge of the map is an explicit anti-pattern per `game-development` skill.
- Input must be bound via Godot's Input Map actions (`move_up`, `move_down`, `move_left`, `move_right`), never raw key checks — rebindability is a project-wide standard from the `game-development` skill, not just this milestone's preference.
- No automated test framework yet (GUT deferred to Milestone 4, per explicit decision — this milestone's verification is manual, running the project in the Godot editor).
- Placeholder art only — no final sprites. A `ColorRect` or a simple placeholder texture stands in for the cat and tileset (spec: "Fora de escopo" — final pixel art is long-term ambition, not vertical-slice-blocking).

## Review Focus

- **Diagonal movement speed**: moving up+right simultaneously must not be faster than a single direction — a reasonable player expects consistent speed in all directions, and unnormalized diagonal input is one of the most common 2D movement bugs.
- **Collision against walls**: the cat must not be able to walk through the placeholder wall tiles — this is the entire point of the milestone's collision work, and it is easy to wire a `TileMapLayer` with no actual collision shapes.
- **Camera never shows outside the map**: panning the cat to any corner of Casa A must not reveal empty space past the level edge — the spec and skill both call this out explicitly as something that breaks immersion.
- **Input holds vs. taps**: movement must respond continuously while a direction is held, not just on the initial key-down — a naive `Input.is_action_just_pressed` instead of `Input.is_action_pressed` would make the cat move one step per keystroke.
- **Project opens without errors**: a missing autoload path or a broken scene reference would silently break every later milestone that builds on this one — this needs an explicit check, not an assumption.

---

## File Structure

- `project.godot` — project settings: display resolution, Input Map actions, `GameState` autoload registration.
- `scripts/autoload/game_state.gd` — empty singleton, registered now so its path is stable for later milestones.
- `scripts/characters/cat_character.gd` — movement logic attached to the cat scene's root node.
- `scenes/characters/cat_character.tscn` — the cat scene: `CharacterBody2D` root, `CollisionShape2D`, placeholder `Sprite2D`/`ColorRect`, `Camera2D` child.
- `scenes/maps/casa_diana/casa_a.tscn` — the first playable map: `Node2D` root, `TileMapLayer` for floor and walls, a `Marker2D` for the cat's spawn point, an instance of `cat_character.tscn`.
- `assets/sprites/cat_placeholder.png` — a small placeholder image (or a `ColorRect` if no image asset is used — decided in Task 2).
- `assets/tilesets/placeholder_tileset.png` — a small placeholder tileset image for floor/wall tiles.

---

### Task 1: Godot project scaffold and folder structure

**Files:**
- Create: `project.godot`
- Create: `.gitignore`
- Create: `scenes/characters/.gitkeep`, `scenes/maps/casa_diana/.gitkeep`, `scenes/maps/casa_nuno/.gitkeep`, `scenes/maps/vizinhanca/.gitkeep`, `scenes/battle/.gitkeep`, `scenes/ui/.gitkeep`
- Create: `scripts/characters/.gitkeep`, `scripts/abilities/.gitkeep`, `scripts/battle/.gitkeep`, `scripts/dialogue/.gitkeep`, `scripts/autoload/.gitkeep`
- Create: `resources/characters/.gitkeep`, `resources/abilities/.gitkeep`, `resources/dialogue/.gitkeep`
- Create: `assets/sprites/.gitkeep`, `assets/tilesets/.gitkeep`, `assets/audio/.gitkeep`

**Interfaces:**
- Consumes: nothing (first task).
- Produces: the project root Godot recognizes (`project.godot`), and the folder skeleton every later task writes into, matching the spec's structure exactly.

- [ ] **Step 1: Create the folder skeleton with `.gitkeep` placeholders**

Godot does not need `.gitkeep` files to function, but git does not track empty directories — without them, `git status` after this task would show nothing, and the folders would vanish on a fresh clone.

Run:
```bash
mkdir -p scenes/characters scenes/maps/casa_diana scenes/maps/casa_nuno scenes/maps/vizinhanca scenes/battle scenes/ui
mkdir -p scripts/characters scripts/abilities scripts/battle scripts/dialogue scripts/autoload
mkdir -p resources/characters resources/abilities resources/dialogue
mkdir -p assets/sprites assets/tilesets assets/audio
touch scenes/characters/.gitkeep scenes/maps/casa_diana/.gitkeep scenes/maps/casa_nuno/.gitkeep scenes/maps/vizinhanca/.gitkeep scenes/battle/.gitkeep scenes/ui/.gitkeep
touch scripts/characters/.gitkeep scripts/abilities/.gitkeep scripts/battle/.gitkeep scripts/dialogue/.gitkeep scripts/autoload/.gitkeep
touch resources/characters/.gitkeep resources/abilities/.gitkeep resources/dialogue/.gitkeep
touch assets/sprites/.gitkeep assets/tilesets/.gitkeep assets/audio/.gitkeep
```

- [ ] **Step 2: Create `.gitignore` for Godot-specific artifacts**

```gitignore
# Godot 4+ specific ignores
.godot/
*.translation

# Imported translations (automatically generated from CSV files)
*.translation

# Mono-specific ignores
.mono/
data_*/

# System/tool-specific ignores
.DS_Store
```

- [ ] **Step 3: Open the project in Godot 4 editor to generate `project.godot`**

This step is manual — open the Godot 4 editor, choose "Import", select this directory, and create a new project. Godot writes `project.godot` and a `.godot/` cache folder (already gitignored). Confirm `project.godot` exists at the repo root before continuing.

Verify: `test -f project.godot && echo "project.godot exists"`
Expected: `project.godot exists`

- [ ] **Step 4: Commit the scaffold**

```bash
git add project.godot .gitignore scenes scripts resources assets
git commit -m "Scaffold Godot 4 project with spec folder structure"
```

---

### Task 2: Input Map configuration

**Files:**
- Modify: `project.godot` (via Godot editor's Project Settings > Input Map)

**Interfaces:**
- Consumes: nothing.
- Produces: four input actions — `move_up`, `move_down`, `move_left`, `move_right` — that Task 4's movement script reads by name. Any later task needing directional input reads these same four action names; do not introduce raw key checks anywhere in this project.

- [ ] **Step 1: Add four Input Map actions in the Godot editor**

Open Project > Project Settings > Input Map. Add these actions with these default key bindings:
- `move_up` → `W` and `Up Arrow`
- `move_down` → `S` and `Down Arrow`
- `move_left` → `A` and `Left Arrow`
- `move_right` → `D` and `Right Arrow`

Binding actions instead of raw keys means rebinding later (or adding gamepad support) is a config change, not a code change — this is a standing project convention, not specific to this task.

- [ ] **Step 2: Verify the actions were written to `project.godot`**

Run: `grep -A2 "move_up\|move_down\|move_left\|move_right" project.godot`
Expected: four `[input]` entries, one per action, each listing at least one `InputEventKey`.

- [ ] **Step 3: Commit**

```bash
git add project.godot
git commit -m "Add directional Input Map actions"
```

---

### Task 3: Cat character scene and movement script

**Files:**
- Create: `scripts/characters/cat_character.gd`
- Create: `scenes/characters/cat_character.tscn`

**Interfaces:**
- Consumes: `move_up`/`move_down`/`move_left`/`move_right` Input Map actions from Task 2.
- Produces: a `CatCharacter` scene instantiable from any map scene, exposing a `speed: float` export and moving via `CharacterBody2D.move_and_slide()`. Task 5 instances this scene directly; no other task needs to know its internals beyond that it is a `CharacterBody2D`.

- [ ] **Step 1: Write the movement script**

```gdscript
# scripts/characters/cat_character.gd
extends CharacterBody2D

@export var speed: float = 120.0

func _physics_process(_delta: float) -> void:
	var input_direction := Vector2(
		Input.get_action_strength("move_right") - Input.get_action_strength("move_left"),
		Input.get_action_strength("move_down") - Input.get_action_strength("move_up")
	)
	velocity = input_direction.normalized() * speed
	move_and_slide()
```

`Vector2.normalized()` here is what keeps diagonal movement from being faster than cardinal movement — without it, `(1, 1)` has a length of ~1.41, so the cat would move ~41% faster moving diagonally than moving straight. `move_and_slide()` is `CharacterBody2D`'s built-in method that applies `velocity` for one physics frame and resolves collisions against anything with a `CollisionShape2D`, which is why the wall collision in Task 5 needs no code of its own here — the engine handles it once both bodies have shapes.

- [ ] **Step 2: Build the scene in the Godot editor**

Create a new scene with a `CharacterBody2D` as the root node, named `CatCharacter`. Attach `scripts/characters/cat_character.gd` to it. Add these children:
- A `CollisionShape2D` with a small `CircleShape2D` or `RectangleShape2D` (roughly cat-sized, e.g. 16x16px) — this defines what the cat's body physically occupies for collision purposes.
- A `Sprite2D` (or `ColorRect` if no placeholder image is used) sized to roughly match the collision shape, so the visible cat and its solid area line up — a hitbox visibly larger or smaller than the sprite reads as unfair or confusing even when technically correct.

Save as `scenes/characters/cat_character.tscn`.

- [ ] **Step 3: Verify the scene has no orphan nodes or missing script errors**

Open `cat_character.tscn` in the Godot editor. Check the Scene panel for warning icons.

Verify: no red/yellow warning icons appear on any node in the Scene dock.
Expected: clean scene tree, script icon shown on the root `CatCharacter` node.

- [ ] **Step 4: Commit**

```bash
git add scripts/characters/cat_character.gd scenes/characters/cat_character.tscn
git commit -m "Add cat character scene with directional movement"
```

---

### Task 4: Casa A map with collision

**Files:**
- Create: `scenes/maps/casa_diana/casa_a.tscn`
- Create: `assets/tilesets/placeholder_tileset.png` (or equivalent placeholder asset)

**Interfaces:**
- Consumes: `scenes/characters/cat_character.tscn` from Task 3 (instanced as a child).
- Produces: a `Node2D`-rooted map scene containing a `TileMapLayer` with physics collision on wall tiles, and a `Marker2D` named `CatSpawn` marking where the cat instance starts. Task 6 (camera limits) reads this scene's `TileMapLayer` bounds to compute camera clamp values.

- [ ] **Step 1: Create a placeholder tileset**

Create a small placeholder image (e.g. two 16x16 tiles side by side: one plain floor color, one wall color) at `assets/tilesets/placeholder_tileset.png`. This can be created with any image tool or Godot's built-in tools — the only requirement is two visually distinct tiles.

- [ ] **Step 2: Build the Casa A scene in the Godot editor**

Create a new scene with a `Node2D` root named `CasaA`. Add a `TileMapLayer` child, assign it a new `TileSet` resource built from `placeholder_tileset.png`, and register both tiles (floor, wall).

Paint a small enclosed room: a rectangular floor area (e.g. 15x10 tiles) fully bordered by wall tiles on all four edges. This is the tutorial room described in the spec's "Mapas do vertical slice" section.

For the wall tile specifically: in the TileSet resource, open the wall tile's physics layer and draw a collision polygon covering the full tile. Floor tiles get no collision shape — only wall tiles are solid. This collision step is what Review Focus item 2 ("Collision against walls") verifies; a `TileMapLayer` with tiles painted but no physics layer configured looks correct visually but has no actual collision.

- [ ] **Step 3: Add the spawn marker and cat instance**

Add a `Marker2D` child named `CatSpawn`, positioned in roughly the center of the room. Instance `scenes/characters/cat_character.tscn` as a child of `CasaA`, and set its position to match `CatSpawn`'s position.

Save as `scenes/maps/casa_diana/casa_a.tscn`.

- [ ] **Step 4: Manually verify collision works**

Set `casa_a.tscn` as the project's main scene (Project > Project Settings > Application > Run > Main Scene), then run the project (F5).

Verify: the cat moves freely inside the room in all four directions using WASD/arrows, and stops at every wall tile — it cannot cross into or past a wall tile from any approach angle (try approaching a wall straight-on and diagonally).
Expected: cat is fully contained within the room; no clipping through walls.

- [ ] **Step 5: Manually verify diagonal movement speed**

With the project still running, move the cat in a straight line along one axis (e.g. only right) for two seconds, noting the distance covered. Then move diagonally (e.g. right+down held together) for two seconds.

Verify: diagonal travel distance looks the same as single-axis travel distance, not visibly faster.
Expected: consistent speed regardless of direction — this confirms the `normalized()` call in Task 3 is working as intended.

- [ ] **Step 6: Commit**

```bash
git add scenes/maps/casa_diana/casa_a.tscn assets/tilesets/placeholder_tileset.png
git commit -m "Add Casa A map with wall collision and cat spawn point"
```

---

### Task 5: Camera with deadzone follow and level bounds

**Files:**
- Modify: `scenes/characters/cat_character.tscn` (add `Camera2D` child)

**Interfaces:**
- Consumes: `TileMapLayer` bounds from Task 4's `casa_a.tscn` to set `limit_left`/`limit_right`/`limit_top`/`limit_bottom`.
- Produces: nothing further consumed by later tasks in this milestone — camera limits are set per-map going forward, a pattern later milestones (Casa B, etc.) repeat rather than import.

- [ ] **Step 1: Add a `Camera2D` child to the cat scene**

Open `scenes/characters/cat_character.tscn` in the Godot editor. Add a `Camera2D` as a child of the `CatCharacter` root. Enable it (`Enabled` property checked — Godot only allows one active camera at a time, and a newly added `Camera2D` defaults to enabled, which is what's wanted here since this is the only camera in the project so far).

- [ ] **Step 2: Configure deadzone follow**

On the `Camera2D` node, set:
- `Position Smoothing > Enabled`: on
- `Position Smoothing > Speed`: `5.0` (a starting value — loose enough to lag slightly behind fast movement, which is the "loose follow" feel called for in the spec's Exploration section)
- `Drag Horizontal Enabled` and `Drag Vertical Enabled`: on
- `Drag Horizontal Offset` / `Drag Vertical Offset`: leave at default `0`
- `Drag Margin` (all four sides): `0.2` — this creates the deadzone: the camera only starts moving once the cat crosses 20% of the way to the screen edge, rather than tracking the cat's exact pixel position every frame.

- [ ] **Step 3: Set camera limits to Casa A's room bounds**

Open `scenes/maps/casa_diana/casa_a.tscn`, select the `TileMapLayer`, and note the pixel-space bounding rectangle of the painted room (tile count × tile size, e.g. a 15x10 room with 16px tiles is 240x160 pixels, offset by wherever the room's top-left tile sits in the scene).

Back in `cat_character.tscn`, set on `Camera2D`:
- `Limit Left`, `Limit Top`, `Limit Right`, `Limit Bottom` to those computed pixel bounds.

This is what Review Focus item 3 ("Camera never shows outside the map") depends on — without explicit limits, `Camera2D` has no bounds at all and will happily show empty space past any edge of the room.

- [ ] **Step 4: Manually verify camera stays within bounds**

Run the project (F5) with `casa_a.tscn` as the main scene. Walk the cat into all four corners of the room.

Verify: at every corner, the visible screen area still shows only the room (floor/walls) — no gray/black empty space past the tilemap's edge in any direction.
Expected: camera view clamps cleanly at each edge.

- [ ] **Step 5: Manually verify continuous movement on held input**

With the project running, press and hold a single direction key (e.g. `D`) for two full seconds without releasing.

Verify: the cat moves continuously the entire time the key is held, not just once per press.
Expected: smooth continuous movement — this confirms `Input.get_action_strength()` (polled every physics frame in `_physics_process`) is being used correctly rather than an edge-triggered check like `is_action_just_pressed`.

- [ ] **Step 6: Commit**

```bash
git add scenes/characters/cat_character.tscn
git commit -m "Add camera with deadzone follow and Casa A bounds"
```

---

### Task 6: GameState autoload registration

**Files:**
- Create: `scripts/autoload/game_state.gd`
- Modify: `project.godot` (Autoload registration)

**Interfaces:**
- Consumes: nothing.
- Produces: a globally accessible `GameState` singleton (accessible from any script as `GameState.xxx`), currently empty. Milestone 3+ will add real properties here (unlocked roster, act progress, story flags) — this task's only job is to reserve the path and prove the autoload wires up cleanly, matching the spec's `autoload/game_state.gd` folder entry.

- [ ] **Step 1: Write the empty singleton script**

```gdscript
# scripts/autoload/game_state.gd
extends Node

# Populated in later milestones: unlocked roster, act progress, story flags.
```

An autoload is Godot's mechanism for state that needs to survive scene changes and be reachable from anywhere — the save file, a scene-transition manager, or (starting in a later milestone) which cats are unlocked. It is registered once here, empty, specifically so nothing later has to retrofit the registration — every future write to this file is additive.

- [ ] **Step 2: Register the autoload in Project Settings**

In the Godot editor: Project > Project Settings > Autoload. Add `scripts/autoload/game_state.gd`, name it `GameState`, and confirm it's enabled.

- [ ] **Step 3: Verify the autoload registration was written to `project.godot`**

Run: `grep -A1 "\[autoload\]" project.godot`
Expected: an `[autoload]` section containing a line referencing `GameState="*res://scripts/autoload/game_state.gd"`.

- [ ] **Step 4: Verify the project runs without errors with the new autoload**

Run the project (F5) with `casa_a.tscn` as the main scene. Check the Godot editor's Output/Debugger panel.

Verify: no red error text referencing `game_state.gd` or `GameState` appears in the Output panel on launch.
Expected: clean launch, same behavior as Task 5's verification (movement, collision, camera all still work).

- [ ] **Step 5: Commit**

```bash
git add scripts/autoload/game_state.gd project.godot
git commit -m "Register empty GameState autoload singleton"
```

---

## Milestone 1 Definition of Done

All six tasks committed, and running the project (F5) from `casa_a.tscn` shows: a cat that moves in four directions at consistent speed via held WASD/arrow input, cannot pass through wall tiles from any angle, is followed by a camera that never reveals space outside the room, and the `GameState` autoload is registered with no startup errors.
