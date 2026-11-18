# NHL 20 PS4 port: handoff

State on 2026-11-18. The next thing to do is at the end, under **Next steps**.

## The project

- Runs the PS4 build of NHL 20 natively on Windows with the SacredFire/bbport loader: the game's
  own x86-64 code, Windows versions of the PS4 system libraries (`src/runtime_*.c`), graphics through
  a shadPS4-based Vulkan backend (`gpu/shadps4/video_core`).
- Started from the NHL 21 port (v0.1.1), which grew from the NHL 19 port and GNSB's UFC 3 port, so
  most fixes are shared with those repos (see **Shared code** below).
- Game: **CUSA15439** (North America) with update **1.20**, recognised by a fingerprint of its
  executable (`scripts/game_check.py`, profile `nhl20`). The README's warning box still says
  "update 1.10": that's a mistake, 1.20 is right.
- Repo: `C:\Users\kshla\source\repos\NHL-20-PS4-PORT`, GitHub `thegh0sts/NHL-20-PS4-PORT` (public).
  Commits use `17183580+thegh0sts@users.noreply.github.com`.
- **Never commit or publish game files or files made from them**: `nhl20\` (the dump), `out\`
  (`boot*.bin`, `eboot.elf`, …), `user\`. Only `out\run.txt` and launcher logs can be shared.

## Building, running, releasing

All in the MSYS2 **CLANG64** shell, from the repo folder:

```
bash scripts/build-msys2.sh                         # build (out/windows/bin)
bash scripts/run-msys2.sh nhl20                     # first run: prepares out/ from the dump
BB_SKIP_PREPARE=1 bash scripts/run-msys2.sh         # later runs; log in out/run.txt
bash scripts/package-msys2.sh                       # dist/NHL20Native-windows-x86_64.zip (+ .sha256)
```

- The version comes from `git describe --tags`. **Tag before building**, or the build is
  `vX-dirty` / `vX-N-gHASH` and `package-msys2.sh` refuses it.
- Release: `git tag vX` → build → package → `git push origin vX` →
  `gh release create vX dist/NHL20Native-windows-x86_64.zip dist/NHL20Native-windows-x86_64.zip.sha256 --title "NHL 20 port vX" --notes-file docs/release-notes/vX.md`.
  Claude sessions can't create tags or releases on GitHub: the owner runs these.
- The GitHub `windows-cross` workflow builds a different archive and its test step fails; releases
  are built locally.
- The game's own output (`printf`) goes to `out/run.txt`, not always to the terminal.

## Releases

| Version | Date | Contents |
|---|---|---|
| v0.1.0 | 2026-11-15 | First test release: boots, menus, Play Now matches; launcher `NHL20.exe` |
| v0.1.1 | 2026-11-17 | Fix for the SimThread crash (red zone, first version, see below) |

Release notes: `docs/release-notes/`. **v0.1.2 is not made yet** (see Next steps).

## Fixed: the SimThread crash (v0.1.1, improved since)

- Symptom: `Guest fault … read access to 00000071425xxxxx at guest offset 0x710df7 (thread
  SimThread)`, `rax=0x13`, same place every time.
- Cause: red-zone protection (`src/runtime_fp.c`, `runtime_redzone_patch_image`) moves the
  game's accesses below the stack pointer 64 KiB lower, so that Windows' exception handling (frequent
  on purpose, for GPU memory tracking) can't overwrite them. It rewrites each access with a jump to a
  stub; an access whose following instructions can't move was **kept**, on its own. In
  FUN_00710920 the write of `[rsp-0x20]` (0x710ceb) moved but its read (0x710de2) was kept, so the
  read got junk → a wild array index. The log line "Red zone: the access at image offset 0x710de2 is
  kept" showed it. 34 write/read pairs in that function were split this way.
- v0.1.1 (commit d4faa9a): if one access of a function can't move, the whole function is left
  alone. Fixed the crash, but left 20 functions (1037 accesses) unprotected.
- **Improved (uncommitted, in the working tree)**: dry runs find the accesses that can't move, then
  keep with them every access of the same function that overlaps their bytes, until nothing changes;
  a function that also reaches its red zone through `lea` or an index register is left alone as a
  whole. NHL 20: 308 accesses in 29 functions stay, 11,556 move, no split pairs (checked offline
  against the game image). Log line: "Red zone: 308 accesses in 29 functions stay where they are…".
  Tested in game on 2026-11-18 (runs fine; the HUD bug below is unrelated).

## Not a bug: the strip at the bottom of the match screen

- What it looked like: in matches, the picture seemed to stop about 5% short of the bottom.
  Underneath was a black strip with 7 yellow boxes labelled "show", "warn", "b/warn", "drop",
  "b/drop", "comp", "b/comp", and it cut into the bottom of the score clock.
- Cause: NHL 20 assumes a TV that crops the edges of the picture (overscan). By default the score
  clock sits high and the panel below it is meant to be off-screen; a PC monitor shows it.
- Fix, in the game: **Settings → Calibration → Score Clock Calibration**. Press down until the
  yellow bar is gone, then confirm. The value is saved with the profile.
- How it was found (2026-11-18):
  - draw trace and frame capture: masks, scissors and the HUD layer were drawn as the game asked;
  - live shader skips (`BB_SKIP_FILE`): the panel is HUD shaders `b8aeee04` and `1a3e7913`, and
    its labels come from the other HUD text shaders.
- Ruled out as causes on the way: the red-zone changes, the upscaler, the safe area, compositing.
- If it should work without the player doing anything, two options:
  - write the calibration into the profile save (where the value is stored isn't known yet);
  - make those two shaders a built-in skip for NHL 20, as a stopgap. The labels stay visible
    unless the score clock is moved down.
- `BB_ONLINE_SERVER=` (network mode off) is not usable: the game crashes after the splash screen
  (read of 0x760 in SimThread, guest offset 0x3095d50), as NHL 19 did.

## Diagnostics in this build

- `BB_T4_TRACE=2` (added 2026-11-18, uncommitted, `vk_rasterizer.cpp`): logs 4096 draws, with
  targets, viewport, scissor, depth and stencil state (ops, compare, ref, masks), blend.
  - It starts when the file `out/t4` appears (`BB_T4_TRIGGER` changes the path). Or use
    `BB_T4_AFTER=<seconds>`: it starts that long after the first draw. `BB_T4_AFTER` isn't built yet.
  - `BB_T4_TRACE=1` is the old portrait-target trace.
- Frame capture (already built):
  ```
  BB_CAPTURE_TRIGGER=out/cap BB_CAPTURE_ON_FLIP=1 BB_CAPTURE_DIR=out
  ```
  `touch out/cap` writes `out/frame_<time>.txt`: every pass and compute dispatch of one frame,
  with shaders, targets, and images sampled/written.
- `BB_REDZONE_SHIFT=0` turns red-zone protection off (for comparison); `BB_FRAME_STATS=1`,
  `BB_LOG_UNRESOLVED=1`, `BB_SSE4A_LOG=1`: see the README's settings table.
- Offline check of the red-zone patcher: build `src/runtime_fp.c` with Zydis into a small harness
  that loads the image from `out/boot.bin` (header `BBPROBE2`; relocations of 32 bytes end at
  0x795200, the image follows) and calls `runtime_redzone_patch_image`. Delete `boot.bin` copies
  after use.

## Shared code: the other ports

The same `src/runtime_fp.c` is in NHL 19 (`NHL-19-PS4-PORT-WINDOWS`), NHL 21, NHL 24 and Madden 17.

- The improved red-zone fix was copied into all of them.
- **NHL 21**: committed and pushed (f103c3b), release notes for v0.1.2 pushed (d95aa64). The
  release is not built yet. A tester's NHL 21 v0.1.1 crashed the same way (offset 0x885d71, kept
  access 0x885d5a). Their other log was a one-off "Device lost" on an RTX 3070 while the main menu
  loaded: cause unknown.
- NHL 19, NHL 24, Madden: the improved fix is in the working tree, uncommitted.

## Next steps

1. Tell players about Score Clock Calibration: add a note to the README and to the v0.1.2 release
   notes.
2. Release v0.1.2 (improved red-zone fix, already committed in 455a0b5), with notes in
   `docs/release-notes/v0.1.2.md`.
3. Fix the README warning box (1.10 → 1.20).
4. Commit and release the improved red-zone fix in the other ports: NHL 19, NHL 24 and Madden
   (NHL 21: v0.1.2 waits to be built).
