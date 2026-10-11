---
title: "COMP 4300 Notes: Building a 2D Game Engine in C++"
description: "My notes on Dave Churchill's COMP 4300 game programming course, grouped by topic: ECS, scenes and actions, collision, vision and AI, cameras, tools, graphics, performance and the final project."
tags:
  - Game Programming
  - C++
cover: /assets/images/posts/comp4300/cover-mosaic.jpg
cover_alt: "The four COMP 4300 reference games: bouncing shapes with an ImGui panel, a Geometry Wars clone, the Mega Mario platformer and a top-down Zelda-like game."
thumbnail: /assets/images/thumbnails/comp4300.jpg
---

I finished Dave Churchill's [COMP 4300](https://www.youtube.com/playlist?list=PL_xRyXins848Mfiv4hIgiMZgLpzFT7ohv) several months ago. It is a game programming course at Memorial University (the recordings are from Fall 2025) in which you build a small 2D engine in C++23 on top of SFML 3 and Dear ImGui step by step, through four assignments (bouncing shapes, a Geometry Wars clone, a Mega Man platformer and a Zelda-like) and a group project.

Lectures and assignments are how the course is organized. These notes are grouped by topic instead, with links to the lectures behind each one. Every screenshot is a frame from Churchill's own lecture demos.

<!--more-->

<style>
.c43-fig { margin: 1.75rem 0; font-size: 0.86rem; line-height: 1.45; }
.c43-fig code { font-size: 0.95em; }
.c43-fig strong { display: block; font-size: 0.95rem; }
.c43-sub { color: var(--muted); font-weight: 400; }
.c43-box { border: 1.5px solid; border-radius: 8px; padding: 0.65rem 0.8rem; min-width: 0; overflow-wrap: anywhere; }
.c43-blue { background: #e8eefb; border-color: #7d9bd1; }
.c43-scene { background: #f1f4fc; border-color: #7d9bd1; }
.c43-data { background: #fcf3e0; border-color: #d2ac5c; }
.c43-sys { background: #e7f3ea; border-color: #78ad87; align-self: start; }
.c43-res { background: #fafaf8; border-color: #b9b9b2; }
.c43-chips { display: flex; flex-wrap: wrap; gap: 0.35rem; margin-top: 0.45rem; }
.c43-chips span { padding: 0.1rem 0.6rem; border: 1px solid #b4c4e4; border-radius: 999px; background: #fff; }
.c43-data .c43-chips span { border-color: #dcc189; }
.c43-link { margin: 0.35rem 0; color: var(--muted); text-align: center; }
.c43-top { display: grid; grid-template-columns: 1.7fr 1fr; gap: 0.6rem; }
.c43-cols { display: grid; grid-template-columns: 1fr 1fr; gap: 0.6rem; margin-top: 0.4rem; }
.c43-sysrow { display: grid; grid-template-columns: 6.2rem 1fr; gap: 0.5rem; margin-top: 0.3rem; }
.c43-sysrow b { font-weight: 700; }
.c43-flow { display: flex; align-items: stretch; gap: 0.3rem; }
.c43-flow .c43-box { flex: 1 1 0; }
.c43-arrow { display: flex; flex-direction: column; align-items: center; justify-content: center; min-width: 3.4rem; color: var(--muted); font-size: 0.78rem; text-align: center; }
.c43-arrow::after { content: "\2192"; font-size: 1.1rem; line-height: 1; }
.c43-branches { display: grid; grid-template-columns: repeat(3, 1fr); gap: 0.6rem; margin-top: 0.7rem; }
.c43-data-row { display: grid; grid-template-columns: 1fr 5.5rem 1fr; gap: 0.5rem; align-items: stretch; margin-top: 0.5rem; }
.c43-data-row:first-child { margin-top: 0; }
.c43-data-row .c43-arrow { min-width: 0; }
.c43-data-row .c43-arrow::after { content: "\2192"; }
.c43-data-row .c43-both::after { content: "\2194"; }
.c43-pair { display: grid; grid-template-columns: 1fr 1fr; gap: 0.6rem; }
.c43-pair img { width: 100%; height: auto; margin: 0 !important; }
@media (max-width: 640px) {
  .c43-top, .c43-cols, .c43-branches, .c43-pair { grid-template-columns: 1fr; }
  .c43-flow { flex-direction: column; }
  .c43-data-row { grid-template-columns: 1fr; gap: 0.2rem; }
  .c43-data-row .c43-arrow::after { content: "\2193"; }
  .c43-data-row .c43-both::after { content: "\2195"; }
  .c43-arrow { flex-direction: row; gap: 0.4rem; min-width: 0; padding: 0.1rem 0; }
  .c43-arrow::after { content: "\2193"; }
  .c43-sysrow { grid-template-columns: 1fr; gap: 0; }
  .c43-sysrow b { margin-top: 0.2rem; }
}
@media print { .c43-fig, .c43-fig2, .c43-box { break-inside: avoid; } }
</style>

## Keep data and logic apart {#ecs}

The rule Churchill singles out as the most important one to remember is to keep logic and data apart ([L5](https://youtu.be/1B3p2DOkso4?t=30)). In ECS terms an entity is an identity, a component is plain data, and a system is a function that runs over every entity that has the components it needs ([L4](https://youtu.be/8_DeUkjQcSU)). A `CTransform` stores a position and velocity and does nothing; `sMovement` is the only code that moves things.

[L4](https://youtu.be/8_DeUkjQcSU) compares five ways to store components in C++ and ends on a `std::tuple` of component types inside `Entity`, with `get<T>()`, `has<T>()`, `add<T>()` and `remove<T>()`: contiguous, no allocation per component, looked up at compile time. The `EntityManager` keeps shared pointers in a vector plus a map from tag to entities, so "all the bullets" is one lookup. It also delays adds and removes: new entities wait in `m_toAdd`, `destroy()` only clears `m_alive`, and both take effect at the start of the next update, because changing a vector while a system iterates over it invalidates the iterators ([L5 at 28:00](https://youtu.be/1B3p2DOkso4?t=1680)).

<figure style="margin: 1.75rem 0;">
  <img src="/assets/images/posts/comp4300/a2-geometry-wars.jpg" alt="Assignment 2 reference solution: dozens of colored polygons on a dark checkered background, a score in the top-left corner, and an ImGui window with checkboxes for the movement, lifespan, collision, spawning, GUI and rendering systems." style="margin: 0 auto;">
  <figcaption markdown="span" style="margin-top: 0.6rem; color: #666; font-size: 0.85rem; line-height: 1.5;">Assignment 2's debug panel switches each system off and lists entities by tag. With movement and lifespan off, bullets freeze in place and never expire, which shows what each system owns ([L6 at 5:06](https://youtu.be/DcMiPoNOmHg?t=306)). Frame from the COMP 4300 lecture video by David Churchill.</figcaption>
</figure>

## An engine made of scenes, assets and actions {#engine}

From Assignment 3 on, the single `Game` class of Assignment 2 splits in two ([L8 at 4:34](https://youtu.be/L_LivvW-8nk?t=274)). `GameEngine` owns what is shared: the window, the loaded `Assets`, input and a map from names to scenes. Each `Scene` (menu, play, Zelda) owns an `EntityManager`, an action map and its own systems, and only the current scene is updated. Opening an inventory screen therefore leaves the gameplay scene alive but paused, and closing it resumes where it left off; Churchill likens the scene map to a finite state machine.

Two habits come with this design. Assets are loaded once and shared, so a thousand birds use one texture. And gameplay code never sees a key: the engine maps keys to action names per scene, and an `Action` is a name plus `START` or `END` ([L9](https://youtu.be/lc7xGct2BYY)). Because actions are tiny, a replay is just a list of frames and actions, which is how Doom demo files worked ([L9 at 43:00](https://youtu.be/lc7xGct2BYY?t=2580)), and an AI can search over the same actions.

<figure class="c43-fig">
<div class="c43-top">
<div class="c43-box c43-blue"><strong>GameEngine</strong><div>window · main loop · input handling</div><div>scenes: <code>map&lt;string, shared_ptr&lt;Scene&gt;&gt;</code></div><div><code>changeScene(name, args)</code></div></div>
<div class="c43-box c43-res"><strong>Assets</strong><div>textures · fonts · animations · sounds</div><div class="c43-sub">owned by the engine; load once, use often</div></div>
</div>
<div class="c43-link">&darr; keyboard and mouse arrive here; only the current scene runs <code>sDoAction(Action)</code>, <code>update()</code> and <code>sRender()</code></div>
<div class="c43-box c43-scene"><strong>Scene <span class="c43-sub">abstract base class</span></strong><div>owns an <code>EntityManager</code>, an action map and a frame counter</div><div class="c43-chips"><span>Scene_Menu</span><span>Scene_Play</span><span>Scene_Zelda</span></div></div>
<div class="c43-link">&darr; owns &nbsp;&nbsp;&nbsp;&nbsp; &darr; runs</div>
<div class="c43-cols">
<div>
<div class="c43-box c43-data"><strong>EntityManager</strong><div><code>vector&lt;shared_ptr&lt;Entity&gt;&gt;</code> plus a map from tag to entities</div><div class="c43-sub">addEntity(tag) · adds and removals are delayed</div></div>
<div class="c43-link">&darr; stores</div>
<div class="c43-box c43-data"><strong>Entity</strong><div>id · tag · alive · a <code>std::tuple</code> of components</div><div class="c43-chips"><span>CTransform</span><span>CAnimation</span><span>CInput</span><span>CBoundingBox</span><span>CHealth</span><span>CGravity</span></div><div class="c43-sub" style="margin-top: 0.45rem;">components are plain data, no behavior</div></div>
</div>
<div class="c43-box c43-sys"><strong>Systems <span class="c43-sub">read and write components</span></strong>
<div class="c43-sysrow"><b>sDoAction</b><span>action to <code>CInput</code> flags</span></div>
<div class="c43-sysrow"><b>sMovement</b><span>velocity, gravity</span></div>
<div class="c43-sysrow"><b>sAI</b><span>patrol, follow, vision</span></div>
<div class="c43-sysrow"><b>sCollision</b><span>AABB detect and resolve</span></div>
<div class="c43-sysrow"><b>sAnimation</b><span>advance sprite frames</span></div>
<div class="c43-sysrow"><b>sStatus</b><span>lifespan, invincibility</span></div>
<div class="c43-sysrow"><b>sCamera</b><span>room or follow view</span></div>
<div class="c43-sysrow"><b>sRender</b><span>entities first, GUI last</span></div>
</div>
</div>
<figcaption markdown="span" style="margin-top: 0.6rem; color: #666; font-size: 0.85rem; line-height: 1.5;">Runtime structure of the assignment engine. System names follow the course's <code>sName</code> convention; which systems exist depends on the scene.</figcaption>
</figure>

## Collision, gravity and movement {#collision}

Collision is two problems, detection and resolution ([L7](https://youtu.be/9QgaLWBkv0s)). For axis-aligned bounding boxes (AABB) the overlap on one axis is `(w1 + w2) / 2 - |x1 - x2|`, and the boxes collide when both axes overlap. The side of the hit comes from the previous frame: if the boxes already overlapped vertically then, the new overlap is horizontal, and the other way round. That is how a brick hit from below breaks while one hit from above is stood on ([L7 at 39:00](https://youtu.be/9QgaLWBkv0s?t=2340)).

Platforming adds little on top ([L10](https://youtu.be/eq8nvzVosIY)). Gravity is an acceleration added to velocity every frame, a jump is an initial upward velocity, and releasing the key cuts it for variable jump height. Speed has to stay below the thinnest obstacle per update or objects tunnel through it; when that is not enough, run several updates per rendered frame, as Super Mario 64 does with four ([L19 at 50:00](https://youtu.be/VFvJLpVkpzk?t=3000)).

<figure style="margin: 1.75rem 0;">
  <img src="/assets/images/posts/comp4300/a3-mega-mario.jpg" alt="Assignment 3 reference solution: a Mega Man sprite standing on a brick floor in a Super Mario style level with question blocks, a green pipe, clouds and a hill." style="margin: 0 auto;">
  <figcaption markdown="span" style="margin-top: 0.6rem; color: #666; font-size: 0.85rem; line-height: 1.5;">Assignment 3 ([L10 at 3:37](https://youtu.be/eq8nvzVosIY?t=217)). Solid tiles carry bounding boxes, decorations such as clouds and bushes do not, and a debug toggle draws the boxes the physics system sees ([L10 at 11:00](https://youtu.be/eq8nvzVosIY?t=660)). Frame from the COMP 4300 lecture video by David Churchill.</figcaption>
</figure>

## Line of sight, lighting and pathfinding {#vision}

Seeing is segment intersection ([L12](https://youtu.be/mcctcfGWTWE)). Write a ray as `A + t·r` and an edge as `C + u·s`; with the 2D cross product, `t = ((C - A) × s) / (r × s)` and `u = ((C - A) × r) / (r × s)`, and the segments cross when both are in `[0, 1]` ([L12 at 27:00](https://youtu.be/mcctcfGWTWE?t=1620)). A rectangle is four such edges. In Assignment 4 an NPC follows the player only while no vision-blocking tile intersects the line between them, and otherwise returns home or patrols its waypoints ([L15](https://youtu.be/Ruy7P0NAvZU)).

Lighting reuses the same test. Evenly spaced rays leak light past corners; casting three rays at every vertex (one at it, two nudged either side), sorting them by angle and drawing an `sf::TriangleFan` gives a clean visibility polygon ([L12 at 46:00](https://youtu.be/mcctcfGWTWE?t=2760)).

Getting somewhere is a separate problem ([L14](https://youtu.be/0OfenyWsE2Y)). Search runs on a cheap representation (a grid, or a navmesh of convex polygons) and the path is refined with line-of-sight checks. When many agents share one goal, a vector field replaces per-agent search: a breadth-first distance map from the goal and an arrow to the lowest neighbour. Steering handles the last stretch: `desired = normalize(target - pos) * maxSpeed`, `steering = desired - velocity`, with arrival scaling the speed down near the target.

<figure class="c43-fig2">
<div class="c43-pair">
<img src="/assets/images/posts/comp4300/lighting-visibility.jpg" alt="A visibility demo: gray polygons on a black background with a yellow light source and a light-gray region showing everything the light can see.">
<img src="/assets/images/posts/comp4300/flow-field.jpg" alt="A flow-field demo: about 20,000 white and orange particles streaming around green walls over a blue distance map.">
</div>
<figcaption markdown="span" style="margin-top: 0.6rem; color: #666; font-size: 0.85rem; line-height: 1.5;">Left: Churchill's visibility demo, a light source and everything it can see ([L12 at 9:22](https://youtu.be/mcctcfGWTWE?t=562)). Right: about 20,000 particles routed around walls by a distance map in real time ([L14 at 48:12](https://youtu.be/0OfenyWsE2Y?t=2892)). Frames from the COMP 4300 lecture videos by David Churchill.</figcaption>
</figure>

## Cameras and views {#cameras}

A 2D camera is just the rectangle of the world that gets drawn ([L13](https://youtu.be/l4bOT8-JWM8)). In SFML that is an `sf::View`: set its center and size, call `window.setView`, draw the world, then switch back to the default view to draw the HUD at fixed window coordinates. Camera behaviors are small variations on that: lock on the player, smoothed follow, a trap box, directional scrolling. Assignment 3 keeps Mega Man centered once he passes the middle of the screen; Assignment 4 shows one room at a time, where the room index is integer division of position by window size, with a follow camera you can toggle ([L15](https://youtu.be/Ruy7P0NAvZU)). Viewports, given as fractions of the window, give split screen and minimaps.

<figure style="margin: 1.75rem 0;">
  <img src="/assets/images/posts/comp4300/a4-zelda.jpg" alt="Assignment 4 reference solution: a Link-like player with a health bar, a heart pickup, a tree-covered cave entrance and an orange enemy with its own health bar on a sand-colored background." style="margin: 0 auto;">
  <figcaption markdown="span" style="margin-top: 0.6rem; color: #666; font-size: 0.85rem; line-height: 1.5;">Assignment 4 draws one 1280 by 768 room of 64 pixel tiles at a time, with a health bar over the player and each enemy ([L15 at 3:12](https://youtu.be/Ruy7P0NAvZU?t=192)). Frame from the COMP 4300 lecture video by David Churchill.</figcaption>
</figure>

## Data-driven design and tools {#tools}

Almost nothing is hard-coded. Assignment 1 reads `Window`, `Font`, `Circle` and `Rectangle` lines from a text file ([L2](https://youtu.be/3WWjsTPdlag)); later assets and levels follow the same pattern (`Texture`, `Animation`, tiles placed in 64 by 64 grid coordinates), and the project requires levels, player and options in external text files.

Dear ImGui makes tools cheap ([L3](https://youtu.be/bEaNm9YSXn0)). It is immediate-mode: the UI is redeclared every frame and widgets edit your variables in place, so a panel to tune a shape, toggle systems, list entities or draw collision boxes is a few lines. Assignment 4 goes one step further with an inspector: click an entity and draw a section for each component it has. That inspector is the first half of a level editor ([L16](https://youtu.be/xCoe5bUka_A)).

The editor itself is a separate scene that ignores physics and AI. Left click selects, right click toggles dragging, positions snap to the grid, and each component the entity lacks gets an add button. Saving writes a temporary file and renames it over the old one so a crash cannot corrupt the save; because components are plain data, serializing them is just writing them out.

<figure class="c43-fig2">
<div class="c43-pair">
<img src="/assets/images/posts/comp4300/a1-bouncing-shapes.jpg" alt="Assignment 1 reference solution: colored circles and rectangles with names drawn at their centers, and an ImGui Shape Properties panel with a shape drop-down open.">
<img src="/assets/images/posts/comp4300/level-editor.jpg" alt="The level editor from lecture 16: a grid with cell coordinates, a Link sprite with a health bar, and an ImGui Selected Entity inspector showing transform, health, damage, bounding box and input components.">
</div>
<figcaption markdown="span" style="margin-top: 0.6rem; color: #666; font-size: 0.85rem; line-height: 1.5;">Left: Assignment 1's ImGui panel edits a shape's visibility, scale, velocity, color and name ([L2 at 15:05](https://youtu.be/3WWjsTPdlag?t=905)). Right: the level editor Churchill demonstrates in Lecture 16, with a grid and a Selected Entity inspector ([L16 at 58:30](https://youtu.be/xCoe5bUka_A?t=3510)). Frames from the COMP 4300 lecture videos by David Churchill.</figcaption>
</figure>

## Animation, shaders and particles {#graphics}

A texture holds pixels on the GPU, and a sprite is a cheap drawable that points at one ([L8](https://youtu.be/L_LivvW-8nk)). A sprite sheet packs frames into one image and `setTextureRect` picks one; an `Animation` advances with `currentFrame = (gameFrame / speed) % frameCount`, and facing left is the same animation drawn with `scale.x = -1`.

Shaders ([L18](https://youtu.be/7n0jtNgTMn0)) are fragment programs that run once per pixel on the GPU. They receive uniforms such as time and resolution and work in normalized 0 to 1 coordinates. In SFML you load an `sf::Shader` and pass it to `window.draw(sprite, &shader)`, and shaders can be edited and reloaded without recompiling the game; the project asks for three.

Particles ([L21](https://youtu.be/ws4IXeUN_m8)) are the one place ECS is the wrong tool: hundreds of thousands of entities waste memory on components they never use. A `std::vector<Particle>` plus one `sf::VertexArray` of quads draws everything in a single call.

<figure class="c43-fig2">
<div class="c43-pair">
<img src="/assets/images/posts/comp4300/shaders-fade-and-tint.jpg" alt="The same Mega Man sprite twice on a gray background: faded and washed out on the left, tinted pink with red outlines on the right.">
<img src="/assets/images/posts/comp4300/particles.jpg" alt="About 100,000 small pink and white particles forming a dense circular cloud on a black background.">
</div>
<figcaption markdown="span" style="margin-top: 0.6rem; color: #666; font-size: 0.85rem; line-height: 1.5;">Left: one Mega Man sprite under two fragment shaders, a fade and a red tint ([L18 at 60:00](https://youtu.be/7n0jtNgTMn0?t=3600)). Right: 100,000 particles drawn from one vertex array at 60 fps ([L21 at 27:30](https://youtu.be/ws4IXeUN_m8?t=1650)). Frames from the COMP 4300 lecture videos by David Churchill.</figcaption>
</figure>

## Performance: caches, loops and measuring {#performance}

The performance lectures are about hardware, not Big-O. [Lecture 17](https://youtu.be/WFMmE3i3VpY) starts from the fact that a cache line is about 64 bytes ([L17 at 22:30](https://youtu.be/WFMmE3i3VpY?t=1350)): entities scattered across the heap cost cache misses, and `malloc` and `free` during gameplay are slow. The fix is a pool allocated up front (100,000 entities in the example, [L17 at 18:30](https://youtu.be/WFMmE3i3VpY?t=1110)) in which an entity is just an index into a tuple of component vectors, which also suits systems that read only one or two component types.

[L19](https://youtu.be/VFvJLpVkpzk) covers the main loop. Tying the simulation to the frame rate makes a game run at different speeds on a 60 Hz and a 240 Hz monitor. A fixed-step loop instead accumulates elapsed time and runs as many fixed updates as are owed before it renders, with interpolation if needed, so the physics stays deterministic.

[L20](https://youtu.be/pL6Ba665uiQ) closes the loop with measurement. A RAII timer plus a macro that compiles away writes `chrome://tracing` JSON. On the platformer it showed that creating the SFML window took longer than loading every asset ([L20 at 72:30](https://youtu.be/pL6Ba665uiQ?t=4350)), that fonts were the slow assets with a 42 KB font the slowest ([L20 at 75:30](https://youtu.be/pL6Ba665uiQ?t=4530)), and that the game's own update took about a millisecond of each 60 fps frame while SFML waited on the buffer swap for the rest ([L20 at 82:00](https://youtu.be/pL6Ba665uiQ?t=4920)).

## The final project: where the topics meet {#project}

The final project ([L11](https://youtu.be/eOduYjeKQ5Y)) is a group assignment for up to four people: build an original game on the same engine, in C++ with ECS, using only SFML and ImGui (Box2D is out). Assignment code can be reused, but assets and levels cannot, and the gameplay has to differ significantly from the assignments. It is worth 50% of the course and has to be passed to pass the course. The [specification](https://github.com/davechurchill/COMP4300_Project) reads like a checklist of the sections above:

- **Scenes and progression** ([engine](#engine)): main menu, overworld map with locked levels, gameplay, inventory, level editor and game over; save and load; an options menu with volume, difficulty and key rebinding.
- **Mechanics** ([ECS](#ecs), [collision](#collision)): two movement abilities with a cooldown or cost, three swappable weapons (one using ammo), three NPC types with basic AI, hit points with invincibility frames, three items with an inventory UI, three status effects, moving tiles, gravity.
- **Seeing and thinking** ([vision](#vision)): ray casting for visibility, ray-cast lighting, pathfinding or steering.
- **Presentation** ([cameras](#cameras), [graphics](#graphics)): two camera views, parallax backgrounds, three shaders, a polished HUD instead of ImGui debug UI, music and sound effects.
- **Tools** ([tools](#tools)): a grid-based ImGui editor that places textures and edits entity parameters (blocks vision, hit points, patrol points). At least three levels plus a separate boss level. Levels, player and configuration are defined in external text files, and every level except the boss battle has to be buildable in the editor.
- **One extra**: a major mechanic outside the list, worth 10% of the project mark.

The project mark is split into a proposal (5%), a short gameplay demo video (10%), the code and gameplay (60%), a 15 to 20 minute report video (20%) and a trailer (5%). The report has to demo every mechanic with a full level playthrough and the editor, and list what did not work.

Two diagrams summarize how the project fits together. First, the scenes it has to ship and how a player moves between them:

<figure class="c43-fig">
<div class="c43-flow">
<div class="c43-box c43-scene"><strong>Main menu</strong><span class="c43-sub">entry point</span></div>
<div class="c43-arrow">play</div>
<div class="c43-box c43-scene"><strong>Overworld map</strong><span class="c43-sub">level select, locks</span></div>
<div class="c43-arrow">select</div>
<div class="c43-box c43-scene"><strong>Gameplay</strong><span class="c43-sub">levels from files</span></div>
<div class="c43-arrow">boss beaten</div>
<div class="c43-box c43-scene"><strong>Game over</strong><span class="c43-sub">credits</span></div>
</div>
<div class="c43-branches">
<div class="c43-box c43-res"><strong>Options</strong><span class="c43-sub">from the main menu: audio, difficulty, key bindings</span></div>
<div class="c43-box c43-data"><strong>Level editor</strong><span class="c43-sub">from the main menu: edits the level files that gameplay loads</span></div>
<div class="c43-box c43-res"><strong>Inventory</strong><span class="c43-sub">from gameplay: items and status effects</span></div>
</div>
<figcaption markdown="span" style="margin-top: 0.6rem; color: #666; font-size: 0.85rem; line-height: 1.5;">The scenes the specification requires and how a player moves between them. Clearing a level returns to the overworld, which unlocks the next level and saves progress.</figcaption>
</figure>

Second, the data behind them. Levels, assets and configuration live in external text files, so the level editor edits the same level files that the gameplay scene loads.

<figure class="c43-fig">
<div class="c43-data-row"><div class="c43-box c43-data"><strong>Assets file</strong><span class="c43-sub"><code>Texture</code>, <code>Animation</code>, font and sound lines</span></div><div class="c43-arrow">loaded once</div><div class="c43-box c43-blue"><strong>GameEngine</strong><span class="c43-sub">shared by every scene</span></div></div>
<div class="c43-data-row"><div class="c43-box c43-data"><strong>Level files</strong><span class="c43-sub">tiles, NPCs, patrol points, per-entity parameters</span></div><div class="c43-arrow c43-both">load, edit, save</div><div class="c43-box c43-scene"><strong>Level editor</strong><span class="c43-sub">grid snapping, ImGui inspector</span></div></div>
<div class="c43-data-row"><div class="c43-box c43-data"><strong>Level files</strong><span class="c43-sub">and the player configuration</span></div><div class="c43-arrow">load at level start</div><div class="c43-box c43-scene"><strong>Gameplay</strong><span class="c43-sub">entities and systems from the assignments</span></div></div>
<div class="c43-data-row"><div class="c43-box c43-data"><strong>Save file</strong><span class="c43-sub">progress and unlocked levels</span></div><div class="c43-arrow c43-both">load, save</div><div class="c43-box c43-scene"><strong>Overworld map</strong><span class="c43-sub">level select and progression</span></div></div>
<figcaption markdown="span" style="margin-top: 0.6rem; color: #666; font-size: 0.85rem; line-height: 1.5;">The external files the specification requires and the scenes that read or write them.</figcaption>
</figure>

## What I would keep

The games are small, but the habits carry over: keep data and logic apart, change structure at frame boundaries, make debugging tools part of the engine, and measure before optimizing.

## Sources

- [COMP 4300 Fall 2025 lecture playlist](https://www.youtube.com/playlist?list=PL_xRyXins848Mfiv4hIgiMZgLpzFT7ohv), David Churchill, Memorial University.
- [Project specification and template](https://github.com/davechurchill/COMP4300_Project), including the marking scheme and feature checklist.
- Screenshots are frames from the lecture videos linked in their captions; the diagrams were drawn for this post.
