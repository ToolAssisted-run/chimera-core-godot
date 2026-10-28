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
  Mesa's llvmpipe through OSMesa, compiled into the core (Mesa 24.0.9, as
  the RSDKv4, touchHLE and PCSX2 cores have it, with LLVM 18.1.8). The GPU
  bridge is for later and never the only renderer.
- **A recompiler?** GDScript is interpreted. PCRE2's JIT (for RegEx) is built
  out. llvmpipe compiles shaders with LLVM, pinned to one CPU (below).
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
  memory and goes into states. Directory listings are sorted.
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
- Speed: see "The software OpenGL" below.
- Through Chimera: `chimera-run` from a movie file and from a project, and
  Chimera's window run headless, all draw the reference's last picture.

## Save data (2026-10-20)

- Export Save Data is every file under `user://`, by its path there
  (`saves/slot.dat`); Chimera zips several. A project's "Save data" slot is
  read during Init, after Godot is set up and before its first scene: a file
  goes to `user://<its name>`, a `.zip` is unpacked with its folders (by
  Godot's own ZIPReader, reading from the core's disk). A zip entry that is
  not a plain relative name (`../escape.txt`) refuses the load and says
  which. A file of the export that was in a folder goes back in it only
  from the zip: a single file's name has no folder.
- Godot keeps two things of its own in `user://` on a desktop: its log
  (`user://logs`) and the OpenGL shader cache (`user://shader_cache`, seen
  in the first export). Both would come out as save data and fill every
  state. The log goes to `/dev/null` (`--log-file`; a file in the core's disk
  that keeps nothing; its lines still reach the console), and the shader
  cache is switched off in the project's settings after they are read and
  before the renderer starts. An `override.cfg` did this at first; a game
  may forbid overrides, or bring its own, so it is gone.

## The software OpenGL: softpipe, then llvmpipe (2026-10-20)

The first build drew with softpipe, as the RSDK cores do. A profile
(callgrind, Dodge the Creeps in play) put 70% of all the time in softpipe's
shader interpreter (`tgsi_exec.c`): every pixel of Godot's canvas shader
interpreted. Measured on this machine: Dodge the Creeps (480x720) 10 frames
a second, the 2D platformer demo (1600x960) 0.85, the 3D platformer demo
(1152x648, made for Forward+) 0.26.

llvmpipe compiles each shader to machine code with LLVM. Two things make
that safe for a machine that must be the same everywhere:

- **One CPU on every computer** (mesa-patches/0001, `CHIMERA_PINNED_CPU`):
  Mesa reports an x86-64 with SSE2 and nothing newer, and one processor;
  LLVM targets the baseline `x86-64`. The code made for a shader is then the
  same bytes whatever CPU runs it, and with one processor llvmpipe
  rasterises on the calling thread and starts none (the sandbox has one).
  The CPUID instruction cannot be trapped in a sandbox, so this is the only
  way to keep the host's features out.
- **No functions found by name.** LLVM's JIT first opens the program with
  `dlopen(NULL)`, which the sandbox's static C library refuses ("Dynamic
  loading not supported"), and Mesa then crashed on the missing JIT.
  `waterbox/jit-symbols.c` answers dlopen, dlsym and dlclose in both builds
  from one short table, so the native build cannot find by name what the
  sandbox cannot. Over the demos, the generated code asked for no name.

LLVM is built by `waterbox/build-llvm.sh` (X86 only, no threads, static, RTTI
on to match Mesa; without CET in both flavours so the code LLVM generates is
the same, as chimera-core-rpcs3 learnt); only its libraries are built for the
guest. The core grew from 125 MB to 187 MB.

Measured with llvmpipe, native = sandbox in every picture as before (the
pictures differ from softpipe's in rounding at the edges; the game is the
same): Dodge the Creeps 600 frames in 9.3 s in the sandbox (65 a second,
from 12), the 2D platformer 8 a second (from 0.85), the 3D platformer about
1 (from 0.26).

The GPU bridge would be faster still, but it gives the renderer a new
OpenGL context at every savestate load, and Godot's renderer cannot rebuild
its textures, buffers and shaders from the machine's memory: that is a port
of its storage classes, and the owner's call.

## A core per Godot version (user-decided, 2026-10-20)

Since 4.3, an exported game's scripts are compiled to a token format that
only the same Godot version loads (`TOKENIZER_VERSION` must match exactly),
and a pack made by a newer Godot is refused by an older one. The owner chose
cores for 4.4, 4.5 and 4.6 beside this one: chimera-core-godot44, -godot45
and -godot46, each its own package and roster entry (one package holds one
engine), each this repository's code with its own Godot pinned (4.4.1,
4.5.2, 4.6.3) and test games exported by that version's editor.

- One module for all of them: `chimera_version.h` gives the minor version,
  `chimera_compat.h` the differences - the display server's headers moved to
  `servers/display/` in 4.6 and its enums to `DisplayServerEnums` in 4.7;
  `GDSOFTCLASS`, `get_cwd`, the file access and multibyte calls are newer
  than 4.4 or 4.5.
- Every core reads the version from the pack's header (at the start of a
  .pck, at the end of an executable it is embedded in) and refuses a game of
  another version, naming the core for it. The 4.4 core also takes games of
  4.0 to 4.3, for which there is no core. Tested by the `version` tests:
  packs with only a header, of another version, embedded, of Godot 3, of 4.2
  and of the core's own.
- Godot 4.4 writes the objects of a module outside its tree beside the
  module's sources: `*.o` is ignored.

## Drawing Size (2026-10-28, chimera issue 380)

Godot draws every pixel on the CPU here (llvmpipe, on one processor, with
the baseline x86-64's instructions), so the time a frame takes follows the
number of pixels drawn: a game drawing at 1920x1080 ran at about one frame
a second. The owner chose two ways to make that faster; this is the first.

The Drawing Size setting (100%, 75%, 50% or 25%) gives the game a window of
that share of the size it asks for, and the game draws at that size; the
picture is shown at the same size in Chimera, softer. A game that stretches
its picture to its window (Godot's `canvas_items` or `disabled` stretch
modes) draws fewer pixels; one that draws at a fixed size of its own
(`viewport`) is not made faster. The game is told the smaller size, so it is
part of the machine: a movie needs the same value. A size the game was
given, asked for again, is kept, so the share is never taken twice.

Measured on Dodge the Creeps (480x720), 600 frames in the sandbox on this
computer: 9.2 s at 100%, 4.8 s at 50% (start included). At 50% it draws
240x360, the same scene, and plays exactly the same steps. The test script
checks that, and that native and the sandbox agree at 50%; it was seen to
fail with the setting ignored.

## Drawing Code: AVX2 where the computer has it (user-decided, 2026-10-28, chimera issue 380)

The second way. llvmpipe was told of the baseline x86-64 on every computer
(mesa-patches/0001), so every picture was the same everywhere, at the cost of
128-bit vectors and none of the newer instructions. The owner chose to let it
use AVX2, and to make that the default.

- The Drawing Code setting: AVX2 (the default) or Baseline. With AVX2, Mesa
  (mesa-patches/0002) asks the computer whether it has all of x86-64-v3
  (AVX2, FMA, BMI1 and 2, LZCNT, MOVBE, F16C, and an OS that saves the wider
  registers); if so, llvmpipe is told of that CPU - 256-bit vectors, still
  one processor - and LLVM makes x86-64-v3's code, the same on every such
  computer. Otherwise, and with Baseline, the 0001 baseline as before.
- The game plays the same either way: only the pictures may differ by a
  shade between a computer with AVX2 and one without. A movie's steps,
  sound and everything the game decides are unchanged; a game that reads
  its own pictures back is the exception.
- The core gives Mesa the setting through `chimera_llvmpipe_avx2`, a weak
  symbol for Mesa, so Mesa's own libOSMesa keeps the baseline.
- miniBox saves the whole vector state (xsave, every feature the OS turns
  on) when a fault stops the guest, so 256-bit registers survive it.

Measured on Dodge the Creeps, 600 frames in the sandbox on this computer,
start included: 9.6 s with Baseline, 8.0 s with AVX2, 4.6 s with AVX2 at
Drawing Size 50%; the same steps in all three. The test script checks the
256-bit code in the sandbox and natively, native = sandbox with it (every
picture, the sound, the steps, savestates), and that Baseline still makes
128-bit code and plays the same steps. The 256-bit checks failed while the
setting was not read when not given (a project made before has none).

## Sharp edges

- `pkill -f` with the name of a running script kills the shell that typed it
  when the command line holds the same words: stop a build by its PID.
- SCons objects land beside the sources: never build in `extern/godot`.
- A static library is only pulled in where something calls it: the Chimera
  module is linked whole (`--whole-archive`), or the guest has no `Init`.

## Next

- A real game from boot to play, and a game with a `.pck` inside its
  executable.
- Windows: the frontend test's scenario on the Windows build.
- Speed: the 3D renderer on llvmpipe; the GPU bridge as an option.
- What a TAS needs from a Godot game's memory: its scripts' variables as a
  property table.
