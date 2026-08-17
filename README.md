# The Stickrooms — Python + C++ + Ursina + CMake

## inpired by Escape The Backrooms by Fancy games

## Architecture

- **C++** (`src/maze_module.cpp`, built with **CMake**): generates the maze
  layout using a recursive-backtracker algorithm. This is compiled into a
  Python extension module via **pybind11**, so Python can `import` it
  directly. This is the piece that benefits from being compiled — maze
  generation is CPU-bound work.
- **Python + Ursina** (`main.py`): everything else — rendering, the player
  controller, enemy AI, UI, stickman models, and gameplay state. Ursina is
  built on Panda3D. Walls/floor/ceiling/characters use `lit_with_shadows_shader`
  with a shadow-casting flashlight, so lighting is dynamic instead of flat
  unlit color.
- **network.py / server.py**: plain-stdlib UDP multiplayer. `server.py` is a
  small relay server; `network.py` is the client used by `main.py`. See the
  Multiplayer section below.

This is the standard way to combine C++ and Python in a game: C++ isn't
"the game engine" here — Ursina/Panda3D still handles rendering — C++ is a
compiled extension module you call into for specific hot logic.

## Prerequisites

- **Python 3.10+**
- **CMake 3.15+** — [cmake.org/download](https://cmake.org/download/)
- **A C++ compiler**:
  - Windows: Visual Studio 2019/2022 with the "Desktop development with C++" workload (provides MSVC), or MinGW-w64
  - macOS: Xcode Command Line Tools (`xcode-select --install`)
  - Linux: `g++` or `clang++` (e.g. `sudo apt install build-essential`)
- Internet access the first time you build — CMake automatically downloads
  pybind11 for you via `FetchContent`, so you don't need to install it
  separately.

## Project structure

```
stickrooms_cpp/
├── CMakeLists.txt        # Build config — fetches pybind11, builds maze_module
├── src/
│   └── maze_module.cpp   # C++ maze generator (compiled to a Python module)
├── main.py                # The game itself (Python + Ursina)
├── network.py              # Multiplayer client (used by main.py)
├── server.py                # Multiplayer relay server (run standalone)
└── requirements.txt
```

## Build steps (VSCode terminal)

1. Open the `stickrooms_cpp` folder in VSCode.

2. Set up your Python environment:
   ```bash
   python -m venv venv
   ```
   Windows: `venv\Scripts\activate`  •  macOS/Linux: `source venv/bin/activate`
   ```bash
   pip install -r requirements.txt
   ```
   In VSCode: `Ctrl+Shift+P` → "Python: Select Interpreter" → pick the `venv` one.

3. Configure and build the C++ module with CMake:
   ```bash
   mkdir build
   cd build
   cmake ..
   cmake --build . --config Release
   ```
   The first `cmake ..` will download pybind11 automatically — this needs
   internet access and may take a minute.

4. Copy the compiled module next to `main.py`:
   - **Windows** (Visual Studio generator): the file is at
     `build\Release\maze_module.pyd` — copy it to the project root
     (next to `main.py`).
   - **Windows** (MinGW) / **Linux** / **macOS**: the file is at
     `build/maze_module*.so` (or `.pyd` on Windows/MinGW) — copy it to the
     project root.

   Alternative to copying: open `main.py` and uncomment the
   `sys.path.insert(...)` line near the top, adjusting the path to wherever
   your build put the compiled file.

5. Run the game:
   ```bash
   cd ..
   python main.py
   ```

## Rebuilding after editing the C++ code

Whenever you change `src/maze_module.cpp`:
```bash
cd build
cmake --build . --config Release
```
Then re-copy the newly built `maze_module.pyd`/`.so` next to `main.py`
(same step 4 above).

## Controls

| Key | Action |
|---|---|
| WASD | Move |
| Mouse | Look |
| Shift | Sprint (drains stamina) |
| F | Toggle flashlight |
| B | Toggle seeing your own stickman body |
| Esc | Pause / release mouse |
| R | Retry after being caught |

## Multiplayer

`network.py`/`server.py` are a small plain-stdlib UDP relay — no extra pip
installs needed.

**1) Start the server once** (on whichever machine will host the session):
```bash
python server.py
```
It listens on port 25565 by default (`python server.py 12345` to use a
different port) and just relays player positions — it doesn't run Ursina
or need the C++ module.

**2) Every player runs the game and points it at the server:**
```bash
python main.py <server_ip>
```
- Testing multiple players on one PC: open a few terminals and run
  `python main.py 127.0.0.1` in each (each will connect as a separate
  player to the same local server).
- LAN play: run `python main.py 192.168.x.x` using the host machine's LAN
  IP address instead.
- Running `python main.py` with no argument defaults to `127.0.0.1`. If no
  server is reachable there, the game prints a message and just continues
  in single-player mode instead of crashing.

**How it works:** whichever client connects first becomes "host" and is
authoritative for the enemy AI — it runs the real wander/chase logic and
reports the result to the server, which relays it to everyone else.
Non-host clients don't run their own enemy AI; they just position their
local enemy entity to match what the host reports, so everyone sees the
same enemy behave identically. Every player is independently catchable —
each client checks its own distance to the (possibly networked) enemy.

Current limitations worth knowing:
- The maze uses a fixed seed when multiplayer is on (`1234567`) so
  everyone's walls line up — it doesn't fetch the seed from the server.
- If the host disconnects, host duty passes to whichever player connected
  next, but the enemy will "teleport" to wherever their locally-running AI
  currently has it, since a fresh `Enemy` isn't handed the outgoing host's
  last exact state.
- There's no player-vs-player collision — players can walk through each other.

## Realistic lighting

Walls, floor, ceiling, and all stickman characters use Ursina's built-in
`lit_with_shadows_shader` instead of flat unlit color, so they actually
respond to light direction and can cast/receive shadows. Sources of light
in the scene:
- A dim `AmbientLight` — soft fill so unlit areas aren't pure black.
- The flickering ceiling `PointLight` fixtures from before (no shadows —
  there are many of them, and shadow-casting on all of them would be
  expensive).
- Your flashlight — a `SpotLight` parented to the camera, the one
  shadow-casting light, giving real dynamic shadows as you move and look
  around. Toggle with `F`.

If shadows look too soft/hard-edged or too dim/bright for your taste, the
easiest knobs to turn are the `AmbientLight` color/alpha near the top of
`main.py` (raise the alpha for a brighter base level) and the flashlight's
`color`/position in the `Player` class.

## Tuning the maze

In `main.py`:
```python
maze_result = maze_module.generate_maze(cells_x=7, cells_y=7, seed=maze_seed)
```
`cells_x` / `cells_y` control maze size (bigger = larger level). `seed` is
randomized each run — pass a fixed integer instead if you want the same
layout every time (useful while testing).

## Where to take the C++ side next

Maze generation was the natural first candidate, but the same
Python↔C++ pattern works for anything CPU-bound:
- Enemy pathfinding (A* across the maze grid) instead of the current
  simple wander/chase logic in `main.py`
- Procedural texture or noise generation
- Physics/collision-heavy systems if the project grows

Add a new function to `src/maze_module.cpp` (or a new `.cpp` file added to
`CMakeLists.txt`), bind it with pybind11 the same way `generate_maze` is
bound, rebuild, and call it from `main.py`.
