# Jade Empire (2005 Xbox / 2007 PC) — session notes

Agent session: 2026-11-18. Goal: online co-op campaign with friends.

## 0. Triage

### Game identity (rule 5)
Observed on the owner's PC (read-only; install not modified):
- Store: Steam (`installscript.vdf` sets `flavour = Steam`). Steam app id / build id:
  unknown (appmanifest not read; it lives outside the granted folder).
- Install: `<Steam library>\steamapps\common\Jade Empire`
- `JadeEmpire.exe`: 1,140,224 bytes, SHA-256
  `0d25f18c028a7dffff9dd110b7be677f7e9b63e1cad86c1eb8e4e8e189c54fe5`,
  FileVersion 1.00, PrivateBuild 1510, PE timestamp 2007-01-23, x86 (32-bit).
- `JadeEmpireLauncher.exe` and `TestApp.exe` are byte-identical, SHA-256
  `6f6b93169f882101d8f2cdc577b6287e4cf9830eaa57d3319469e9ceab103262`.
- Engine: BioWare Odyssey derivative (files: `chitin.key`, `dialog.tlk`, `override/`).
- OS: Windows.

### Exe packing
Observed:
- `JadeEmpire.exe` is packed with PECompact 2 (`PEC2` marker in the section table, two
  sections, `.text` raw 0x10e200 vs virtual 0x494000, entropy 8.0, only 4 kernel32
  imports). Stub entry at VA 0x0089ce08.
- Emulating the stub (Unicorn, analysis only) shows the loader resolving 211 imports
  from: kernel32, user32, gdi32, advapi32, ole32, shell32, d3d9, d3dx9_29, dinput8,
  xinput9_1_0, dsound, winmm, binkw32, vorbisfile, dbghelp, **steam.dll**
  (`SteamStartup`, `SteamIsAppSubscribed`, `SteamCleanup`).
- No ws2_32 / wsock32 / dplay imports among those 211.

Derived:
- The PECompact loader wraps the legacy Steam (steam.dll) ownership check, i.e. it is a
  Steam DRM wrapper. Emulation was stopped at `SteamStartup`; we did NOT fake a
  successful check. Route chosen instead: dump the code from the game after the owner
  launches it normally through Steam (no DRM bypass), which is also how the co-op DLL
  will run.
- No networking imports → no shipped netcode; co-op needs a sync layer from zero.


### Prior work
Observations:
- `DIRECTORY.md` and `data/directory.yaml`: no Jade Empire entry. Only "jade" hit is
  TMNT (2007) on Ubisoft's unrelated *Jade* engine.
- `games/`: no `jade-empire` folder before this session.
- Repo issues (web search page, `q=jade`): no Jade Empire claim issue.
  `gh` CLI could not be used (no linked GitHub account), so open PRs were not read.
- Web: no co-op/multiplayer mod found on Nexus or ModDB; existing mods are content /
  texture / UI mods using the `override` folder.
- xoreos (GPL-3.0 Aurora/Odyssey reimplementation) lists Jade Empire as a target;
  public updates describe models, areas, partial main menu, partial XACT audio and some
  script work. Not playable; no netcode.
  Sources: https://www.xoreos.org/blog/2015/02/15/jade-empire-models-and-areas/ ,
  https://www.gamingonlinux.com/2018/07/the-open-source-project-xoreos-released-version-005-dawn-star/
- reone / McChaffeeFAF kotor-coop (see `games/kotor`) target KotOR 1/2 only.

Derived:
- Triage category: "Closed binary, no MP, no scripting extender" → build from zero,
  "very hard" per playbook/triage.md. Nearest reuse: KotOR co-op's host-authority design
  (same engine family, NWScript VM) — but JE has real-time action combat, unlike
  KotOR's round-based combat, so sync requirements are much stricter.

Unknowns:
- Owner's store/version; whether the PC exe has any leftover netcode or debug console.
- Whether xoreos' JE code is far enough along to be a base.

## 2. Workspace (2026-11-18)

Observed:
- Owner's copy is played through **Steam Family Sharing** ("Da biblioteca da sua Família
  Steam" on the game page). Fresh install, first launch this session.
- Steam shows two launch options: the game and "Launch Jade Empire Configuration".
  Play goes through `JadeEmpireLauncher.exe` (Play / Configure / Advanced / Exit).
- First launch wrote `JadeEmpire.ini` (ScrW=2560, ScrH=1080, 60 Hz). No log files written
  (`logs/` holds only `dummy.txt`).
- Two launches under computer-use control: the game window appeared, then the process
  ended within ~10–30 s with no error dialog seen. Cause unknown. Candidates (guesses):
  computer use hides/minimises non-granted windows (exclusive-fullscreen D3D9 may not
  survive that); 2560x1080 ultrawide; legacy steam.dll ownership check under Family
  Sharing. Next: owner launches it by hand to tell these apart.
- Scylla v0.9.8 (official release https://github.com/NtQuery/Scylla/releases/tag/v0.9.8,
  no upstream hash published). Our pin: `Scylla_v0.9.8.rar` SHA-256
  `48a4338d999ec5f33b5964c51893a04fc9e2d104b0c7786f50751f7db5dcbe52`,
  `Scylla_x86.exe` SHA-256 `4b49b0827f447d8d272c4f118c7c0afd2c786c12751e8e6e32e8477586afcc75`.
- Repo check: no Jade Empire issue or PR (web views of issues and PRs, 2026-11-18).

### Owner's fix and the code dump (2026-11-18)
Observed:
- Owner applied a community Steam guide fix: the game exe was renamed to
  `JadeEmpireLauncher.exe` (SHA-256 unchanged, `0d25f18c…c54fe5`) and the original
  launcher kept as `JadeEmpireLauncherBackup.exe`, so Steam starts the game directly.
  After that the game reached the main menu and stayed up (fullscreen 2560x1080).
- Dump taken from the running process (launched normally through Steam, at the main
  menu) with Scylla x86 v0.9.8: IAT at VA 0x006E1000, size 0x350, 199 imports, 0 invalid,
  0 direct imports. Image base 0x00400000, image size 0x49D000. OEP left at 0x00401000
  (the stub jump) — the dump is for static analysis only, not meant to run.
  `JadeEmpire_dump_SCY.exe` SHA-256 `94611bb37ed87cde8a96c2c599735ca60d6b1a2886bd63c47ebae724ba9304c2`
  (owner's PC only; never committed).
- Strings: 121 source file names (`.\ccreature.cpp`, `.\cplayercreature.cpp`,
  `.\aimaster.cpp`, `.\ccombatstyles.cpp`, `.\cobjectstorage.cpp`, `.\carea.cpp`,
  `.\cconversation.cpp`, `.\cappbase.cpp`, …) and 247 `Class::Method` strings — good
  anchors for naming functions.
- NWScript VM present (`CVirtualMachineCommands::…`, "This nwscript command is not mapped
  to a function #"), same lineage as KotOR.
- Debug hooks: `%sdebug.ini`, `debugpanel`, `fnt_console`, "Console queued animation …".
  Build tag `2005-3-14__1915` / "Automated Build".
- Xbox heritage: `CXbInput::GetGUIInput()`.
- No socket / network strings from the game itself (only Windows error-name tables).

Derived:
- `debug.ini` + `debugpanel` may give a developer console → candidate for the in-game
  bridge / bots (playbook/in-game-bridge.md). Needs checking in Ghidra.

## 3. Understand — first Ghidra pass (2026-11-18)
Tools: Ghidra 12.2 (official release, SHA-256 `2c2cd776…a7be67`, matches the release
page), Temurin JDK 25.0.5+8 (SHA-256 `68982529…78d7fb`, matches Adoptium's .sha256.txt).
Script: `ExportAnchors.java` (auto-names functions from `Class::Method` strings,
decompiles functions that reference anchor strings).

Observed (dump base 0x400000):
- 16,127 functions, 9,062 defined strings; 173 functions auto-named from single
  `Class::Method` string references (e.g. `CAppBase::Update` @ 0x0040CA50).
- Engine init @ 0x0040AC40: builds `<game dir>\debug.ini` path with sprintf and loads it
  into an INI object, then mounts resource roots (LOCALIZEDROOT, Fonts, Movies,
  Override, `OVERRIDE:`, `HD0:Data\loadscreens`, `globalunload_a`, `miniglobal_a`,
  `dialog`).
- A developer debug menu still exists: GUI panel class built @ 0x005262F0 that loads the
  `debugpanel` GUI (controls `lTitle`, `lLegend`, `lsChoices`) from `OVERRIDE4:debugpanel`
  and registers 6 input handlers.

Unknown:
- Which keys the game reads from `debug.ini`, and how/if the debug panel can be opened in
  the retail build. Next target: callers of the INI getter on the debug.ini object, and
  callers/constructor sites of the panel @ 0x005262F0.

### Developer debug panel (2026-11-18, Ghidra on the Scylla dump)
Observed:
- Panel class ctor @ 0x005262F0 (0xA30 bytes, created by 0x00448870, stored at
  `app->guiManager(+0xB4)->+0x78`). It parses an INI-style menu file
  `OVERRIDE4:debugpanel` with the engine's own text-INI reader (0x0065BE80 load,
  0x0065C7D0 count, 0x0065C650 enumerate sections, 0x0065C5B0 get value). Each section's
  `file` key names a sub-menu file (`OVERRIDE4:%s`), loaded by 0x00525E90.
- On selection (handlers 0x00525B10 / 0x00525CD0) it reads the item's `command` key; a
  `t/f` token is replaced with ` 1` or ` 0` (toggle items), then the panel closes and the
  game unpauses (0x0040BE70 = set-paused on the app object, flag at app+0x3C).
- It is opened from the in-game input handler 0x00448FC0 (switch case 6 @ 0x0044A3CC)
  only if `DAT_006F28A0 != 0`, global @ 0x0076C58C != 1, and game-state flag 0x3D is
  clear. `DAT_006F28A0` is 0 in .data and has **no writes** anywhere (one read only):
  the menu is compiled off in the retail build.
- `OVERRIDE4:` resource root and the `debugpanel` GUI layout are not shipped (unknown
  whether the GUI resource exists in the game's data; to check with a resource lister).

Derived:
- A DLL can set `0x006F28A0 = 1` at runtime to re-enable the panel, and supply its own
  `debugpanel` menu files. Where the `command` string goes after selection is not traced
  yet — if it reaches a script/console executor, this is a ready-made command channel for
  the in-game bridge and bots.

Unknown:
- Which input opens it (case 6 of the handler's event switch), what executes `command`,
  and what `debug.ini` controls (loaded at init 0x0040AC40 into a stack-local reader).

### Debug panel is a shell; script runner found (2026-11-18)
Observed:
- In the panel's select handler (0x00525CD0, disassembly 0x00525D47–0x00525E71) the
  `command` string is read into a stack-local CExoString, the `t/f` toggle is applied,
  and the string is then destroyed. No call consumes it; the panel closes and the game
  unpauses. The command executor appears compiled out of the retail build.
- Owner reports (from community reading, unverified here) that **K** opens the debug
  menu. The code only shows it as event index 6 of the action loop in 0x00448FC0;
  key mapping not traced.
- Script runner: `0x00697840` is called as `thiscall(vm, CExoString *resref,
  uint32 oidSelf, int 1)`, returning 0 on failure (then "Problem running script" via
  OutputDebugStringA, 0x0044E610) — same shape as NWN/KotOR
  `CVirtualMachine::RunScript`. `vm = *(*(int*)0x0076C924 + 0x9C)` (app object global
  0x0076C924). Return value read with `0x006948C0` (type 3 = int).
  Second caller: campaign script loader 0x00425780 (`ccampaign.cpp`, "Bad script").
- App object global `0x0076C924`: +0x3C paused flag, +0x9C script VM, +0xB4 GUI manager.

Derived:
- The bridge for bots should call `RunScript` directly with our own compiled `.ncs`
  scripts (from a test-install `override/`), instead of relying on the debug panel.

Unknown:
- An NWScript compiler that targets Jade Empire's function table (nwscript.nss for JE),
  or whether to hand-assemble `.ncs`. To research.
- Whether a copy of the install outside Steam's folder still passes the steam.dll
  check (needed for the isolated test install, rule 2). Install size: 6.9 GB.

Toolchain available in the agent workspace: clang 18 + lld-link + llvm-dlltool can build
a freestanding 32-bit Windows DLL (no install needed on the owner's PC).

## 2b. Test install and first in-game bridge (2026-11-18)
Test install (rule 2): `%USERPROFILE%\Downloads\JadeEmpireCoop\test_install`, a copy of the
install without `movies/` (4.3 GB) and `sound/` (1.2 GB; a few sound files from an
aborted first copy are present). File list and `chitin.key` / exe hashes match the
original. The copy's `JadeEmpire.ini` (original kept as `JadeEmpire.ini.orig`):
ScrW=1280, ScrH=720, Windowed=1, NoMovies=1, NoSound=1, NoLogos=1.

Observed:
- Started directly (not from Steam), the copy shows "Failed to find Steam." Fix: copy
  Steam's own `steam.dll` from the Steam folder into the copy (the owner did it). The
  ownership check still runs against the Steam client. After that the copy reached the
  main loop.
- The copy does not need the original's admin/compat flags to start.

### jebridge v0 (proxy `xinput9_1_0.dll`)
Code: freestanding C (no CRT), clang 18 `--target=i686-pc-windows-msvc` + lld-link,
16 KB. The game's own `xinput9_1_0.dll` (SHA-256 `9c14c5af…bd0e0cf6`) is renamed
`xinput_orig.dll` in the test copy; the proxy forwards XInputGetState/SetState/
GetCapabilities to it in code (lld-link mangled forwarder exports on x86 to
`_xinput_orig.*`, so `.def` forwarders were not used).
- Main-thread pump: IAT hook of `PeekMessageA` in the IAT range 0x006E1000–0x006E1350,
  pumping only on the first thread that calls it (frame counter = distinct GetTickCount
  values).
- Protocol: coop-anything bridge v1 subset — `ping`, `version`, `methods` on the transport
  thread; `je.state`, `je.setDebugPanel`, `je.runScript {resref, oid}` on the main thread.
- Transports: TCP 127.0.0.1:47310.. (endpoint file `coopbridge.json`) and a **file
  transport** (`bridge\in\*.json` → `bridge\out\*.json`), added because the agent's shell
  runs in a VM that cannot reach the owner's loopback, and the bridge must not listen on
  the LAN.
- Test script `jebr_noop.ncs` (`void main(){}`: header `NCS V1.0` `B` + size, `JSR +8`,
  `RETN`, `RETN`; header and entry stub checked against the game's own
  `override\j00_a_chest_1.ncs`).

Results (pid 4648, the run after steam.dll was added):
- `je.state` → `app 0x02850048, paused 0, vm 0x062F10A0, debugPanelFlag 0`, thread ==
  main thread (31324).
- `je.runScript {"resref":"jebr_noop"}` → `rc 1` (script found and run) on the main thread.
- `je.runScript {"resref":"jebr_doesnotexist"}` → `rc 0`.
- unknown method → -32601; missing `resref` → -32602.
- `ping` → frames 429, mainLoopAgeMs 16, ready true.

Bug found and fixed: these requests had been written while an earlier copy was stuck at
the Steam error; the next process picked them up and ran them during start-up. v0.1 now
deletes request files left over from an earlier run when it starts.

Open problem: about 5 s after that ping the copy stopped answering even transport-thread
requests and the DLL file was no longer locked, so the process had ended. Cause unknown
(crash vs. closed; owner asked). Candidates: missing/partial `sound` files despite
NoSound=1, the hook, or something unrelated.
