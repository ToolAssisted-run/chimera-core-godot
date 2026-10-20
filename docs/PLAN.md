# The Godot core: plan and design log

Chimera issue #322 asked for Godot games. The owner decided on 2026-10-20 to
build a core, starting with Godot 4.7.2, the latest stable release
(user-decided, 2026-10-20). Unity (#320) and Unreal (#321) were declined the
same day: their engines are closed binaries, and the sandbox runs only code
built for it.

## The answers before any code (docs/new-core.md, section 1, in Chimera)

- **A new core?** Yes. No core here runs a game engine of this kind.
- **Licence.** Godot is MIT (`extern/godot/LICENSE.txt`); its third-party
  libraries are permissive and listed with their texts in `COPYRIGHT.txt`.
  Binary and modified redistribution are allowed; the package carries the
  texts.
- **One thread on one clock?** Yes. Godot builds without threads
  (`threads=no`, made for the web): its worker pool runs tasks on the main
  thread, and loading is synchronous. Its Movie Maker mode already runs a
  game on a fixed clock: `--fixed-fps` makes every frame exactly 1/fps of a
  second of game time and skips every wait for the real clock, and the sound
  is mixed on request by the "Dummy" audio driver without its thread. The
  core does exactly that.
- **The picture.** Godot's Compatibility renderer is OpenGL 3.3. It draws on
  Mesa's softpipe through OSMesa, compiled into the core (the same Mesa
  24.0.9 as the RSDKv4, touchHLE and PCSX2 cores). The GPU bridge is for
  later and never the only renderer.
- **A recompiler?** GDScript is interpreted. PCRE2's JIT (for RegEx) is built
  out.
- **What the user brings.** The game's `.pck`, or its executable with the
  pack inside. Nothing else.
- **What CI can run.** Godot's "Dodge the Creeps" demo (MIT), exported to a
  `.pck`, and this repository's probe game (`tests/probe`); both are in
  `tests/`.
- **Size.** The guest is 125 MB before packing (36 MB in the package). A
  state of Dodge the Creeps is 31 MB. The layout gives Godot 1 GiB of heap
  and 4 GiB of mmap space.

## How it is built (2026-10-20)

- **Godot as libraries.** `build-godot.sh` builds the `linuxbsd` export
  template (`target=template_release`) with `library_type=static_library`,
  in a copy of the submodule per flavour (SCons writes beside the sources).
  Off: X11, Wayland, sound servers, D-Bus, speech, system fonts, udev, SDL,
  AccessKit, Vulkan, OpenXR, cameras, PCRE2's JIT.
- **The Chimera module** (`module/chimera`, a Godot module given to SCons as
  `custom_modules`) holds everything of the core: the OS, the disk, the
  window and the calls Chimera makes. The core's entry points live in it, so
  every file that touches Godot is compiled with Godot's own flags.
- **The core finds the pack the way an exported game does.** Export
  templates refuse `--main-pack` (path overrides are built out). The core
  tells Godot its executable is `/game/<pack name without .pck>`, and Godot
  finds `<name>.pck` beside it; for an executable with the pack inside, the
  executable path is that file, and Godot finds the pack at its end.
- **Patches.** 0001: two files that use `std::unique_ptr` include `<memory>`
  themselves (a build without threads did not bring it in). The other
  threads=no hole, thread naming, is closed with `-DPTHREAD_NO_RENAME`.

## The machine (2026-10-20)

- **The OS** (`chimera_os.cpp`) answers every question from the core:
  `get_ticks_usec` is frames x 1,000,000 / fps and stands still within a
  frame; the date is the Starting Clock setting plus the frames, in UTC;
  one processor; no environment variables, processes or dynamic libraries;
  `get_entropy` is a splitmix64 stream in the OS object (so in the state).
  Godot was checked for code that waits for its clock to move inside a frame
  (none outside the editor).
- **The disk** (`chimera_fs.cpp`) is one tree in the core's memory for
  `res://`, `user://` and plain paths: `/game` holds the project's files,
  read through the descriptors Chimera mounted (never copied in); `/user` is
  `user://`, empty at power-on; anything a game writes anywhere stays in
  memory and goes into states. Directory listings are sorted. File logging
  is turned off by an `override.cfg` the core puts beside the executable.
- **The window** (`chimera_display.cpp`) is the size the game's project
  settings ask for, up to 1920x1080 (the package's capacity); the one screen
  is that size. After each frame the picture is read back with
  `glReadPixels`, rows flipped, made opaque. Turbo skips the reading only.
  The window tells the game once that it has focus and the mouse is in it,
  as a desktop does, or GUI hovering never starts.
- **Input** (`chimera_core.cpp`): before each frame, the controls that
  changed become Godot's input events: keys (keycode, physical keycode,
  label, the character a US layout types), the mouse (motion in pixels of
  the window, buttons with the pressed mask, a double click when the same
  button is pressed again within 400 ms and 5 pixels, the wheel a notch a
  frame) and one game controller connected at start in Godot's own button
  numbering (so no mapping is looked up). Shift, Ctrl and Alt go down before
  the other keys of a frame and up after them: with one order for all, Shift
  and A in one frame typed "a" (measured with the probe).
- **Sound**: 44100 Hz stereo, mixed for exactly one frame's share of the
  second (735 samples at 60, the remainder carried at rates that do not
  divide it), converted from Godot's 32-bit samples.
- **Lag**: Godot reads its input every frame; no frame is a lag frame.

## The native reference computes as the sandbox does (2026-10-20)

The sandbox's C library is musl and the native build's is glibc, and their
`sin`, `pow`, `expf` and the rest differ in the last bit. Instead of a
replacement maths library (the RSDK cores' detmath.c, an approximation), the
native build links the very objects the sandbox's libm was made of
(`make-native-libm.sh`: miniBox's guest musl math objects, plain x86-64
code, renamed `__wrap_<name>` and reached through the linker's `--wrap`).
The two builds then run the same machine code for every maths function, in
Godot and in Mesa alike.

## Measured (2026-10-20)

- Dodge the Creeps, 600 frames (the game started with Enter, the player
  moved with the arrows, creeps spawned at random): native = sandbox in
  every picture, the sound and the steps; the same with a save and load
  before every frame, and saved at frame 300 into a new host.
- The probe: the clock (1/60 s a frame, 2000-01-01 at power-on), Shift+A
  typing "A", the mouse's pixel, a double click, the wheel, the controller's
  button and axes.
- Speed: about 10 frames a second for Dodge the Creeps (480x720), in both
  builds, on this machine (a 20-thread x86-64).
- Through Chimera: `chimera-run` from a movie file and from a project, and
  Chimera's window under Xvfb, all draw the reference's last picture.

## Sharp edges

- `pkill -f` with the name of a running script kills the shell that typed it
  when the command line holds the same words: stop a build by its PID.
- SCons objects land beside the sources: never build in `extern/godot`.
- A static library is only pulled in where something calls it: the Chimera
  module is linked whole (`--whole-archive`), or the guest has no `Init`.

## Next

- Save data: Export Save Data from `user://`, and a project's save data put
  in place before the game starts.
- A real game from boot to play, and a game with a `.pck` inside its
  executable.
- Windows: the frontend test's scenario on the Windows build.
- Speed: where the frame goes on softpipe; the GPU bridge as an option.
- What a TAS needs from a Godot game's memory: its scripts' variables as a
  property table.
