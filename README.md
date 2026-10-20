# chimera-core-godot

A core for [Chimera](https://github.com/ToolAssisted-run/chimera), the
frontend for tool-assisted speedruns, that plays games made with the
**[Godot Engine](https://godotengine.org)**. It is Godot's own engine for
exported games (4.7.2), built for Chimera's sandbox, and produces one file,
`godot.chimeraCore`, that goes in Chimera's `Cores` folder.

## Status

New, and not published yet.

- Godot 4 games written in GDScript, drawn with Godot's Compatibility
  renderer (OpenGL 3.3) on a software OpenGL inside the core. A game made
  for the Forward+ or Mobile renderer is drawn by the Compatibility renderer
  too, which lacks some of their effects.
- Tried: Godot's "Dodge the Creeps" demo and this repository's probe game.
  The sandboxed core and a plain native build agree on every picture, the
  sound and everything the game printed; savestates land where the straight
  run does. No commercial game has been tried yet.
- Not there: games written in C# (Godot's .NET version) and games that need
  native extensions (GDExtension libraries), which the sandbox cannot load;
  Godot 3 games; the GPU. The software OpenGL is slow: about 10 frames a
  second for a small 2D game on a fast computer.

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
`COPYRIGHT.txt`; the software OpenGL is Mesa (MIT). The package carries every
licence text. See [LICENSE](LICENSE).
