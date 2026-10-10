# Flee the Facility 2D

A 2-player, top-down pygame version of Roblox's *Flee the Facility*. The two players hack computers and then escape through an exit. A simple AI monster (Verity) wanders the map and chases any player it sees.

The whole map fits on one screen. There is no camera movement or zooming.

## How to run

```
pip install pygame
python main.py
```

`verity happy.png` and `verity angry.png` must be in the same folder as `main.py`.

## Working on this project

Edit the code directly in this folder on the `main` branch. Don't use separate git worktrees or feature branches. This applies to AI assistants too.

## Controls

| | Player 1 (blue) | Player 2 (orange) |
|---|---|---|
| Move forward / back | W / S | Up / Down |
| Turn left / right | A / D | Left / Right |
| Interact (hack, rescue) | E | P |

Only forward/back moves the player. Left/right only rotate them.

## Rules

- **Computers.** There are 10 computers, drawn as red boxes. Stand next to one (within `INTERACT_RADIUS`) and hold your interact key for 5 seconds (`HACK_TIME` frames) to solve it. Solved computers turn green.
- **The monster.** Verity wanders randomly. When it sees a player it switches to the angry image and chases at 1.5x player speed. When it can no longer see anyone it goes back to the happy image and wanders again.
- **Monster vision** uses vectors only:
  1. `to_player = player.position - enemy.position`
  2. The player must be within `DETECTION_RANGE` (`to_player.length()`).
  3. `to_player.normalize().dot(enemy.forward_vector()) > FOV_THRESHOLD`. This means the player is in front of the monster.
  4. The line from the monster to the player must not cross any wall (`line_hits_wall`).
- **Caught.** If the monster gets within `CATCH_RADIUS` of a player, that player is moved into the nearest empty tube (cyan circle) and is stuck there.
- **Rescue.** The other player stands at the tube and holds their interact key for 5 seconds (`RESCUE_TIME`). There is no time limit for being stuck.
- **Darkness.** The forest (every outdoor area except the path) is dark. Each player has a cone-shaped flashlight (`FLASHLIGHT_RANGE`, `FLASHLIGHT_THRESHOLD`, using the same dot-product test as monster vision). Inside the forest, the monster is only drawn if it is inside a flashlight cone.
- **Exits.** The two exits (bottom of the vertical path, and the right edge next to the horizontal path) are grey and locked. They turn yellow and open once all 10 computers are solved. A player who walks through one has escaped.
- **Win / lose.** The players win when every player who isn't trapped has escaped. The monster wins if both players are in tubes at the same time.

## Map

Built from the hand-drawn map. Walls are black lines. Indoor floors are light grey, the path is beige and the forest is green.

| Area | Computers | Tube |
|---|---|---|
| Small Shed (top left) | 0 | 1 |
| White House | 1 | 1 |
| Maze (empty room for now; maze walls can be added) | 1 | 1 |
| Garage | 1 | 1 |
| Log | 1 | 1 |
| Brown House | 3 | 1 |
| Barn | 2 | 1 |
| Shed (bottom left) | 1 | 1 |
| Forest (left / right) | 0 | 1 each |

All map positions in `main.py` use the original drawing's coordinates (1080 x 1160) and are shrunk by `MAP_SCALE` using `scale()`, `scaled_point()` and `scaled_rect()`. To move a computer, tube or doorway, change its numbers in the map data lists. To resize the whole game, change `MAP_SCALE`.

## Code tour (`main.py`)

| Part | What it is |
|---|---|
| Settings | Speeds, ranges, timers and colours. Tune the game here. |
| `ROOMS`, `PATH_RECTS` | Rects for each building and the path. These are used for floors and for `is_dark`. |
| `DOORWAYS` | A gap in each room's wall: `(room, side, gap start, gap end)`. |
| `COMPUTER_SPOTS`, `TUBE_SPOTS`, `EXIT_LINES` | Where things are. |
| `Wall` | A line segment, with `start` and `end`. `build_walls()` makes all of them. |
| `Computer` | `progress` (frames held) and `solved`. Draws red or green plus a progress bar. |
| `Tube` | `trapped_player` and `rescue_progress`. |
| `Exit` | `is_open`. |
| `Player` | `position`, `angle`, `state` (`"FREE"`, `"TRAPPED"`, `"ESCAPED"`) and its own keys. |
| `Enemy` | `position`, `angle`, `state` (`"PATROL"`, `"CHASE"`), `image` (swap between `happy_image` and `angry_image`). |

Everything is already drawn. The game logic is empty: methods like `Player.move` and `Enemy.can_see` just `pass` or return a placeholder, and the game loop only draws.

Angles work like in class: `pygame.Vector2(1, 0).rotate(angle)` is the facing direction, and 0 degrees points right.

## Tasks

Game-loop logic goes between `keys = pygame.key.get_pressed()` and `draw_floor(screen)`.

1. **Player movement.** Write `Player.forward_vector`, `turn` and `move`. In the loop, call them for each `"FREE"` player using that player's own keys.
2. **Wall collision** for players and the monster. A simple approach: try the move, and if the new position is closer than `radius` to any wall segment, undo it. To get the closest point on a segment, project onto it: `t = (p - a).dot(b - a) / (b - a).length_squared()`, clamp `t` between 0 and 1, and the closest point is `a + (b - a) * t`.
3. **Hacking.** In the loop, a `"FREE"` player within `INTERACT_RADIUS` of an unsolved computer who is holding their `interact_key` adds 1 to `computer.progress`. When it reaches `HACK_TIME`, set `solved = True`.
4. **Monster wandering.** Write `Enemy.forward_vector`, `wander` and `move`. Every so often (use `wander_timer`), pick a random angle and move forward at `ENEMY_SPEED`.
5. **Monster vision and chasing.** Write `Enemy.can_see` (see "Monster vision" above), `chase` and `line_hits_wall`. In the loop: if the monster can see a `"FREE"` player, set state `"CHASE"`, switch to `angry_image` and chase at `ENEMY_CHASE_SPEED`. Otherwise set state `"PATROL"`, switch to `happy_image` and wander. For segment intersection, segments AB and CD cross if C and D are on opposite sides of AB *and* A and B are on opposite sides of CD. Use the 2D cross product (`Vector2.cross`) to tell which side a point is on.
6. **Catching.** Write `nearest_tube`, which returns the closest empty tube (use `min()` with `distance_to`, like the missiles in class). In the loop, if the monster is within `CATCH_RADIUS` of a `"FREE"` player, move that player to the nearest tube, set their state to `"TRAPPED"`, and set `tube.trapped_player`.
7. **Rescuing.** In the loop, a `"FREE"` player within `INTERACT_RADIUS` of a tube with a trapped player, holding their `interact_key`, fills `rescue_progress`. When it reaches `RESCUE_TIME`, free the trapped player and reset the tube.
8. **Darkness and flashlight.** Write `is_dark` (true when a point is outside every rect in `ROOMS` and `PATH_RECTS`; use `rect.collidepoint`) and `in_flashlight` (the same test as monster vision, using `FLASHLIGHT_RANGE` and `FLASHLIGHT_THRESHOLD`). When drawing, only draw the monster in the dark if it is inside a player's flashlight. Then draw the darkness: one way is to draw a black surface over the screen, cut out the room/path rects and each player's cone polygon (using `set_colorkey`), then blit it.
9. **Exits and win/lose.** When every computer is solved, set `is_open` on the exits. A player touching an open exit becomes `"ESCAPED"`. Set `game_state` when the monster wins (both players `"TRAPPED"`) or the players win, and show a message on screen.
