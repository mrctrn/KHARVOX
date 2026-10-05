# KHARVOX v1.11 on DOOM (2016) — GOG Edition: Diagnosis, Fixes & Status

**Date:** 2026-10-04 (last updated **2026-10-05**)
**Author:** Diagnostics performed with opencode assistance
**Game:** DOOM (2016), GOG Galaxy build (`DOOMx64vk.exe`)
**Mod:** KHARVOX v1.11 (github.com/CactusVRStudios/KHARVOX, source commit `ff348bbd62204067fe6e16abc99be2526cd08dc0`)
**Headset:** Meta Quest 3 over **Steam Link** (SteamVR 2.17.10, `vrlink` driver)
**GPU:** AMD Radeon 8060S (integrated, Strix Point) — driver 32.0.31036.15

---

## 1. Executive Status

| # | Problem | Status |
|---|---------|--------|
| 1 | Intro failed: `Create intro OpenXR instance (-32)` — ReShade implicit OpenXR layer | ✅ **Fixed** (ReShade layer disabled in registry) |
| 2 | DOOM crashed instantly: `vkCreateInstance failed (VK_ERROR_INITIALIZATION_FAILED)` / exit `0xC0000409` — KHARVOX hard-codes Steam-build byte signatures, GOG build fails them | ✅ **Fixed** (binary patch of `KharvoxLayer.dll`, verified working) |
| 3 | Game reaches VR but hangs with black window + AMD bug report — **GPU hang (LiveKernelEvent 141)** + unstable Steam Link stream | ⚠️ **Open** — resolution/load reduction in progress |
| 4 | Headset image always black while Steam dashboard/desktop render fine (audio plays) — `copy submit failed -13` (`VK_ERROR_UNKNOWN`): the XR copy command buffer (graphics queue family 0) is submitted to the *present* queue, and in live-stream runs DOOM switches its present queue to the **compute family (1)** from present call ~22 → every frame ends with 0 XR layers | ✅ **Fixed** — source fix deployed; verified in 19:42–19:45 session (zero `copy submit failed`, `Frame 120 stable`) (§8) |
| 5 | Steam Link stream drops (`Connection inactive` at 16:55:24 and 18:51:53) → OpenXR session stuck `SYNCHRONIZED`, `shouldRender=0` → black (different cause than #4) | ⚠️ **Open** — environment/network side (§7) |
| 6 | Game runs in VR but shows a **flat quad window** (no immersive stereo) and the **pistol cannot aim or fire** — `installPlayerPhysicsOriginCapture` Steam signature (RVA `0xE3F8D3`) not found on GOG build → gameplay anchor never valid → `KharvoxCameraGameplayActive()` false → presentation stuck QUAD + weapon gated off | ✅ **Fixed** — Fix #4 scan-based install verified in 20:34 session: capture at `0xe3e213`, auto-switch to PROJECTION reached (§9) |
| 7 | Immediately after switching to **PROJECTION** the headset goes **black** (audio/input alive; menu/QUAD phases visible) — AER source qualification requires the Steam-fingerprinted world-producer observer; `installEngine` fails the GOG PE gate → `aerSourceObservation` never recorded → `OpenXRBootstrap.cpp:5549` forces `layerCount=0` every projection frame | ✅ **Fixed & verified 2026-10-04 22:14 session** — all 25 projection cycles `layers=1`, `[STEAMXR-AER-PAIR] complete pair published` flowing, no `[AER-SOURCE-PAIR] r262 ACTIVE`, campaign playable (§10) |
| 8 | **Weapon viewmodel does not follow the VR controller** (6DOF grip works, model stays screen-anchored) — 24 Steam-signature checks fail on the GOG build, incl. `axis-copy 0x3B5400` (final transform), `useMuzzleAsFireAxis 0xD5EFF3` (shot direction), `active weapon-data 0xF137B0` (two-hand identity), hands scanners `0xD7D5D0`/`0xF275A0`, hands resources `0x16E18D0`, collision-pushback `0x6305B4`, hit-reaction `0xD88998` | 🔄 **Fix #6 implemented 2026-10-05 & deployed (`70a64ec3.`)** — transform + muzzle branch now scan-based (GOG `0x3b43e0` / `0xd5dc83`), pending in-headset verification (§14); remaining 19 mismatches deferred |
| 8b | After Fix #6: **ghost flat arms** + pistol in the ghost hands, and the **left-stick pause window opens outside the headset** — `active weapon-data 0xF137B0` signature is ambiguous on GOG (2 hits) so two-hand identity stays `unknown`; Pause Show/Hide 20-byte signatures are ambiguous so `Pause QUAD disabled` | 🔄 **Fix #7 implemented 2026-10-05 & deployed (`657ff02c.`)** — identity entry GOG `0xF12190` with caller gate `0xD7C352`, Pause Show/Hide GOG `0x1068ba0`/`0x10670a0` (§15), pending in-headset verification |
| 8c | Moving the tracked arms also moves the game **legs** | ⚠️ **Open** — not yet diagnosed (§15) |
| 9 | Image softness at `supersampleScale 0.8` — resolution reduced as a mitigation after **LiveKernelEvent 141** GPU hang (§1.3/§7) | ✅ **Re-tested at 1.0 on 2026-10-05 — keep 1.0 (§13)** — GPU frame time only 5.47→6.26 ms, dropped frames 376→10, +25% render pixels |

**Current state:** the mod boots DOOM into VR, the `copy submit failed -13` black is fixed (§8), Fix #4 (§9) delivered stereo PROJECTION, and Fix #5 (§10) is verified — immersive gameplay works end-to-end (22:14 session, foreground held, clean `exit code=0`). The 2026-10-05 session (§13) confirmed `supersampleScale 1.0` is the right setting: 15422 presents / 10 dropped (0.065%) vs 5544 / 376 at 0.8, GPU 6.26 ms against an 11.1 ms 90 Hz budget, no reprojection. Remaining polish items: (a) **game-side frame rate ~55 FPS** — the game itself submits below 90 Hz (11577 presents / 208.8 s), which caps smoothness more than SS does; (b) **weapon viewmodel does not follow the VR controller** — Fix #6 implemented and deployed 2026-10-05, awaiting in-headset verification (§14); (c) controller button context remapping (GAMEPLAY/UI vs `[BACK-WEAPON]`) — working as designed, needs familiarization; (d) Steam Link drops (§7); (e) Meta Horizon Link "never full immersive" (deferred, different runtime).

---

## 2. Environment Facts

- Game: `C:\Program Files\GOG Galaxy\Games\DOOM (2016)\DOOMx64vk.exe`
  - Size 101,520,384 bytes; PE `SizeOfImage = 0x1415F000`
- Mod install: `C:\Games\KHARVOX-v1.11\` (desktop shortcut; zip in `Downloads\KHARVOX-v1.11.zip`)
- Source copy: `C:\Users\mtrani\AppData\Local\Temp\kharvox_src\KHARVOX-ff348bbd…\` (also a copy at `D:\Documents\mod-project`)
- Active OpenXR runtime: `HKLM\SOFTWARE\Khronos\OpenXR\1\ActiveRuntime` → `SteamVR\steamxr_win64.json`
- Logs: `%TEMP%\KHARVOX.log`, `KHARVOX-VR-INTRO.log`, `KHARVOX-launcher-errors.log`, `KHARVOX-NATIVE-STEREO.log`, `KHARVOX-previous-launch.log`
- DOOM's own error logs: `C:\Users\mtrani\Saved Games\id Software\DOOM\base\ErrorLog_*.txt`
- SteamVR logs: `C:\Program Files (x86)\Steam\logs\vrserver.txt`
- Build toolchain present: CMake (`C:\Program Files\CMake`), VS 18 Community (`...\VC\Auxiliary\Build\vcvars64.bat`), dotnet SDK 10.0.401, git, `llvm-objdump` (`C:\Program Files\AMD\ROCm\6.4\bin\`)

---

## 3. Fix #1 — ReShade OpenXR Layer (Intro Crash)

**Symptom:** `KHARVOX-VR-INTRO.log` → `ERROR: Create intro OpenXR instance (-32)`; launcher: *"VR GameIntro ended before dismissal. DOOM was not started."*

**Cause:** `-32` = `XR_ERROR_FILE_ACCESS_ERROR`. ReShade registers an **implicit** OpenXR API layer:

```
HKLM\SOFTWARE\Khronos\OpenXR\1\ApiLayers\Implicit
  "C:\ProgramData\ReShade\ReShade64_XR.json" = DWORD 0   ← 0 = layer LOADS
```

The intro executable loads it, instance creation fails. KHARVOX 1.11 release notes mention disabling ReShade "for the launched game" — but **not** for `KharvoxIntro.exe`.

**Fix applied (permanent, elevated PowerShell):**

```powershell
Set-ItemProperty -Path "HKLM:\SOFTWARE\Khronos\OpenXR\1\ApiLayers\Implicit" `
  -Name "C:\ProgramData\ReShade\ReShade64_XR.json" -Value 1 -Type DWord
```

(32-bit view `WOW6432Node\…` for `ReShade32_XR.json` was already `1`.)

**Revert:**

```powershell
Set-ItemProperty -Path "HKLM:\SOFTWARE\Khronos\OpenXR\1\ApiLayers\Implicit" `
  -Name "C:\ProgramData\ReShade\ReShade64_XR.json" -Value 0 -Type DWord
```

**Result:** intro runs and tears down cleanly (`Completion: cleanup finished; handoff=ready`).

---

## 4. Fix #2 — GOG vs Steam Build Mismatch (DOOM Crash)

**Symptom:** DOOM exited with `0xC0000409`; `ErrorLog_*.txt` ended with:

```
FATAL ERROR: vkCreateInstance failed with error (VK_ERROR_INITIALIZATION_FAILED)
```

and in KHARVOX's extended log the last line before the crash:

```
[INDEPENDENT-SURFACE] supported engine size accessors could not be installed
```

**Root cause (confirmed from source, `src/vulkan/KharvoxLayer.cpp:950`):**

```cpp
if(!kharvox::installIndependentEngineSize(...)){
    logLine("[INDEPENDENT-SURFACE] supported engine size accessors could not be installed");
    return VK_ERROR_INITIALIZATION_FAILED;      // ← fatal
}
```

`installIndependentEngineSize()` (`src/vulkan/IndependentEngineSize.h`) verifies **exact byte signatures** hard-coded for the **Steam build** of `DOOMx64vk.exe`:

- `base+0x1871480` must equal `8B 05 AA 9F FD 01 C3 CC` (engine width getter)
- `base+0x1871250` must equal `8B 05 DE A1 FD 01 C3 CC` (engine height getter)
- plus engine-memory-guard signature at `base+0x1a32da0` (84 bytes)

Verification of the GOG executable:

| Check | Result |
|---|---|
| `SizeOfImage >= 0x384b438` | ✅ passes (`0x1415F000`) |
| width/height getter signatures | ❌ **not present** (full-file scan: 0 matches) |

The official README states it directly:

> **"Works only with a legal Steam version of Doom 2016"** — README.md:26

The GitHub repository currently has **zero issues** (no GOG reports exist).

**Why an env-var workaround is impossible:** the only variable that skips the fatal check (`KHARVOX_VULKAN_SFS=1`) is stripped by the launcher (`VulkanSfs.ClearEnvironment()`), and if forced manually it blocks XR session start (`OpenXRBootstrap.cpp:3496`).

---

## 5. Fix #2 Implementation — Binary Patch of `KharvoxLayer.dll`

**Decision:** patch the DLL rather than rebuild (source build remains a clean alternative, see §12).

**Patch site:** `C:\Games\KHARVOX-v1.11\KharvoxLayer.dll` (8,969,728 bytes, image base `0x180000000`)

`vkCreateInstance` hook, file offset **`0x27C64`** (VA `0x180028864`):

| | bytes |
|---|---|
| before | `0F 85 DE 00 00 00` (`jne` → success path) |
| after | `E9 DF 00 00 00 90` (`jmp` → success path, `nop`) |

Effect: the return value of `installIndependentEngineSize()` is ignored — the layer logs *"engine render size accessors armed"* and continues with **core-sized-window presentation** instead of failing instance creation. All other (non-fatal) signature checks behave as before.

**Backups & companion changes:**

- `C:\Games\KHARVOX-v1.11\KharvoxLayer.dll.orig` ← original DLL
- `C:\Games\KHARVOX-v1.11\native_sfs_build.txt.orig` ← original manifest
- `native_sfs_build.txt` line 2 updated to the patched DLL's SHA-256 (the launcher verifies it for SFS mode):

```
KHARVOX_NATIVE_SFS_1
5ec410fedb37e1dfdf1d74a354ccc6e097764085f217e9c04043e5e48fcb37b8
```

**Verification (llvm-objdump):**

```
180028862:  testb  %al, %al
180028864:  jmp    0x180028948        ; success path (was: jne)
180028869:  nop
```

**Rollback:**

```powershell
Copy-Item C:\Games\KHARVOX-v1.11\KharvoxLayer.dll.orig        C:\Games\KHARVOX-v1.11\KharvoxLayer.dll -Force
Copy-Item C:\Games\KHARVOX-v1.11\native_sfs_build.txt.orig    C:\Games\KHARVOX-v1.11\native_sfs_build.txt -Force
```

---

## 6. Verification of Fix #2

Manual test with the launcher's exact argument list + SFS environment (`KHARVOX_ENABLE_LAYER=1`, `KHARVOX_SFS_NATIVE_PROBE/VATIVE_VR/SOURCE_RING=1`, `KHARVOX_SFS_PROFILE`), 60 s run:

```
[WSI-STARTUP] engine render size accessors armed; presentation=core-sized-window
[INDEPENDENT-SURFACE] headset recommendation=2220x2376 source=2220x2376 … real window matches source
xrCreateInstance → xrCreateSession XR_SUCCESS → Session created → xrBeginSession XR_SUCCESS
[STEAM-XR-SPLIT] consumed=240 wait/begin/end=240/240/239   ← 240 stable XR frames
```

No crash, no new `ErrorLog`, registry/marker files restored to original state.

Through the real launcher (two runs):

- **14:06 run:** full init → `Frame 1 submitted mode=QUAD result=XR_SUCCESS` → GPU hang 4 s later
- **14:09 run:** init complete → `stereo images 2220x2376 created` → app hang at 12 s (black window)

⇒ The compatibility problem is solved; the new failure is GPU/stream level.

---

## 7. Open Issue — GPU Hang + Steam Link Instability

**Evidence:**

| Signal | Meaning |
|---|---|
| `LiveKernelEvent 141` at 14:06:58 and 14:09:17 (WER) | **GPU hang / TDR** — this is the "AMD Bug Report" popup |
| `AppHangB1`, Event 1002: `DOOMx64vk.exe stopped interacting` | game froze *because* the GPU hung (black window, exit `0x00000001`) |
| vrserver.txt: `HandleUnrecoverableError` ×28, `Null VTE_CLIENT_SESSION_ID` ×**2278**, video-packet timeouts ×28 | Steam Link stream is chronically unstable |
| Run 1: AMF encoder (`videoamfenc`) force-resetting the stream 14:06:55–57, GPU hang at :58 | encode glitch immediately precedes the hang |
| `steamtours.exe` (SteamVR Home) crash-looping (`0xc0000005` in `tier0.dll` at 13:57, 14:08, 14:10) | SteamVR-side instability |
| No `Event 4101` in System log, no dump in `C:\Windows\LiveKernelReports` | hang captured as LiveKernel WER report only |

**Load context:** the AMD 8060S is an **iGPU** — it renders DOOM VR (2220×2376 per eye at 100 % source size) **and** simultaneously encodes the Steam Link stream with AMF. This overload is the prime suspect.

### Resolution / Render Scale under SteamVR

The launcher **deliberately disables** the Render Scale control when SteamVR is the runtime:

```csharp
// launcher/KharvoxLauncher/…/KharvoxRunner.cs:1604
EffectiveRenderScale(selected, steamVrRuntime) => steamVrRuntime ? 100m : Math.Max(50m, selected);
```

> *"With SteamVR, adjust resolution within SteamVR; the launcher RenderScale control is disabled."* — InfoForm.cs:921

Consequences:

- The launcher always writes `render_scale.cfg = 1` and `KHARVOX_RENDER_SCALE=1` under SteamVR (100 %).
- The FSR checkbox is inert under SteamVR: `fsr1Enabled = UseFsrUpscaling && effectiveRenderScale < 100` → never true.
- **The lever under SteamVR is SteamVR's own Render Resolution**, because KHARVOX derives its source extent from `xrGetViewConfigurationViews → recommendedImageRectWidth/Height`, which SteamVR scales with that setting.
- Note: `render_scale.cfg` content is a plain ratio (`Inv(x)` in the launcher is invariant-culture `ToString`, *not* a mathematical inverse): `1` = 100 %, `0.8` = 80 %.

### Action Plan (ordered)

1. **SteamVR → ☰ → Video → Render Resolution → ~80 %** (target ≈ 1700–1800 px per-eye width instead of 2220), apply, restart SteamVR, relaunch.
   - Per-app alternative: override video settings for `system.generated.openxr.kharvox.doomx64vk.exe` while it runs.
2. If still hanging → switch renderer **SFS → AER** (README's official fallback: *"SFS is the default… If SFS causes problems, select AER as a fallback."*).
3. If still hanging → **change the streaming path**: the KHARVOX README recommends **Meta Quest Link or Virtual Desktop** for Quest headsets (Steam Link is not the recommended path; its 2278 stream errors are a major red flag). Also lower the stream resolution/codec in the Quest's Steam Link app.
4. Disable **SteamVR Home** (`steamtours`) to stop its crash loop.

---

## 8. Fix #3 — Black Headset Image: `copy submit failed -13` (Present-Queue Family Mismatch)

**Symptom:** SteamVR dashboard and the desktop view render fine in the headset, but the game image is always black while game audio plays. Reproduced on every run where the Steam Link stream was live (`shouldRender=1`); runs with a dead stream are black for a different reason (`shouldRender=0`, §7).

**Log fingerprint:**

- `[KHARVOX][XR] copy submit failed -13` repeating every frame (3971× in the 18:49 run), followed by `ORDER` reason `copy-submit-failed layers=0` → `xrEndFrame` submitted with **zero layers** → compositor shows nothing → black.
- `-13` = `VK_ERROR_UNKNOWN` (return of `vkQueueSubmit`). The launcher only classifies `-4` as fatal (`launcher/KharvoxLauncher/KharvoxRunner.cs:1393`), so this failure is silent in the launcher UI.

**Root cause (confirmed by read-only code analysis):**

| # | Fact | Location |
|---|------|----------|
| 1 | The XR copy command buffer is allocated from a pool created for **`s.queueFamily` = graphics family 0** | `src/openxr/OpenXRBootstrap.cpp:3217` |
| 2 | Queue selection adopts only the graphics queue (`flags & VK_QUEUE_GRAPHICS_BIT`); the compute queue (family 1, flags=14) is rejected by design | `OpenXRBootstrap.cpp:3469-3491`, `RuntimeVulkanDispatch.h:8-12` |
| 3 | The owner copy is submitted to **`q` — the queue DOOM passed to `vkQueuePresentKHR`** — not to `s.queue` | `OpenXRBootstrap.cpp:5220` |
| 4 | On submit failure: release acquired images, `log("copy submit failed …")`, `endEmptyFrame("copy-submit-failed")`, return — every subsequent frame repeats this path | `OpenXRBootstrap.cpp:5246-5252` |
| 5 | **DOOM switches its present queue mid-run:** dead-stream/healthy runs present 240/240 sampled calls (calls 1–240) on the graphics queue with `waits=1`; live-stream failing runs switch at present **call 22** to the **compute queue (family 1)** with `waits=0` (224/246 sampled entries) — the first `-13` occurs exactly at that switch | `[PRESENT] entry` lines in `%TEMP%\KHARVOX*.log` |
| 6 | A family-mismatched `vkQueueSubmit` (family-0 command buffer on a family-1 queue) is invalid usage; the AMD driver returns `VK_ERROR_UNKNOWN` | Vulkan spec |

Who initiates the family-1 presents is not yet identified (`[PRESENT] entry` lines carry `thread=` IDs; the next repro will show it). The fix does not depend on it.

**Evidence-handling caveat:** `KHARVOX-previous-launch.log` is overwritten on every launch — the 3971-line evidence log was lost this way. Archive `%TEMP%\KHARVOX*.log` before relaunching during any repro.

**Approved fix (2026-10-04):**

1. In `KharvoxXRPresent`: compute `queueMismatch = (q != s.queue)`; submit the copy on `copyQueue = mismatch ? s.queue : q` (`:5220`), and `queueWaitIdle(copyQueue)` in the no-fence fallback (`:5228`).
2. When mismatched, force the copy-completion wait before returning (fold `!queueMismatch` into the `earlyRelease` condition at `:5202`, before `copyLifetime` consumes it at `:5204`) so the downstream present on `q` can never race the copy's image-layout transitions.
3. Add a rate-limited diagnostic: `[PRESENT] copy queue redirect present=… xr=…` (identifies the foreign presenter via its `thread=`).
4. The normal path (`q == s.queue`, i.e. every healthy observed run) stays bit-for-bit unchanged.
5. The source build additionally makes `installIndependentEngineSize()` failure non-fatal (`src/vulkan/KharvoxLayer.cpp:950`) — the source equivalent of the §5 byte patch (required, otherwise the fresh DLL aborts instance creation on the GOG build).
6. Deploy: back up, replace `C:\Games\KHARVOX-v1.11\KharvoxLayer.dll`, **update line 2 of `native_sfs_build.txt` with the new SHA-256** (the launcher verifies it for SFS mode), keep `.orig` backups. Rollback = `KharvoxLayer.dll.orig` + `native_sfs_build.txt.orig`.
7. Verify with one live Steam Link session: no `copy submit failed` in `%TEMP%\KHARVOX.log`, `Frame 120 stable` reached, redirect log explains the queue switch, game visible in the headset.

**Risk:** the source snapshot (`ff348bbd`) may differ slightly from the released v1.11 DLL (§12 covers building it).

**Follow-ups (not in this fix):** Steam Link connection drops (§7, environment/network); Meta Horizon Link never reaching full-immersive mode (separate OpenXR runtime path); optional `-13` addition to the launcher's fatal classification.

---

## 9. Fix #4 — Flat Quad Window + Frozen Weapon (GOG Hook Signature Drift)

**Symptom (2026-10-04 evening session):** game runs in VR — controllers active, character moves forward/back — but the headset shows a large flat game window instead of immersive stereo, and the pistol cannot be aimed or fired. (The user's SteamVR render-scale reduction to 80% was **not** a factor.)

**Log fingerprint:**

- `Presentation mode: QUAD` forever; never `Presentation automatic switch: PROJECTION`; `[STEAM-XR-ORDER] … reason=quad-layer`; `[QUAD-POSE] … meters=2.0x1.12 … z=-6.76`.
- `player physics-origin capture signature mismatch; stable weapon anchor disabled` (`CameraHook.cpp`).
- `[BACK-WEAPON] … gameplay=no`.

**Root cause (single chain):**

1. `installPlayerPhysicsOriginCapture` matched a 15-byte code signature at hardcoded Steam RVA `0xE3F8D3`. On the GOG build the same code sits at `0xE3E213`; two unrelated functions repeat the 15 bytes (0x4DBE11, 0xD09C39) — install failed.
2. Consequence A (presentation): the player physics anchor never became valid → `KharvoxCameraGameplayActive()` always false → `gameplayPresentationInterrupted` (`OpenXRBootstrap.cpp`) keeps `desiredQuad` → stuck in QUAD = flat window.
3. Consequence B (weapon): 6DOF weapon tracking and the fire path are gated on `KharvoxCameraGameplayActive()` (`WeaponHook.cpp` sites) → pistol frozen, trigger dead. Character movement is XInput-based and ungated, so it still worked.

**GOG vs Steam RVAs (render-camera cluster, drift Δ ≈ 0x16C0):**

| Hook constant (Steam) | RVA Steam | RVA GOG | How resolved |
|---|---|---|---|
| Capture call site | 0xE3F8D3 | **0xE3E213** | 15-byte sig + context: ViewAxis call `FF 90 68 03 00 00` at +0x23 AND `E8` at +0x92 |
| ViewAxis double-apply guard | 0xE3F8FC | 0xE3E23C | derived = site +0x29 |
| `gameplayCamera` caller | 0xE3F96A | 0xE3E2AA | derived = site +0x97; independently confirmed (call at 0xE3E2A5 targets camera-update at 0xE6E970) |
| Animated/cinematic caller | 0xA30CFF | 0xA2F73F *(candidate)* | only caller of camera-update 0xE6E970 in the 0xA2F/0xA30 region (Δ0x15C0) |
| idPlayer physics offset / GetOrigin vtable slot | 0x14E58 / 0x80 | **identical** | `lea reg,[reg+0x14E58]; mov rax,[reg]; call [rax+80h]` found at 0x3A50F6, 0x93C843, 0x97ABD7, 0xB11D64, 0xD05961 |
| camera-update function | — | 0xE6E970 (tail 0xE6E9D3 = DOOMPatchCamera scan target) | scan-verified |

**Implementation:**

1. `installPlayerPhysicsOriginCapture` now scans `.text` for the signature **plus structural context** (ViewAxis call at +0x23, `E8` at +0x92), with fallback to exact Steam RVA `0xE3F8D3` (Steam behavior preserved).
2. On success it stores derived guard RVAs into atomics (`renderCameraAxisGuardRva`, `renderCameraGameplayCallerRva`); a shifted site also swaps `animatedCameraCallerRva` to 0xA2F73F.
3. Call sites now read the atomics instead of literals: `CameraHook.cpp:778` (axis guard), `:1541`/`:1554` (cinematic), `:1670` (gameplayCamera).
4. The install log prints the resolved RVA and derived values.
5. HUD caller guards (`HudHook.cpp`, `HudMenuPolicy.h`) still use Steam RVAs — secondary, unchanged.

**Deployed:** `KharvoxLayer.dll` SHA-256 `9dbfaa5b395cbc4fcd678a5b1f40b8fea5e520d2ee0836af38fc6ffd8b5913f7` (source build with fixes #3 + #4). Previous 3-fix build saved as `KharvoxLayer.dll.sfsfix1-backup` (SHA `18b1c1ab…`); pristine `.orig` pair and patched-release `.v111-gogpatched-backup` untouched. `native_sfs_build.txt` line 2 updated.

**Verification checklist (next Steam Link session):**

- `stable idPhysics_Player origin capture installed at gameplay RVA 0xe3e213 (axis guard 0xe3e23c, gameplay caller 0xe3e2aa, animated caller 0xa2f73f)`
- `[BACK-WEAPON] … gameplay=yes`
- `Presentation automatic switch: PROJECTION` and `Frame … mode=PROJECTION`
- Pistol tracks in 6DOF and fires; glory kills show `cinematic entry` / `[IMMERSIVE] cinematic exit` logs.

**Residual risk:** animated-camera caller 0xA2F73F is a region-based mapping, not structurally proven — if wrong, cinematic/glory-kill classification misbehaves. ~15 other Steam-signature scans (focus/ledge/sync/ammo/muzzle/HUD) still fail on GOG; their features stay disabled with per-feature mismatch warnings (§11).

---

## 10. Fix #5 — Black Screen in PROJECTION Mode (Missing AER Source Observer on GOG)

**Symptom (2026-10-04, 20:34 session):** Fix #4 worked — `stable idPhysics_Player origin capture installed at gameplay RVA 0xe3e213 …`, presentation auto-switched QUAD→PROJECTION with stereo active — but roughly one second into the campaign the headset went **black** (audio and input kept working). Menu and cinematic/QUAD phases stayed visible. Pattern: **QUAD → visible, PROJECTION → black.**

**Log fingerprint:**

- `[STEAM-XR-ORDER] … reason=projection-layer layers=0` sustained through every PROJECTION phase (cycle 840 @20:34:59 → layers=0, again 20:35:48 → 20:36:02), while QUAD phases show `layers=1` (cycle 720 @20:34:58, cinematic 20:35:13–47). `xrEndFrame` returned XR_SUCCESS, `shouldRender=1` — but with **zero composition layers** submitted.
- `[AER-SOURCE-PAIR] … qualified=0 ambiguous=0 sourceEye=-1 sourcePose=0 domain=0 … publish=0` repeating every cycle — the AER source observation is never recorded.
- `KHARVOX-NATIVE-STEREO.log`: 10× `[RENDER-CVAR] AER setter profile/installation failed; visual settings preservation unavailable` — **not cosmetic** (correcting §11): it marks `installEngine` failing at the PE gate, i.e. the *same* failure that leaves the world-producer observer uninstalled.
- `world camera INACTIVE after 90 present frames` only appears during level loading — normal, not causal.

**Root-cause chain:**

1. `OpenXRBootstrap.cpp:5549`: `ei.layerCount = (aerSourceMode && !s.steamXrAerPairReady) || … ? 0 : layerCount` — deliberate guard: never submit uninitialized/mislabelled projection images while priming.
2. `aerSourceMode` (`:4371`, via `aerUseSourceQualification` in `AerSourceTracking.h:10-12`) turns on with the coherent pair path and `steamXrAerPairReady` is cleared on activation (`:4376`) — it re-arms only when `aerSourceQualified` (`:4381`) sees a valid observation from `KharvoxCameraTakeAerWorldSource()`.
3. The only producer of that observation is `KharvoxCameraRecordAerWorldSource` (`CameraHook.cpp:2062`), called solely from `NativeAerWorldProducer.inc:161` inside `hookAerBuildViewMatrices`, installed by `NativeStereo.cpp installEngine(visualOnly=true)`.
4. **`installEngine` returns false at its first gate (`NativeStereo.cpp:176`)** — Steam `TimeDateStamp=1711036533` / `SizeOfImage=336990208` vs GOG `1740486338` / `0x1415F000=336982016`. Both mismatch → early return before any hook install (later gates are also Steam-only: world-producer `0x1830b80`, model-matrix `0x1ade630`, freshShadow cvar setter `0x295780`, per-target FNV hashes).
5. No observer → no observation → pair never "ready" → `layerCount=0` every projection frame → black.

**Fix #5 (Option A — implemented):** disable source qualification when the producer observer is not installed, falling back to the pre-r262 coherent-pair projection path (cycle 720 proves that path presents `layers=1`):

1. `CameraHook.h/.cpp`: `void KharvoxCameraSetAerWorldSourceObserver(bool)` / `bool KharvoxCameraAerWorldSourceObserverActive()` backed by `static std::atomic<bool>` (default **false**).
2. `NativeStereo.cpp`: clears the flag on `installEngine` entry (covers the PE gate, every validator failure including the fingerprint-mismatch else-branch, and MinHook failures), and sets `true` only after a successful visualOnly install when `realAerBuildViewMatrices` is non-null (next to the `[AER-WORLD-SOURCE] r262 validated … ACTIVE` log).
3. `OpenXRBootstrap.cpp:4371`: `aerSourceMode = aerUseSourceQualification(…) && KharvoxCameraAerWorldSourceObserverActive()` — with the observer present, behavior is unchanged from Steam/r262; absent (GOG today), source mode stays off and the coherent-pair path presents layers normally.

**Build/test/deploy:** Release rebuilt from `ff348bbd` source; `ctest -C Release` 103/105 (the 2 failures — `hand-scene-depth-gpu`, `native-depth-seed-gpu` — are AMD GPU-compute tests unrelated to these edits, pre-existing on this machine). `KharvoxLayer.dll` SHA-256 **`d69fb39f84385a059ea9c924bae503d1914c95c10210e3ff48c58993de3bcb43`** deployed; previous Fix-#4 build saved as `KharvoxLayer.dll.fix5-backup` (SHA `9dbfaa5b…`); `native_sfs_build.txt` line 2 updated and verified equal to the deployed SHA.

**Deferred (Option B — full GOG parity):** locate Steam `0x1830b80` (world producer), `0x1ade630` (model matrix), `0x295780` (freshShadow cvar setter) in the GOG exe, relax the PE gate, recompute the FNV fingerprints — no Steam reference binary is available on this machine. Only pursue if Option A produces eye/turn-compensation artifacts.

**SFS renderer note:** switching the launcher renderer to SFS/native would **not** avoid this problem and is worse — the same PE gate blocks the full native hook profile (`EngineProfile.inc`), native mode needs many more Steam-RVA hooks (view-layout `0x36111E8`/`0x284FFC8`, …), and SFS consumes the *same* missing observation. AER is the viable path on GOG.

**Verification checklist (next session):** sustained `projection-layer layers=1` during gameplay projection; no endless `qualified=0 publish=0` (or no `[AER-SOURCE-PAIR] r262 ACTIVE` line at all); world visible in headset at `Presentation automatic switch: PROJECTION`; cinematic → centered-QUAD transitions still work; watch for eye/turn artifacts (fallback-path risk).

---

## 11. Other Learnings (Non-Causal but Documented)

- **Launcher layer registration:** HKCU implicit Vulkan layer `HKCU\SOFTWARE\Khronos\Vulkan\ImplicitLayers` → `C:\Games\KHARVOX-v1.11\KharvoxLayer.json` (DWORD `0` = loads, nonzero = disabled; the launcher manages this value).
- **Layer enable/disable env:** `KharvoxLayer.json` → `enable_environment: KHARVOX_ENABLE_LAYER=1`, `disable_environment: KHARVOX_DISABLE_LAYER=1`.
- **Temporary marker files** (created by launcher at launch, deleted at exit): `enable_xr_session`, `enable_xr_mediated`, `enable_native_xr_resolution`, `enable_6dof_weapon`, `enable_two_hand_weapon`, `enable_cached_eye_reprojection`, `render_scale.cfg` — the XR session will not start without `enable_xr_session`.
- **Non-fatal signature checks:** CameraHook focus/ledge/cinematic mismatches only log a warning (e.g. `[FOCUS] … trace signature mismatch; HMD USE ray disabled`).
- Other Vulkan implicit layers present (EOS, GOG Galaxy, Steam overlays, fossilize) were **tested and ruled out** as crash causes.
- `vulkaninfo` healthy: loader 1.4.341, AMD proprietary driver 26.6.4 (LLPC), API 1.4.349.
- Vanilla (unmodded) GOG DOOM runs fine with `VK_LOADER_DEBUG=all` — DOOM's own Vulkan path is healthy.
- `KHARVOX-NATIVE-STEREO.log` shows: `[RENDER-CVAR] AER setter profile/installation failed; visual settings preservation unavailable` — **corrected 2026-10-04:** not cosmetic; it marks `installEngine` failing the Steam PE gate on GOG, which is the root cause of the PROJECTION black screen (§10).
- README note: *"Changing graphics quality while a level is loaded can freeze or crash the game. Change quality in the main menu before loading a level."*
- KHARVOX is fully offline: no update checks, telemetry, or downloads.

---

## 12. Optional Next Step — Build Your Own Source Patch

A source build is fully possible on this machine (CMake + VS 18 Community C++ + dotnet SDK 10 + git) and is the clean, maintainable variant of the binary patch:

```cpp
// src/vulkan/KharvoxLayer.cpp:950  — make non-fatal
if(!kharvox::installIndependentEngineSize(...)){
    logLine("[INDEPENDENT-SURFACE] engine size accessors unavailable; using core-sized-window presentation");
    // do NOT return VK_ERROR_INITIALIZATION_FAILED
}
```

README build steps:

```powershell
git clone --recurse-submodules https://github.com/CactusVRStudios/KHARVOX.git
cmake -S . -B ..\kharvox-build\native -A x64 -DKHARVOX_BUILD_LAYER=ON -DBUILD_TESTING=ON
cmake --build ..\kharvox-build\native --config Release --parallel
dotnet restore launcher\KharvoxLauncher\KharvoxLauncher.csproj
dotnet build  launcher\KharvoxLauncher\KharvoxLauncher.csproj --configuration Release --no-restore
```

After building: replace `KharvoxLayer.dll`, update line 2 of `native_sfs_build.txt` with the new SHA-256.

**Scope caveat:** a source build (like the binary patch) restores *functionality*, not *Steam parity* — the engine-size hooks stay uninstalled because the signatures target the Steam executable. Full GOG parity would require reverse-engineering the GOG build to locate its own window-size getters (`IndependentEngineSize.h`).

---

## 13. Supersampling / Performance Verification (2026-10-05)

`supersampleScale` had been dropped to **0.8** as a mitigation after the **LiveKernelEvent 141** GPU hang (§1.3/§7). The 2026-10-05 session (18:45:54–18:49:28, `KHARVOX.log` 1,103,260 B, `exit code=0`, uptime 227.5 s, foreground held) re-tested at **1.0**.

### Result: keep `supersampleScale 1.0`

| Metric | SS 0.8 (2026-10-04) | SS 1.0 (2026-10-05) |
|---|---|---|
| GPU frame time (`vrcompositor.txt` ApplicationTime) | 5.47 ms | **6.26 ms** |
| CPU frame time | 11.93 ms | **10.44 ms** |
| Render resolution (WSI-RESIZE) | 3520x1980 | **3936x2214** (+25% pixels) |
| Presents / dropped / reprojected | 5544 / **376** (6.8%) / 0 | 15422 / **10** (0.065%) / 0 |
| Present rate | 64.3/s | 55.5/s |

- **90 Hz budget is 11.11 ms; 6.26 ms GPU = ~44 % headroom.** The +25 % pixel cost is affordable, and dropped frames fell ~98 % despite the higher resolution (the 0.8 run also included the unstable Steam Link stretch).
- Revert path if stutter is ever felt: set `"supersampleScale" : 0.8` in `C:\Program Files (x86)\Steam\config\steamvr.vrsettings` (`"supersampleManualOverride" : true` keeps Steam from re-deriving it).
- XR output confirmed at 90 Hz: `xrEndFrame` eye bounds **2064x2208 @ 90 Hz**; launcher render scale 100 %/100 %/100 %; `Raw ideal render target scale = 1.14`.

### Game-side frame rate is the real smoothness cap

`11577 presents / 208.8 s ≈ 55 FPS` — DOOM itself submits below the 90 Hz headset rate, so even a free SS increase cannot exceed what the game produces. Present rate 64.3/s → 55.5/s (−14 %) between runs, but that comparison mixes loading/menu phases and the unstable-link period; it is not a clean A/B. The GPU is not the limiter (6.26 ms), so the ceiling is game-side CPU/game logic or its own frame pacing.

### Pair-rate question resolved (not a defect)

The logs show ~47.7 XR submits/s where 90 would be expected. Resolution: **published pair (23.9/s) + reused pair (23.9/s) = 47.7/s ≈ the configured `fullPairHz 45`** — a strict 1:1 alternating-eye design (`OpenXRBootstrap.cpp:4438` `steamXrAerReusePublishedPair`, `:4899` `handPairWillPublish`, logging at `:5302-5329`). Each eye of the pair is submitted separately, so 45 pair-equivalents/s is the design rate, not a dropped rate. Session totals: `published=4096 reused=4096`.

### Present stalls (32 `STALL` lines)

| Class | Count | Detail |
|---|---|---|
| `present-wrapper-total` | 29 | min 20.0 / median 23.4 / max **777.6** / avg 50.0 ms |
| `between-present-calls` | 3 | 74.2 / 99.6 / 155.6 ms |

Only **777.6 ms @ 18:45:54.367** matters — it is the startup/first-frame init stall. The rest are 20–35 ms (threshold 33.4 ms: 35.1 @18:48:11, 33.65 @18:48:07, 33.34 @18:46:44). Present log lines interleave from 32 threads (120 negative deltas) — only `call=` deltas are trustworthy.

### Focus churn 18:47:41–18:48:51

Six `FOCUSED↔VISIBLE` flips = the user opening Steam dashboard/menus (foreground stayed `gameForeground=1`). Not a bug.

### Reading the numbers live

- SteamVR already graphs in-headset: `"perfGraphInHMD" : true`, `"showAdvancedGraphs" : true` (Developer settings).
- Game-side counter: launch arg `+com_showFPS "1"`.
- Offline extraction: `%TEMP%\KHARVOX.log` + `C:\Program Files (x86)\Steam\logs\vrcompositor.txt` (`Total … presents / dropped / reprojected`, `FPS Average Target 90`). Note `vrcompositor.txt` totals accumulate per session-start — the line stamped 18:49:28 is the 2026-10-05 run (session total dropped 41 includes startup/loading).

---

## 14. Fix #6 — Weapon Viewmodel Not Following the Controller

**Symptom:** 6DOF grip tracking works (`[WEAPON] native 6DOF ENABLED`), but the weapon/arms viewmodel stays screen-anchored instead of following the VR controller. **Status: implemented 2026-10-05, deployed, awaiting in-headset verification.** Non-fatal — the game remains playable; native arms render while the VR hands overlay is unaligned.

**Cause:** 24 `signature mismatch` lines in the 2026-10-05 log — every Steam-fingerprinted weapon/HUD hook declines to install on the GOG build. Breakdown: **10 `[WEAPON]`**, **11 `[HUD]`**, **3 `[CAMERA]`**.

Weapon hooks (the ones that matter for aiming):

| RVA | Hook | Effect when missing |
|---|---|---|
| `0x3B5400` | axis-copy — final weapon transform | **weapon model not parented to controller** |
| `0xD5EFF3` | `useMuzzleAsFireAxis` branch | VR shot direction disabled |
| `0xF137B0` | active weapon-data — two-hand identity | stays `UNKNOWN/einhändig` |
| `0xD7D5D0` | `idHands` update — joint scanner | hands joint scan disabled |
| `0xF275A0` | weapon-render — child-model probe | child-model probe disabled |
| `0x16E18D0` | `idHands` resource — restart null guard | restart null guard disabled |
| `0xD60760` | hands-FOV projection neutralizer | projection neutralizer disabled |
| `0x6305B4` | hands collision-pushback suppression | suppression disabled |
| `0xD88998` | hands hit-reaction gate suppression | suppression disabled |
| — | native inventory/ammo | Shoulder Weapon keeps timeout-only fallback |

HUD/camera hooks (cosmetic/UX, lower priority): `DOOMHUDCrosshair 0xC317BE`, `HUD5 origin-setter 0x3B7500`, `idMenuScreen_Shell_Pause`, `idMenuManager_PlayerUpgrade`, `idInteractable_WeaponModBot`, `idDesignSystems_UpgradeStation`, `HUD-MOVIE`, `OFFHAND-HUD`, `RUNE-START`, `TUTORIAL-BINDING`, `TUTORIAL-RENDER`, `FocusTracker trace` (HMD USE ray), `CINEMATIC-CLASS ledge` + `sync-state` (Glory Kill/Jump exceptions).

**Approach (same technique as Fix #4, §9):** the release DLL's byte signatures are Steam-exe-specific. Locate each GOG counterpart by following the code around the already-found anchors (e.g. the Fix #4 capture at `0xe3e213`) or by disassembling `DOOMx64vk.exe` (`llvm-objdump` is installed), then convert the fixed-offset reads to pattern scans. Start with `0x3B5400` (transform) + `0xD5EFF3` (fire axis) — those two alone should make the gun follow the controller and aim where it points.

**GOG counterparts (reverse-engineered and verified unique on the GOG exe, 2026-10-05):**

| Steam RVA | GOG RVA | Signature used | Uniqueness check |
|---|---|---|---|
| `0x3B5400` | `0x3B43E0` | 17-byte `0F B6 41 71 84 41 70 75 50 8B 02 89 81 D4 00 00 00` (identical on GOG, no wildcard) | exactly 1 hit in `.text`, at a function entry; body byte-identical |
| `0xD7E518` (weapon caller return) | `0xD7D1A8` | call-site context `48 8B CE` before + `0F B6 46 71` after (111 call sites to the target, this context = 1 hit) | unique |
| `0xD8AE66` (animated prop caller return) | `0xD89B16` | context `F3 41 0F 11 85 80 00 00 00` before + `8B 86 40 01 00 00` after | unique |
| `0xD5EFF3` (muzzle fire-axis branch) | `0xD5DC83` | 13-byte context `80 BF 21 06 00 00 00 0F 85 6A 06 00 00`, branch = match + 7 | unique; the bare `0F 85 6A 06 00 00` alone has 3 hits (0x634D7D, 0xD5DC83, 0xDDC7D8) so it is unusable by itself |
| `0xF9227F` (HUD final caller) | unresolved | — | deferred: the HUD gate is stored as `0` on shifted builds, so it never matches a foreign return address |

**Implementation (`src/weapon/WeaponHook.cpp`):**
1. File-scope atomics `weaponAxisCallerRva` / `weaponPropCallerRva` / `hudFinalCallerRva` default to the Steam RVAs (Steam behaviour unchanged).
2. Helpers `findUniqueExecutableBytes` (fail-closed: **unique match or failure, never first-match-wins**), `findAxisCopySite` (legacy `image+0x3B5400` first, else 17-byte scan), `findMuzzleBranchSite` (legacy first, else 13-byte context + 7), `resolveAxisCopyCallers` (walks `E8` call sites with `int32_t` displacement → target, matches the two structural contexts, stores the return RVAs; unresolved prop/HUD entries are stored as `0` so no other function's return address can satisfy the gate).
3. `setRenderEntityAxisHook` now compares `callerRva` against the atomics instead of hardcoded constants; `installMuzzleFireAxisOverride` and the `KharvoxWeaponInstallHook` install path use the scanners; any scan failure takes the existing restore path and returns `false` (hook disabled, native arms still render — non-fatal, fail-closed).
4. Success logs are dynamic: `final idHands origin/axis hook installed at RVA 0x3b43e0; weapon caller gated to RVA 0xd7d1a8; prop caller gated to RVA 0xd89b16` and `useMuzzleAsFireAxis branch armed at RVA 0xd5dc83`.
5. Note: `HudHook.cpp:2016`'s raw `progNative(0x3b5400)` call needs no change — it sits behind Steam-only gates (`installProgMeterHooks` → `progHooksReady`) that GOG never sets (log: `hooks unavailable; native Ammo grouping retained`).

**Build / deploy (2026-10-05):** Release rebuilt; `ctest -C Release` **103/105** (the 2 failures are the pre-existing AMD GPU tests `hand-scene-depth-gpu`, `native-depth-seed-gpu`). Deployed `KharvoxLayer.dll` SHA-256 **`70a64ec394981328b6a6106ddc986e3f6eba950c3836156f5d9a6de6c3943afc`**; previous Fix #5 build saved as `KharvoxLayer.dll.fix6-backup` (SHA `d69fb39f…`); `native_sfs_build.txt` line 2 updated and verified equal to the deployed SHA.

**Verification checklist (next session):**
- **no** `axis-copy signature mismatch`, **no** `axis-copy caller signature mismatch`, **no** `useMuzzleAsFireAxis branch signature mismatch`
- present: `final idHands origin/axis hook installed at RVA 0x3b43e0 …` and `useMuzzleAsFireAxis branch armed at RVA 0xd5dc83`
- in headset: weapon model follows the controller (6DOF), shots go where the muzzle points
- if a mismatch line appears instead: the hook failed closed (native arms render, gameplay unaffected) — re-run the scanners against the exe

**Still open (deferred, non-blocking):** the other 19 `signature mismatch` lines — active weapon-data `0xF137B0` (two-hand identity), hands update `0xD7D5D0`, render probe `0xF275A0`, hands resource `0x16E18D0`, FOV `0xD60760`, collision pushback `0x6305B4`, hit reaction `0xD88998`, inventory/ammo, plus 11 HUD and 3 CAMERA hooks. Same scan technique applies if aiming polish (two-hand identity, hit reactions) is wanted later.

---

## 15. Fix #7 - Ghost Arms, Pistol Not In Hand, Pause Window Outside Vision

**Symptoms reported after the Fix #6 play test (verbatim):**

> "So I tried again the initial part of the game. When I move now the hands in a direction it move also arms from the game. It is progress. Unfortnuately the flat arm are always visible like a ghost. It is like I moving ghost arms and the pistol is not in my hand, but in the ghost one. Another strange thing is that I move my arms, in the games it moves up the legs. Very strange. Another weard thing, when I click on the left thumbstick it appears a game window, but it is outside of my vision on left but I see only the boarder"

| Symptom | Root cause | Fix #7 change |
|---|---|---|
| Ghost flat arms; pistol sits in the ghost arms, not the tracked hands | `active weapon-data` entry (Steam `0xF137B0`) — its 15-byte signature has **2 hits** on GOG, so the hook never installed, `observeActiveWeaponData` never ran, `activeWeaponKind` stayed `Unknown` (log: `role=root weapon=unknown`, `stored=1.000000`) | Unique **42-byte** signature → GOG `0xF12190`; the idHands caller gate moved `0xD7D6C2` → GOG `0xD7C352`, now resolved at install time (0 = disabled) |
| Left thumbstick opens a menu window that sits outside the headset | Pause lifecycle 20-byte signatures are ambiguous on GOG → `idMenuScreen_Shell_Pause lifecycle signature mismatch; Pause QUAD disabled` → the native window opens with no VR quad | Unique **64/46-byte** signatures → GOG Show `0x1068BA0`, Hide `0x10670A0`; trampoline now replays only the 20 leading (RIP-independent) bytes |
| Moving tracked arms also moves the game legs | **not yet diagnosed** — candidates: caller context on the animated-prop path, entity `0x22d69d694d0` with `nativeLocal=0,0,0.714111`, or the `[IMMERSIVE] animation position + HMD-only rotation ACTIVE` path | deferred |

### Verified GOG signatures (exactly 1 hit in the executable sections)

| Hook | Bytes | GOG RVA | Steam RVA |
|---|---|---|---|
| Active weapon-data entry | 42 (`44 8B CA 83 FA FF 75 07 44 8B 89 D4 08 00 00` + tail `… 48 85 C0 75 16`) | `0xF12190` | `0xF137B0` |
| idHands caller gate | `8B 93 D4 08 00 00` before the `E8`, `48 85 C0` after it | `0xD7C352` | `0xD7D6C2` |
| Pause ShowScreen | 64 (prologue incl. cookie load `48 8B 05 ED AF 6D 04`) | `0x1068BA0` | `0x106A8D0` |
| Pause HideScreen | 46 (prologue incl. cookie load `48 8B 05 FD CA 6D 04`) | `0x10670A0` | `0x1068DD0` |

The Show/Hide spacing stays `0x1B00` on both builds. The caller gate was cross-checked twice: Steam `0xD7D6C2 - 0xD7E518 = -0xE56` applied to the already-resolved GOG axis-copy caller `0xD7D1A8` gives `0xD7C352`, and a disassembly of `0x140D7C352..0x140D7D1A8` contains no `ret`/`endbr` — the gate site and the axis-copy site are in the same function. Of the eight GOG call sites matching the fireMode prefix, `0xD7C352` is the only one followed by `testq %rax,%rax` before folding the weapon angle floats.

### Implementation

- `src/weapon/WeaponHook.cpp` — new file-scope `std::atomic<uintptr_t> weaponDataCallerRva{0xD7D6C2}`; `findWeaponDataSite()` (Steam fast path at `0xF137B0`, else unique 42-byte scan); `resolveWeaponDataCaller()` (E8 walk with pre/post structural context, ambiguous → no store); `activeWeaponDataHook` now reads the atomic and rejects a `0` gate; `installActiveWeaponIdentityHook` verifies 42 bytes but still patches only the first 15 (instruction boundary at offset 15).
- `src/hud/HudHook.cpp` — `installEntryHook` gained a `patchSize` argument (default = signature size) because verification now spans the RIP-relative security-cookie load that a relocated trampoline must **not** copy; `findUniqueHudEntry()` unique-match scanner; `installPauseMenuLifecycleHooks` keeps the Steam 20-byte fast path, otherwise scans, and logs the resolved RVAs.
- Fail-closed throughout: an unresolved caller gate is stored as `0` (never matches), and any scan miss keeps the existing `…signature mismatch…` messages with gameplay otherwise unaffected.

**Build / deploy (2026-10-05):** Release rebuilt; `ctest -C Release` **103/105** (pre-existing AMD GPU failures `hand-scene-depth-gpu`, `native-depth-seed-gpu`). Deployed `KharvoxLayer.dll` SHA-256 **`657ff02c0d13b7ab5e7afc75d396509ffbe754faf41486c2cae5152cda7da62f`**; Fix #6 build saved as `KharvoxLayer.dll.fix7-backup` (SHA `70a64ec3…`); `native_sfs_build.txt` line 2 updated; the Fix #6 session log archived to `%TEMP%\KHARVOX-logs-archive\2026-10-05_2216-fix6-played\`.

**Verification greps for the next session:**
- `Get-Content $env:TEMP\KHARVOX.log | Select-String 'active weapon identity|caller-gated|Pause lifecycle hooks installed|Pause QUAD|signature mismatch|weapon='`
- expect **no** `active weapon-data signature mismatch` and **no** `idMenuScreen_Shell_Pause lifecycle signature mismatch`; expect `…installed at RVA 0xf12190; decl paths are caller-gated to idHands at RVA 0xd7c352`, `ShowScreen RVA 0x1068ba0, HideScreen RVA 0x10670a0`, `Pause QUAD active`, and `weapon=` no longer `unknown`.

---

## 16. Quick Reference

| Item | Value |
|---|---|
| ReShade revert (elevated) | `Set-ItemProperty "HKLM:\SOFTWARE\Khronos\OpenXR\1\ApiLayers\Implicit" -Name "C:\ProgramData\ReShade\ReShade64_XR.json" -Value 0 -Type DWord` |
| DLL rollback | copy `KharvoxLayer.dll.orig` → `KharvoxLayer.dll`; restore `native_sfs_build.txt.orig` |
| **Deployed source-build SHA-256 (Fix #7, active)** | `657ff02c0d13b7ab5e7afc75d396509ffbe754faf41486c2cae5152cda7da62f` (`native_sfs_build.txt` line 2) |
| Backup — Fix #6 build | `KharvoxLayer.dll.fix7-backup` (SHA `70a64ec394981328b6a6106ddc986e3f6eba950c3836156f5d9a6de6c3943afc`, fixes #6) |
| Backup — Fix #5 build | `KharvoxLayer.dll.fix6-backup` (SHA `d69fb39f84385a059ea9c924bae503d1914c95c10210e3ff48c58993de3bcb43`, fixes #5) |
| Backup — Fix #4 build | `KharvoxLayer.dll.fix5-backup` (SHA `9dbfaa5b395cbc4fcd678a5b1f40b8fea5e520d2ee0836af38fc6ffd8b5913f7`, fixes #3+#4) |
| Backup — 3-fix build, pre-#4 | `KharvoxLayer.dll.sfsfix1-backup` (SHA `18b1c1ab0230be433b1d046e73535e276f43ee77d5fec534210e95ffe276dc6e`) |
| Patched DLL SHA-256 (legacy byte-patched release) | `5EC410FEDB37E1DFDF1D74A354CCC6E097764085F217E9C04043E5E48FCB37B8` (`.v111-gogpatched-backup`) |
| Supersampling (keep) | `C:\Program Files (x86)\Steam\config\steamvr.vrsettings` → `"supersampleScale" : 1`, `"supersampleManualOverride" : true` (§13) |
| Perf graphs in headset | same file → `"perfGraphInHMD" : true`, `"showAdvancedGraphs" : true` |
| Rebuild command | `cmake --build ..\kharvox-build\native --config Release --parallel` (workdir = source tree) |
| Deploy | copy `KharvoxLayer.dll` → `C:\Games\KHARVOX-v1.11\`; update line 2 of `native_sfs_build.txt` with new SHA-256 |
| Fix #4 verify grep | `Get-Content $env:TEMP\KHARVOX.log \| Select-String 'capture installed\|BACK-WEAPON\|PROJECTION\|signature mismatch'` |
| Fix #5/#6 verify grep | `Get-Content $env:TEMP\KHARVOX.log \| Select-String 'AER-SOURCE-PAIR\|layers=\|STEAMXR-AER-PAIR\|WEAPON\|signature mismatch'` |
| Fresh-launch log check | `Get-Content $env:TEMP\KHARVOX.log \| Select-String 'exited\|FATAL\|Frame \d+ submitted\|Session created\|accessors'` |
| Frame time / dropped check | `Select-String 'C:\Program Files (x86)\Steam\logs\vrcompositor.txt' -Pattern 'ApplicationTime\|presents' \| Select-Object -Last 6` |
| GPU hang check | `Get-WinEvent -FilterHashtable @{LogName='Application'; StartTime=(Get-Date).AddMinutes(-30)} \| Where-Object Message -match 'LiveKernelEvent'` |
| khronos XrResult ref | https://registry.khronos.org/OpenXR/specs/1.1/man/html/XrResult.html |
