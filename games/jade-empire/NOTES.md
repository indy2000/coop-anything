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

The run ending ~5 s after that ping was the owner closing the copy (confirmed), not a crash.

### jebridge v0.1 check run (pid 9164, 2026-11-18, file transport from the agent VM)
On start the DLL dropped the 8 leftover request files (log: "dropped stale request files
from an earlier run"). `checks.py` (11 checks):
```
PASS ping ready / version build / state on main thread / runScript noop rc=1 /
     runScript missing rc=0 / runScript no resref -32602 / unknown method -32601 /
     debug flag on / debug flag read back / 20x runScript (2.59 s) / frames advance after tests
SUMMARY: 11/11 passed
```
These checks have no mutation twins yet (playbook/in-game-bridge.md §6): to do before
calling the bridge tested.

### Script function table, return values, first state reads (2026-11-18)
Observed:
- The game ships `override\nwscript.nss` (engine function declarations, 1,058 functions)
  and 143 `.nss` sources + 100 `.ndb` debug files in `override\`. Routine number =
  declaration order; early numbers match NWN/KotOR (PrintString #1, AssignCommand #6,
  ExecuteScript #8, FloatToInt #231). Useful ones: GetPlayer #548, GetHealth #49,
  GetMaxHealth #50, GetPosition #27, GetArea #24, GetIsObjectValid #42,
  GetGlobalBoolean #776 / SetGlobalBoolean #777, CreateObject #243, JumpToLocation #313.
- BioWare left cut multiplayer routines in the table with a `ZZ_CUT_` prefix:
  `ZZ_CUT_GetNextPC` #549, `ZZ_CUT_AddToParty` #572, `ZZ_CUT_RemoveFromParty` #573,
  `ZZ_CUT_SendMessageToPC` #374, `ZZ_CUT_GetPCPlayerName` #371,
  `ZZ_CUT_GetLastPlayerDying` #410, `ZZ_CUT_GetModuleName` #561. Whether the engine
  still implements them: unknown.
- No `debugpanel` GUI ships in `override\` (58 other .gui files do). With the flag at
  0x006F28A0 set to 1 (confirmed by read-back), pressing K in game did nothing (owner
  test). Panel not pursued further.
- Return values: `0x006948C0` = `thiscall(vm, int *type, int *value)` → bool (call site
  0x0044E740); type 3 = int. Exposed as `je.evalScript {resref}` in jebridge v0.2.
- `override\` is indexed at game start: scripts added while running are not found
  (rc 0). Overwriting an already-indexed script while running did not change its
  behaviour either (probably cached after first run; unverified).
- Hand-assembled probes (`ncsasm.py`; pattern `RSADDI; JSR 8; RETN; …; CPDOWNSP -8,4;
  MOVSP -4; RETN` matches the game's own `j04_c_gold_14b.ncs`). Results on the owner's
  new game (pid 40804): `jebr_const` → 4242, `GetIsObjectValid(GetPlayer())` → 1,
  `GetHealth(GetPlayer())` → 120, `GetMaxHealth` → 120. The position probe first used
  the wrong opcode for MUL (0x10 = LEQ; MUL is 0x16) and returned rc 0 without harming
  the game; fixed version not yet verified (needs a restart).

Next: serve bridge scripts from memory instead of `override\` (hook the resource
lookup) so probes can change without restarting the game.

### In-memory scripts, no restarts (jebridge v0.3, 2026-11-18)
Observed (Ghidra on the dump):
- `RunScript` 0x00697840 loads code with `0x00695600` = `thiscall(vm, CExoString *resref)`
  (0 = ok, -0x5E = too deep (depth > 7), -0x5F/-0x60 = not found/bad file). It asks the
  VM's resource loader (vm+0x3CC, vtable +4 demand, +0x18 release) for the NCS, checks
  `NCS V1.0B`, then copies the code after the 13-byte header into script slot
  `vm + 0x10 + depth*0x28` with `0x00694800` = `thiscall(vm, slot, code, len)` and executes
  with `0x00697360`. Depth counter at vm+0x0C. Slots are freed by `0x00695B20`.
- `0x00695600` has exactly three callers (E8 rel32): 0x00697872 (RunScript),
  0x0069583C, 0x006974DE.

Built: jebridge v0.3 (`xinput9_1_0.dll` SHA-256 `09e3ea9f…2eda5d53`) re-targets those three
call sites to a replacement that serves names registered with
`je.loadScript {name, hex}` (whole NCS file, ≤16-char name) from the DLL's memory and
passes every other name to the original loader. It checks each site is `E8 → 0x00695600`
before patching.

Check run (pid 21732, owner controlling the character in a new game), `memtest.py`:
```
PASS server 0.3 / load jm_pcok, jm_const, jm_px, jm_maxhp, jm_py, jm_pz, jm_hp (names not in override)
PASS mem const = 4242 / mem hp > 0 (120) / mem position (-60.257, 26.44, 23.839)
PASS hot replace -> 7 (same name, new bytes, no restart)
PASS bad hex rejected / unknown mem name -> not run / override path still works / game still running
SUMMARY: 16/16 passed
```
Position tracking while the owner walked for 16 s (1 Hz): positions changed while moving,
held while standing, 14.04 m travelled in the xy plane.

Derived: the bridge can now read and change game state live through the game's own
1,058 script functions with no restart, which is the basis for bots and for the co-op
sync layer. Still missing: mutation twins for these checks; a script compiler (only
hand-assembled probes so far).

### First bot actions (2026-11-18, jebridge v0.3 + client `ncs.py` / `bot.py`)
All scripts are hand-assembled NCS sent with `je.loadScript` (no restarts). Routine numbers
from the game's `nwscript.nss`. Patterns used:
- run as a creature: `ExecuteScript(oTarget, sScript)` (#8, JE argument order: object
  first); the executed script uses OBJECT_SELF = CONSTO 0.
- remember objects across scripts: `SetLocalObject(GetPlayer(), 190, o)` /
  `GetLocalObject(GetPlayer(), 190)` (#58/#54; JE locals are numbered). Slot numbers
  190/191 chosen by us; collision with game use unknown.
- location: `Location(GetArea(x), [fx, fy, fz], facing)` (#215/#24); vector = 3 CONSTF
  with z pushed last.

Observed:
| Action | Result |
|---|---|
| `JumpToLocation` on the player (#313) | teleported exactly (0.00 m error) |
| `ActionMoveToLocation` / `ActionMoveToObject` / `ActionForceMoveToLocation` on the player | ignored while the player has control (outside cutscenes) |
| `CreateObject(2, "j00h_abbot0", loc)` (#243) | creature spawned at the requested point, `GetIsObjectValid` = 1 |
| `ActionJumpToLocation` on the spawned creature (#214) | executed via its action queue (0.00 m error) |
| `ActionMoveToLocation` on a level NPC and on the spawned creature, game window focused | walked 3 m, stopped 0.15 m from target in ~1.5–2 s |
| same, game window **not** focused | accepted, but nothing moves |
| `DestroyObject(o, 0.0)` (#241) on the spawned creature | still valid afterwards (focused and unfocused). Cause unknown. |
| `GetNearestObject(GetPlayer(), 2, n)` (#227) | 0-based; index 0 was our spawned creature |

Derived (CORRECTED below — the focus explanation was confounded, see "Focus and pause"):
- ~~The game freezes world simulation while its window is unfocused~~: the before/after
  test changed two things at once (window focus and which NPC was nearest).
- The remote player's puppet in co-op will be an ordinary creature (spawned, driven by
  action queues), which works. The local player is moved by its own input; bots for the
  local player need input injection or cutscene mode (to research).

### Focus and pause (jebridge v0.4, 2026-11-18)
Observed (Ghidra): window procedure 0x005D9320 (registered via 0x005D9640 from
0x004A3B21). On `WM_ACTIVATE` (6) it writes `0x0075ECDC` = 0 (inactive, logs
"aurlite.cpp NULL #1!!!") or 1 (active, "aurlite.cpp SET #1!!!"). ~20 functions read that
flag, including `CGameObject::GetPosition` 0x00434430 and the app update 0x004A3A40.

jebridge v0.4 (`xinput9_1_0.dll` SHA-256 `dd281b5d…34647ab6`) adds `je.keepActive {value}`
(writes 1 to 0x0075ECDC every frame while on), a startup marker `bridge\keep_active`, and
`appActive`/`keepActive` in `je.state`.

Observed (owner's game unfocused, pid 39188):
- `je.state` while unfocused: `appActive 0`, `paused 1` (app+0x3C).
- `je.keepActive 1` → within 0.5 s `appActive 1`, `paused 0`. After `je.keepActive 0`
  the flag stays 1 until the next WM_ACTIVATE (event-driven).
- `focustest.py` 5/6: FAIL "unfocused, keepActive off: NPC does not walk" — a level NPC
  walked 2.85 m with the game unfocused and keepActive off (expected not to). PASS the
  rest (keepActive on/off, NPC walks with keepActive on, game still running).
- A third spawned abbot (spawned at player + (2, −1, 0), ended at z 23.28 vs player
  23.84) did not walk and was not destroyed, both with `paused 1` and after keepActive
  made `paused 0`. The second abbot (player + (2, +1, 0)) had walked.

Derived: an inactive window makes the game pause itself (app+0x3C), and keepActive
undoes that. That alone does not explain the walking failures.

Guesses (unverified): spawn points off the walkmesh stop pathing; `DestroyObject`
(#241) needs a different call (signature or a JE-specific flag). Next: spawn at
`GetLocation(GetPlayer())` + an offset found by `GetNearestObjectToLocation`/walkmesh
query, a controlled A/B with one creature, and check `DestroyObject` against a game
script that uses it.

Test copy state: 2–3 test abbots are left standing in the test copy's prologue (not
saved; gone on reload).

## Session 2 (2026-11-18 afternoon)
Start-of-session check: fork up to date; repo search (API, `is:issue`/`is:pull-request`
"jade") shows only our claim #407 (no comments) and two closed TMNT PRs (Ubisoft Jade
engine, unrelated).

### Spawned creatures: why they didn't walk, and how to remove them
Observed (`nwscript.nss` comments):
- `DestroyObject` (#241): "This will only deactivate creatures, placeables, doors and
  triggers". So `GetIsObjectValid` = 1 after it is expected.
- `CreateObject` (#243) documents projectiles, placeables and stores; creatures are not
  listed. JE toggles pre-placed creatures with `SetObjectActive` (#756, 6 args:
  object, bActive, spawnAnim, despawnAnim, crust, durationMs) / `GetObjectActive` (#755).

Observed (owner's game, test copy, `bridge\keep_active` marker on):
- A spawned `j00h_abbot0` is born **dead**: `GetIsDead` 1, `GetHealth` 0, `GetMaxHealth`
  10000, `GetObjectActive` 1. `SetObjectActive(TRUE)` and `SetHealth(100)` (#156) do not
  clear "dead". Dead creatures ignore move actions. Level NPCs: dead 0, health 25,
  AI level 2 (our spawn: AI level 3).
- Spawning a template whose model the current area has not loaded triggers an engine
  assert and closes the game: "Exo Assert … ccreatureappearance.cpp line 354 …
  CCreatureAppearance::CreateBodyAppearance(): Model not found N_Bandit_" (template
  `j02c_banditc` in the prologue area). The client now only spawns templates from an
  allowlist verified per area (`ncs.SAFE_TEMPLATES`, currently `j00h_abbot0` for the
  prologue).
- `restest.py`, same creature, one change at a time:
```
PASS fresh spawn is dead (known)            {"dead": 1, "hp": 0, "active": 1}
PASS dead spawn does not walk (control)     {"moved": 0.0, "toTarget": 3.0}
PASS alive after resurrection               {"dead": 0, "hp": 10000, "active": 1}
PASS walks after resurrection               {"moved": 2.86, "toTarget": 0.15}
PASS walks again                            {"moved": 2.36, "toTarget": 0.14}
PASS deactivated                            {"dead": 0, "hp": 10000, "active": 0}
PASS game still running
SUMMARY: 7/7 passed
```
  Resurrection = `ApplyEffectToObject(o, DURATION_TYPE_INSTANT=0, EffectResurrection(0), 0.0)`
  (#220, #82).
- Earlier "walking depends on place" idea was wrong: a spawn next to a walking level NPC
  also failed while dead.

Derived: puppet lifecycle for the remote player = CreateObject (area-safe template) →
EffectResurrection → action-queue moves → SetObjectActive(FALSE) to remove. The
"dead spawn does not walk" control doubles as the mutation twin for the walk check.

Corrects session 1: the second abbot "walked" and the third did not; with every spawn
born dead, which creature was measured in session 1 is uncertain (slot 190 was
overwritten by each spawn). Treat session-1 spawned-creature results as unreliable.

### Follow test: the remote-player avatar in one game (automated, no human input)
`followtest.py`: spawn → resurrect a puppet next to the player; teleport the player
(`JumpToLocation`) around a 6 m square (4 legs); after each jump the client polls the
puppet at 4 Hz and orders `ActionMoveToLocation` to a point 1 m beside the player (run if
> 4 m away) until it is within 1.6 m (limit 8 s). Each leg also requires the puppet to
have moved > 2 m (added after the first mutation run showed leg 4 could pass with a
puppet that never moved, because the path ends where it spawned).
```
== real
PASS leg 1  moved 3.58 m, 1.93 s, gap 1.06 m, 1 order
PASS leg 2  moved 5.93 m, 3.24 s, gap 1.19 m, 1 order
PASS leg 3  moved 6.20 m, 3.22 s, gap 1.48 m, 1 order
PASS leg 4  moved 5.96 m, 2.65 s, gap 1.07 m, 1 order
SUMMARY: 4/4 passed
== mutation (no resurrection)
FAIL legs 1-4  moved 0.0 m
SUMMARY: 0/4 passed
```
Observed: `JumpToLocation` on the player lands up to ~1 m from the requested point
(probably snapped to the walkmesh); the test reads the actual position after the jump.

### Two game instances on one PC (jebridge v0.5)
Observed (Ghidra):
- Single-instance guard in 0x004A3A40: `OpenMutexA(…, "JadeEmpire")` (push at
  0x004A3A6B); if it does not exist, `CreateMutexA(…, "JadeEmpire")` (push at 0x004A3A9A)
  and the game initialises; otherwise it skips initialisation. The same string
  (0x006EC73C) is also passed to 0x00598030 from 0x004A53B0 (app name), so the string
  itself is not changed.
- 0x004A53B0 also calls `FindWindowA("JadeEmpPC", "Jade Empire")` and exits if found, but
  the game registers its window class as "JadeDirect3D" (0x005D9640), so this check
  never matches the game's own window.

Built: jebridge v0.5 (`xinput9_1_0.dll` SHA-256 `6412d829…41d86bda3`): with the marker
`bridge\multi_instance`, `DllMain` re-points both mutex pushes to a per-process name
(`JadeEmpire_jeb_<pid>`) after checking they are `68 3C C7 6E 00`. DllMain runs while the
PECompact stub resolves imports, after it has unpacked `.text`, so the patch lands before
WinMain.

Second isolated test install: `%USERPROFILE%\Downloads\JadeEmpireCoop\test_install_b`,
copied from `test_install` (5,434 files, file lists match, exe SHA-256 unchanged), with its
own `bridge\` (keep_active + multi_instance). The bridge's TCP endpoint moves to the next
free port (A 47310, B 47311). The file transport is per install folder.

Observed: both copies started at once (pids 9008 and 24656), both logged
"multi_instance: single-instance mutex made per-process", both had a valid player in the
same area at the same position.

`crossfollow.py`: the relay (agent client) reads A's player position at 4 Hz and drives a
resurrected puppet in B; A's player is teleported around a 6 m square.
```
== real
PASS leg 1  moved 3.96 m, 3.23 s, gap 1.03 m, 1 order
PASS leg 2  moved 4.81 m, 3.27 s, gap 0.95 m, 1 order
PASS leg 3  moved 5.09 m, 2.27 s, gap 1.52 m, 1 order
PASS leg 4  moved 5.84 m, 3.18 s, gap 0.95 m, 1 order
PASS B's own player untouched
SUMMARY: 5/5 passed
== mutation (relay off)
FAIL legs 1-4 (moved 0.0 m); PASS B's own player untouched
SUMMARY: 1/5 passed
```
Derived: the core of "remote player shown as a puppet" works between two real game
processes. The transport is still the agent's file relay over a VM, not networking; the
relay runs outside the game.

Next: move the relay into the DLL (game-to-game over UDP, then Steam relay for friends),
puppet facing/animations, and the hard part — host authority for enemies and combat.

### Mod code repository
Our code now lives in https://github.com/indy2000/jade-empire-coop (MIT; no game files):
`dll/` (jebridge, reproducible build with `/Brepro`: three builds, two folders, same
SHA-256 `ebab8eee…46b593c7` for the v0.5 source), `tools/ncsasm.py`, `client/` tests.
Note: the v0.5 DLL deployed in the test installs (`6412d829…`) was built before `/Brepro`,
so its hash differs from the repo build of the same source.

### Game-to-game sync without the agent relay (jebridge v0.6, `dll/netsync.inc`)
Design: with `bridge\net.txt` (`local <port>`, `peer <ip> <port>`, `template <resref>`)
and after `je.netStart`, each game reads its player position with DLL-built in-memory
scripts, sends it over UDP (24-byte packet, int32 millimetres), and drives the other
player's puppet itself (spawn from the area-safe template → EffectResurrection →
ActionMoveToLocation to 1 m beside the remote player; JumpToLocation when > 15 m). Local
object slot 192 holds the puppet. `je.netStatus`, `je.netPause` (test mutation). With a
127.x peer the socket binds 127.0.0.1 only (no firewall prompt).

Observed, in order:
1. First v0.6 started scripts from the first armed frame: both copies closed ~2 s after
   start (log stopped right after "net on"). Changed to run nothing until `je.netStart`.
2. Armed on A only: A closed right after the breadcrumb "net: first player eval". Cause
   not identified. The C NCS builder output was compared byte for byte with the Python
   builder (identical).
3. Added a vectored exception logger (first 5 access violations, with stage), and a
   guard to run scripts only when the VM is idle (depth −1 at vm+0x0C). Next run: no
   crash; the guard never triggered (`vmBusySkips` 0), so it does not explain step 2.
   The logger later recorded one access violation per copy at eip 0x00487327 reading
   address 0x8, stage `jbn_pcok`, in the run the owner closed (likely during shutdown,
   unconfirmed).
4. Packets flowed both ways and each game spawned the other's puppet, but the received
   y was −1 mm in both directions while x and z were right; `jbn_p1` evaluated through
   the bridge was correct (30459). The puppet was therefore spawned at y ≈ 0. Rebuilt
   with an int wire format, values kept in memory, frame pointers: still y = −1.
5. Next run, **the same binary** (`4070cc8a…`, confirmed by hashing the file staged from
   the PC) sent y correctly. The y = −1 bug is therefore **intermittent and unexplained**.
   The guess "several scripts in one frame" is not confirmed. A one-script-per-frame
   rewrite (`1973dcd2…`) was built but not deployed: the file-commit tool wrote a cached
   earlier file for the reused staging path. Lesson: give every build a unique staging
   file name.

`crossnet.py` (binary `4070cc8a…`, both copies loaded the same save, same area):
```
== real
PASS net on, sending, receiving (A and B) / B and A each spawned the other's puppet
PASS leg 1-4  A moved 6.0 m each leg; B's puppet moved 5.85-5.91 m, 2.62-2.67 s, gap 0.91-1.09 m
PASS B's own player untouched
SUMMARY: 9/9 passed
== mutation (je.netPause on B)
FAIL legs 1-4 (puppet moved 0.0 m); PASS the 5 checks that do not depend on B's sync
SUMMARY: 5/9 passed
```
A first run of the test failed legs 1 and 3 because it measured the puppet's start
position after A's teleport; the DLL reacts within ~100 ms, faster than the bridge read.
The test now measures before the teleport.

Open: the intermittent y = −1; test the one-script-per-frame build; Steam relay for
friends on other PCs; facing and animations; enemies and combat.

### v0.6.1–0.6.3: y = −1 investigation, isolated tests, stuck puppets
Deployment check: since the cached-file mishap, every build is staged under a unique name
and the file on the PC is hashed after deployment (0.6.1 `7dcf223e…`, 0.6.2 `0230acb1…`,
0.6.3 `0612f388…`; each matches the reproducible repo build of its commit).

y = −1:
- `je.evalMany` (0.6.1) runs several scripts back to back in one main-thread call.
  `yinvest.py`: 7 orderings/combinations of the position scripts × 10 runs on both copies
  all returned correct values. **Refuted:** "several scripts in one frame" causes it.
- With 0.6.1 armed: sent = actual position in every sample, both players at the same
  spot, after teleports to different spots, and while the owner walked A by hand (45
  samples, 0 with y ≈ 0). **Not reproduced since; cause unknown.**

Test isolation: one `crossnet.py` run failed 5/9 after the owner had walked A near the
fences: the test started from A's current position, the puppet partly got stuck, and
the following mutation run "passed" legs because the puppet was still finishing earlier
orders. The test now teleports A to the open start point of the save and waits for the
puppet to settle before measuring or pausing. 0.6.1: real 10/10, mutation all legs FAIL.

Stuck puppets (0.6.3): > 3 m from the remote player and < 0.3 m progress in 2 s → jump
beside the player; the jump script also calls `SetCommandable(TRUE, OBJECT_SELF)` (#162).
`blockedtest.py` makes B's puppet non-commandable (a synthetic block: walk orders refused):
```
0.6.1 (no rule)                              0/2  gap 5.9-6.9 m after 9 s
0.6.2 (rule, jump without SetCommandable)    0/2  stuckJumps 4 and 8, the jump was refused too
0.6.3                                        2/2  one jump each, ~2 s, gap 1.0 m
crossnet.py regression on 0.6.3              10/10
```
Limit: the block is synthetic. `stucktest.py` tried the points where the puppet got stuck
in the failed run, but from the open start they are reachable even on 0.6.1 (3/3), so a
physical block has not been reproduced on demand yet.

### v0.7.0: Steam (Spacewar, AppID 480) transport, tested with a mock
Download (approved by the owner): Steamworks.NET Standalone 2025.164.1, 32-bit
`steam_api.dll` from the release zip, SHA-256
`fdafb7c65ec7d4182b42bf92cc8671c68e4dc547cc6e613899d2618a47eb65da`
([releases](https://github.com/rlabrecque/Steamworks.NET/releases)). Not committed anywhere;
the player puts it next to the exe of the test copy.

Design (`dll/steamnet.inc` in [indy2000/jade-empire-coop](https://github.com/indy2000/jade-empire-coop),
commit `73a883c`), following [networking-steam-spacewar.md](../../playbook/networking-steam-spacewar.md):
- Flat C API through `LoadLibrary`/`GetProcAddress`, manual callback dispatch, all on the
  game's main thread, started on the first tick after `je.netStart`.
- Host: `CreateLobby(friends only, 2)`, lobby data `jecoop=1`, rich presence `connect`,
  lobby id written to `bridge\lobby.txt`. Joiner: `JoinLobby(<id from net.txt>)`.
- Callbacks used: 703 (call result), 513 LobbyCreated, 504 LobbyEnter, 1251 session request.
  The host accepts only lobby members, the joiner only the lobby owner.
- Same 24-byte packet as UDP; `SendMessageToUser` unreliable + no Nagle, channel 0.
- Sender identity read at offset 12 of `SteamNetworkingMessage_t` on 32-bit, fallback 16;
  the DLL counts which one matched (`senderAt12` / `senderAt16` in `je.netStatus`).
- 64-bit id ↔ decimal without the CRT helpers (`__aulldiv`/`__allmul`).

Mock (`mock/steam_api_mock.c`): a fake `steam_api.dll` exporting the same flat functions,
two copies on one PC talk UDP on 127.0.0.1; config `steam_mock.txt`. It raises 1251 once per
stranger until accepted, like real Steam. **It only proves our side of the API contract;
the offsets and callback layouts are what we believe real Steam uses — unverified.**

Results on the owner's PC, two test copies, same save, mock build `2e59229e…`, DLL `9554a6ee…`:
```
status: both in lobby 109775240917155841 (state 2), host accepted 1 rejected 0,
        senderAt12 46/42, senderAt16 0
crossnet.py (steam)              SUMMARY: 13/13 passed   puppet moved 5.8-5.9 m, gap 0.9-1.2 m, ~2.6 s
crossnet.py --mutate-pause (B)   SUMMARY: 9/13 passed    all 4 movement legs FAIL (moved 0.0 m)
```
Seen: the host counts `sendErr 6` from sends before the session was accepted. 0.7.0 as
committed (`8203645…`) counts a send as an error only when the send call itself failed.

Open: smoke test with real Steam on one copy (does `SteamAPI_Init` work for AppID 480 in this
game, is a lobby created); real two-account test between two PCs (confirms offset 12);
join through the friends list; facing/animations; combat; story; area transitions.

Smoke test with real Steam (one copy, owner's account, DLL `8203645…`, real `steam_api.dll`
`fdafb7c6…`, `steam_appid.txt` = 480, game started normally through its own launcher):
```
je.netStart → within 3 s: steam on, own SteamID read, lobby created (lobbyState 2),
lobby id written to bridge\lobby.txt; no peer yet → sent 0, sendErr 0 for 24 s; game kept running
```
So `SteamAPI_InitFlat`, the callback dispatch and `CreateLobby` work inside Jade Empire next
to its legacy `steam.dll` check. Still unverified with real Steam: join, session accept,
messages and the sender offset (needs a second account on another PC).

### v0.7.1: Ctrl+Shift+N arming and a player package
Players without the bridge client arm net with Ctrl+Shift+N (game window focused); one beep.
Package `JadeEmpireCoop-0.7.1.zip` (scripts in `package/` of the mod repo): `instalar.bat` finds
the game through the Steam registry key and `libraryfolders.vdf`, checks the exe hash, robocopies
the install to `%USERPROFILE%\JadeEmpireCoop\jogo` (original untouched), swaps in the proxy DLL,
copies `Steam.dll` and the pinned `steam_api.dll`, writes `steam_appid.txt`; `hospedar.bat` /
`entrar.bat` (asks for the lobby code) / `mostrar-codigo.bat` / `desinstalar.bat`.
Owner ran install + host on his PC: hashes in the copy match, log shows
`armed by Ctrl+Shift+N` → `steam up` → `lobby created`, and `mostrar-codigo.bat` showed the code.
Next: friend joins from another PC (first real two-account test).
