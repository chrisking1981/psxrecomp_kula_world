# The oracle methodology — how this project proves correctness

Written as a self-contained briefing for an AI agent (or human) joining the
project. Read this before debugging any "the game does X but should do Y"
problem.

## 1. What "the oracle" is and why it exists

A static recompiler has one fundamental question it must answer over and
over: **"what would real PS1 hardware have done here?"** Guessing is
forbidden (CLAUDE.md §2/§14). The BIOS disassembly and Ghidra tell you what
the *code* says; only an execution oracle tells you what the *hardware state*
looks like at runtime.

The oracle is **Beetle PSX** (the mednafen-psx libretro core) — a mature,
accuracy-focused PS1 emulator. It runs the SAME BIOS file and the SAME disc
image as our recompiled runtime. When the two disagree, Beetle is presumed
right; the divergence IS the bug lead. We do not use DuckStation (CLAUDE.md
§16: where Beetle has gaps, build new tooling — don't switch oracles).

## 2. Architecture: two independent processes, one wire protocol

Two separate binaries, both built from this repo:

| Binary | What runs inside | TCP debug port |
|---|---|---|
| `psx-runtime` / `<game>-runtime` | our recompiled BIOS+game | **4370** |
| `psx-beetle` | Beetle PSX core | **4380** (next free if taken) |

Both expose the **identical JSON-over-newline TCP protocol**: `ping`,
`read_ram`, `dump_ram`, `press`, `set_input`, `clear_input`, `pad_status`,
`get_frame`, `screenshot`, `screenshot_file`, `fntrace_*`, `sio_trace`,
`cdrom_cmd_dump` / `cdrom_cmd_reset`, `spu_voices`, and more. One request per
line, one JSON reply per line, e.g.:

```
{"cmd":"read_ram","addr":"0x800DB12C","len":16}
{"cmd":"press","buttons":49151,"frames":4}      // PS1 pad word, active-low
{"cmd":"screenshot_file","path":"/tmp/shot.png"}
```

**Any tool written against one port works unchanged against the other.**
That symmetry is the whole point: ask both sides the same question, diff the
answers. Implementations differ (psx-beetle uses libretro hooks; psx-runtime
uses recomp-emitted instrumentation) but the protocol never does.

**Why two processes and not one?** An earlier embedded design
(psx-beetleoracle.exe, retired 2026-05-05) ran both backends in one process
with shared input, in lockstep. They desynced constantly: once internal state
diverged, the same keypress drove them to different screens, and "compare on
the same press" became meaningless. Independent processes let you navigate
EACH side to the state you want on its own timeline, then compare
concrete state (RAM, command streams, screenshots) — never shared memory.

## 3. Building the oracle (Linux; see docs/beetle-linux.md)

The integration targets the last C++ tree of beetle-psx (upstream later
converted to C, which our hooks don't compile against). Pin the commit and
apply our instrumentation patches:

```bash
git clone https://github.com/libretro/beetle-psx-libretro.git beetle-psx
cd beetle-psx
git checkout 5759277b
patch -p1 < ../docs/beetle_wtrace_hook.patch      # RAM write-trace hook
patch -p1 < ../docs/beetle_sio_trace_hook.patch   # SIO byte-exchange hook
patch -p1 < ../docs/beetle_cdcmd_trace_hook.patch # CD-command dispatch hook
make platform=unix STATIC_LINKING=1 HAVE_LIGHTREC=0 -j"$(nproc)"
cp mednafen_psx_libretro.so libmednafen_psx.a     # ar archive despite .so name
cd ../runtime && cmake -B build -G Ninja -DPSX_LAUNCHER=OFF -DPSX_DEBUG_TOOLS=ON
ninja -C build psx-beetle
xvfb-run -a ./build/psx-beetle ../bios/SCPH1001.BIN --disc "<game>.cue"
```

Key files: `runtime/src/beetle_main.cpp`, `beetle_libretro.cpp`,
`beetle_debug_server.c`; the patches live in `docs/*.patch` so the pinned
checkout can always be re-derived.

## 4. The method — how a hunt actually goes

1. **Reproduce on both sides.** Boot the same BIOS + disc in psx-runtime and
   psx-beetle. Drive each independently (debug-server `press`, or just let
   attract mode run) to the state where our side misbehaves.
2. **Capture the same signal on both ports.** Pick the narrowest observable
   that brackets the symptom: CD command stream (`cdrom_cmd_dump`), SIO bytes
   (`sio_trace`), RAM contents (`read_ram`), function dispatches
   (`fntrace_dump`), pixels (`screenshot_file`).
3. **Diff.** The first place the streams disagree is the divergence. Work
   backward from it: what state input made our side take a different branch?
4. **If the signal you need doesn't exist — build it** (CLAUDE.md §15/§16).
   Add a hook to BOTH binaries under the SAME command name, commit the beetle
   side as a patch file in docs/. Never route around missing visibility with
   guesses; never "lean on" a broken tool.
5. **Fix in our runtime/recompiler, re-run, confirm the streams now match**
   (or the symptom is gone). The oracle is never patched to match us.

### Worked example — the Kula World CD wedge (real hunt from this repo)

Symptom: our Kula World froze at its first level load; Beetle played on.

- Built the CD-command hook (beetle_cdcmd_trace_hook.patch) because no
  existing signal showed WHICH CD commands each side issued. Exposed it as
  `cdrom_cmd_dump` on both ports.
- Diff: the oracle read level data with a clean
  `Setloc → SeekL → Setmode → ReadN → Pause` pattern. Our side issued
  NEITHER SeekL nor ReadN — only a GetStat/Init/Setloc/Pause retry loop.
- Timeline pinning: our side DID stream 307 sectors fine, then at frame ~890
  the BIOS ran a mid-game disc re-identify (ReadTOC + GetID) that the oracle
  NEVER runs. Same BIOS bytes on both sides ⇒ a *state* difference steered
  the branch, not the CD sim.
- Root causes found by chasing that state: GetID returned a hardcoded wrong
  region ('SCEI' vs the disc's real SCEE/SCEA), a bogus status bit
  (ADPBUSY pinned high), and a CD-controller version byte (0x97 PSone-era,
  where SCPH-1001's sub-CPU reports 0x94 09 19 C0) that set a kernel flag
  triggering the spurious ReadTOC. Each fix was verified by re-diffing the
  command streams against the oracle.

### Tooling built along the way (reusable)

- `PSX_CD_TRAP_CMD=<byte>` / `PSX_CD_TRAP_NTH=<n>`: the process raises
  SIGSTOP the instant the Nth matching CD command dispatches, so a debugger
  attaches at the exact moment with zero run-speed cost (a gdb conditional
  breakpoint is ~100× slower via ptrace). Implemented symmetrically in both
  binaries.
- `PSX_FNTRACE_ALL=1`: record every function dispatch from boot; on an
  abnormal PC=0 exit the runtime dumps the last dispatch chain to
  `psx_cps_exit_trace.json` — this is how the Tekken 3 wild-jump chain was
  captured.
- When the debug server is unavailable, `import -window root shot.png`
  against the Xvfb display still gives you pixels.

## 5. Rules that make this work (do not relax)

- **Never one process with both backends.** Cross-process TCP only.
- **Protocol symmetry is sacred.** New debug commands go into BOTH servers
  under the same name, or explicitly document why one side can't have it.
- **Beetle is the reference, never the patient.** Beetle patches add
  *visibility hooks only* — never behavior changes to make it agree with us.
- **Two broken implementations agreeing proves nothing** (CLAUDE.md §15).
  Direct verification against the oracle, or it didn't happen.
- **Unknown is acceptable, guessing is not.** If neither disasm, Ghidra, nor
  the oracle answers it, the answer is "I don't know yet" and the next step
  is building the tool that CAN answer it.
