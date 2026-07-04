# Building & running Tekken 3 locally (Windows / macOS / Linux)

The pipeline is IDENTICAL to Kula World's. Follow the platform doc first —
it has the toolchain setup, clone/branch instructions, build commands, run
instructions, and keyboard controls:

- **Windows 11:** `docs/WINDOWS_BUILD_KULA.md`
- **macOS (Apple Silicon or Intel):** `docs/MACOS_BUILD_KULA.md`
- **Linux:** same as the Windows doc's steps 5–8 with the usual
  apt/pkg-config SDL2 toolchain (see `docs/beetle-linux.md` for the optional
  oracle).

Then apply these Tekken-specific substitutions.

## 1. User-supplied files (copyrighted — not in the repo)

Place your own dump in `games/tekken3/` (note: **multi-track** — track 1 is
data, tracks 2/3 are CD audio; all four files are required):

| File | What it is |
|---|---|
| `games/tekken3/Tekken 3 (USA) (Track 1).bin` | data track (~632 MB) |
| `games/tekken3/Tekken 3 (USA) (Track 2).bin` | CDDA music |
| `games/tekken3/Tekken 3 (USA) (Track 3).bin` | CDDA music |
| `games/tekken3/Tekken 3 (USA).cue` | cue sheet naming all three |
| `bios/SCPH1001.BIN` | your PS1 BIOS dump (same as Kula) |

## 2. Extract the boot EXE

The boot binary lives in a subdirectory on the disc: `\TEKKEN3\SLUS_004.02`
(LBA 25, 1,185,792 bytes). Extract it to `games/tekken3/SLUS_004.02` using
dumpsxiso/mkpsxiso or any Mode2/2352-aware ISO9660 reader (same procedure as
the Kula doc's step 4, different path/filename). Verify it starts with the
ASCII magic `PS-X EXE`.

## 3. Generate and build

```bash
# game C (BIOS C generation is unchanged from the Kula doc)
./recompiler/build/psxrecomp-game --config games/tekken3/game.toml

# runtime target is tekken3-runtime instead of kula-runtime
cmake --build runtime/build --target tekken3-runtime -j8

# run from the repo root
./runtime/build/tekken3-runtime
```

`games/tekken3/game.toml` is already on the branch (entry 0x80079C70, load
0x80010000, KSEG0 addresses — no KUSEG quirk here).

## 4. What to expect (verified vs not)

Verified on this branch (see `docs/tekken3-findings.md` for the full story):

- Boots via the recompiled BIOS, streams and MDEC-decodes the **intro FMV**,
  then renders the **attract-mode fight in real-time 3D**.
- This depends on the text-divergence guard (commit "runtime: text-divergence
  guard"): Tekken unpacks ~448 KB of code over its own EXE image at runtime;
  the guard detects the rewrite and routes that region to the dirty-RAM
  interpreter. You should see `psxrecomp: text guard armed (...)` on stdout
  at launch — if you don't, you're on a stale branch.

NOT yet verified (may be the next wall): menu input, starting an actual
fight, CDDA music (tracks 2/3), SPU audio. The rewritten region currently
runs interpreted, so expect reduced speed in engine-heavy scenes until it is
captured and compiled native.
