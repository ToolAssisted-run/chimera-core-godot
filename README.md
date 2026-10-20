# chimera-core-godot

A core for [Chimera](https://github.com/ToolAssisted-run/chimera), the
frontend for tool-assisted speedruns, that plays games made with the
**[Godot Engine](https://godotengine.org)**. It is Godot's own engine for
exported games (4.7.2), built for Chimera's sandbox, and produces one file,
`godot.chimeraCore`, that goes in Chimera's `Cores` folder.

Each Godot 4 version has its core, because a game's compiled scripts only
load in the version that exported them: this one runs Godot 4.7 games, and
[4.6](https://github.com/ToolAssisted-run/chimera-core-godot46),
[4.5](https://github.com/ToolAssisted-run/chimera-core-godot45) and
[4.4](https://github.com/ToolAssisted-run/chimera-core-godot44) (which also
takes games of 4.0 to 4.3) have theirs, built from this repository's code. A
core refuses a game of another version and says which core to use.

## Status

Published: each push to `main` that passes the tests publishes a `dev`
release, and a dated nightly is made once a day when `main` moved. Chimera's
Core Manager lists it.

- Godot 4 games written in GDScript, drawn with Godot's Compatibility
  renderer (OpenGL 3.3) on a software OpenGL inside the core (Mesa's
  llvmpipe, made to generate the same code on every computer). A game made
  for the Forward+ or Mobile renderer is drawn by the Compatibility renderer
  too, which lacks some of their effects.
- Tried: Godot's "Dodge the Creeps" demo and this repository's probe game.
  The sandboxed core and a plain native build agree on every picture, the
  sound and everything the game printed; savestates land where the straight
  run does. No commercial game has been tried yet.
- Not there: games written in C# (Godot's .NET version) and games that need
  native extensions (GDExtension libraries), which the sandbox cannot load;
  Godot 3 games; the GPU. On a fast computer a small 2D game runs at about
  60 frames a second, a 1600x960 one at about 8, and a 3D one at about 1.

## What you bring

The game's `.pck` file: an exported Godot game is an executable and a `.pck`
beside it (`Game.exe` and `Game.pck`). A game exported as one file has the
`.pck` inside its executable: give the executable instead. Files the game
loads from beside its executable (a second `.pck`, data) go in the project's
"Other files" slot.

## Buttons and settings

- **The keyboard**: every key of a US keyboard, under its name. A key types
  the character it would on that keyboard, with Shift.
- **The mouse**: Mouse X and Mouse Y say where it is in the window (0 to
  65535 from edge to edge), with its buttons and its wheel (a notch for each
  frame Mouse Wheel Up or Down is held).
- **A game controller**: player 1's, with its sticks and triggers, in
  Godot's own layout.
- **Settings**: Frame Rate (how many frames make a second of the game's time;
  60 by default), Starting Clock (the date the game reads; movies need the
  same value) and Godot's Log (Godot's `--verbose`).

What a game writes to `user://` (its saves and settings) lives in the core's
memory, starts empty, and is carried by savestates. Emulator > Export Save
Data writes it out (several files as a `.zip`), and a project's "Save data"
slot puts such files, or that `.zip`, back in `user://` before the game
starts.

## Building and testing

[docs/BUILDING.md](docs/BUILDING.md) has every step. [docs/PLAN.md](docs/PLAN.md)
is the design log: what was decided, why, and what was measured.

## Licence

This repository's own code is MIT, and so is Godot. The libraries inside
Godot are under their own permissive licences, listed in Godot's
`COPYRIGHT.txt`; the software OpenGL is Mesa (MIT) with LLVM (Apache 2.0
with the LLVM exception). The package carries every licence text. See
[LICENSE](LICENSE).
