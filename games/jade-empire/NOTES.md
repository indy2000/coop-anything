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
