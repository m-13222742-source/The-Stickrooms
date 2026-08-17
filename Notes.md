# Note on platform support

This release's precompiled C++ module (`maze_module.cp313-win_amd64.pyd`)
was built for **Windows, 64-bit, Python 3.13**. It will not import on
other setups (Mac, Linux, or a different Python version/bitness on
Windows) — you'll get a `ModuleNotFoundError` when running `main.py`.

This isn't a permanent limitation of the game — the C++ source
(`src/maze_module.cpp`) and `CMakeLists.txt` are fully cross-platform.
If you're not on Windows 64-bit / Python 3.13, build it yourself instead
of using the included `.pyd`:

```bash
python -m venv venv
# Windows: venv\Scripts\activate   |   Mac/Linux: source venv/bin/activate
pip install -r requirements.txt

mkdir build && cd build
cmake ..
cmake --build . --config Release
```

Then copy the resulting `maze_module.pyd` (Windows) or `maze_module.so`
(Mac/Linux) from `build/` (or `build/Release/`) into the project root,
next to `main.py`, and run:

```bash
python main.py
```

See `README.md` for full build instructions and requirements (CMake 3.15+,
a C++ compiler, internet access on first build to fetch pybind11).
