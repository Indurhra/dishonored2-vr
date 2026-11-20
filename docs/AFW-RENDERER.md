# AFW renderer, PSVR2 triggers and the ImGui VR menu

Ported from the Indurhra DLSS/AFW fork (`dishonored-2-dlss-afw`, docs `DLSS-AFW-IMPLEMENTATION.md`, `PSVR2-TRIGGERS.md`) onto this
codebase. Tuning follows `AFW_RENDERING_GUIDE.md` (the user's AFW guide; concepts only, nothing copied from engines it describes).

## Renderer row (VR menu)

Open the VR menu with the **right grip at your right temple** in gameplay: page 1 is VR Settings, the right stick (left / right) pages on to the Body, Controllers and Gestures guide, the same gesture closes it. Holding left Y opens the game's pause menu (upstream behaviour). Pause > VR Settings is the game's own options page.
Row 13 of page 1, **Renderer**, has three choices (saved in `build\vr-settings.cfg`, file version 12):

| Choice | What renders |
|---|---|
| Native stereo | both eyes every frame |
| Alternate eye (built-in warp) | one eye per frame; the other eye's last image is re-projected (`stale_eye_warp.hpp`) |
| AFW (alternate frame warping) | one eye per frame; the other eye is **rebuilt** from the fresh eye and its own last image (VR Stereo Hub's open AFW, compiled in) |

Choose with the stick, then press **A** (or point at the green **A: APPLY** chip): every switch is live, no restart. Sessions start
alternate-capable, and Native simply renders both eyes every frame (as menus already did). If alternate eye could not start (its
prerequisites failed) the row shows **unavailable** for Alternate/AFW. `build
ative-classic` restores upstream's paired-only start for
Native. Any AFW refusal falls back to the built-in warp for that frame; a hard failure (plugin missing, ABI mismatch, eye-size change)
falls back for the session, and the description plate shows the reason.

Requirements: none beyond alternate eye. AFW is VR Stereo Hub's open-source held-eye rebuild from their Dishonored 1 VR mod
(`third_party/vsh_afw`, zlib license, modified; credited in `THIRD_PARTY_NOTICES.md`), compiled into the mod DLL. The closed PureDark
plugin is no longer used.

## How AFW is wired

- `third_party/vsh_afw/core/gfx/afw_warp.cpp`: the rebuild (D3D11). Per held-eye pixel it takes the hands and weapon from the fresh eye
  (reprojected by depth), the world from the held eye's own last image carried by the camera change, and the fresh eye's world where
  the held one moved or was uncovered (the stale test), with a disocclusion fill. Small adapters stand in for VSH's logging, depth
  probe and clarity headers.
- `src/afw_vsh.hpp` (the Dishonored 2 adapter): per captured fresh eye, one compute pass turns the R32 reverse-Z device depth into
  linear metres and writes the foreground mask; the eye pose and grip poses come from the stale warp's `WarpCamera`. No engine matrices.
- `src/game_xr.hpp`: at the held-eye seam (`v792`), AFW is tried first; on refusal the built-in warp draws that frame.

### Player-model depth (the part that made it work in D2)

Dishonored 2 compresses the depth of the player's models (`r_playerDepthHack`) so flat-screen arms never clip walls. The mod's cvar
override only changes the cache; loading a player restores the models' value, so the arms keep compressed depth. Measured from the
stereo disparity of still hands in the simulator (`afw capture` with depth dumps): world pixels need a factor 1.00, the free hand and
arms 5.04 (IQR 5.00-5.07), the held sword and **the hand that holds it** 9.0 (IQR 8.96-9.10). Device depth alone cannot tell the two
classes apart, so per pixel the adapter takes the class whose point lands nearer one of this render's grips or palms. Held items are
not in the rig mask; device depth above 0.04 (a world surface reaches that only within 25 cm) marks them. When the native
player-depth path (`build\world-hand-depth`) has verifiably set the models to world depth, the factors drop to 1 automatically.

## Tuning

Console (`artifacts\vr-command.txt`): `afw status`, `afw on|off`, `afw capture N`, `afw tune <key> <value>` (also one `key value` per
line in `build\afw-tune`):

| Key | Default | Meaning |
|---|---|---|
| `mask` | 1 | the foreground (arms, body, held items) from the rig mask + near range; 0 = nearer than `fg` metres |
| `rigdepth` | auto (5) | arms' depth compression; setting it pins it |
| `helddepth` | 9 | held items' (and the holding hand's) compression |
| `near` | 0.04 | device depth above which a non-rig pixel is a held item (0 = rig mask only) |
| `stillmm`, `stilldeg` | 3, 1 | hands moving less than this (head-relative, per two frames) count as still: the held eye keeps its own shading |
| `stale` | 0.015 | the stale test's relative depth tolerance |
| `nearmiss` | 6 | texels: a near-miss candidate beats the fill |
| `fg`, `body` | 0.75, 0.45 | metres, used only with `mask 0` |
| `heldhands`, `edgehands`, `stillshade`, `ownhands` | VSH defaults | VSH switches (-1 keeps the default) |
| `debug` | 0 | tints the held eye by source: green fresh hands, orange held own hands, blue fresh world, red last resort, magenta fill |

Knobs (files in `build\`): `afw` (start with AFW, for development runs without saved settings), `alternate-eye`, `menu-gdi` (old
GDI panel), `no-psvr2-triggers`.

## PSVR2 adaptive triggers

`src/psvr2_triggers.hpp` publishes the wanted left-trigger profile (pistol wall, crossbow wall, power vibration while pulled) into a
250 ms shared-memory lease; `DVR_psvr2\dvr_psvr2_bridge.exe` (source `third_party/psvr2/Bridge.cpp`, PSVR2Toolkit, MIT) applies it.
Only when the OpenXR runtime is SteamVR **and** the headset is a PlayStation VR2: other headsets never start the helper. The item comes
from the left hand's native equip name. `psvr2 status` prints runtime/headset, helper state and the current profile.

## ImGui VR menu

`src/vr_menu_imgui.hpp` draws the main settings page with Dear ImGui 1.92.8 (`third_party/imgui`, MIT) into the existing 640x560 panel
texture, at the same row rectangles, so pointer and stick navigation are unchanged. Calibration pages keep the GDI image. Preview
without the game: `tools\build-vr-menu-preview.cmd` → `artifacts\vr-menu-preview-*.png`.

## Build

`tools\build-dll-fast.cmd <name>.dll` builds MinHook, ImGui and the mod DLL only (no check suite, warnings not fatal: this toolset
reports 9 pre-existing upstream warnings under `/WX`). The full `tools\build.ps1` also needs the gitignored SDK binaries
(`third_party\dlss\lib\nvsdk_ngx_d.lib` 310.9.16, `third_party\fidelityfx\libs\*.lib` from Luma Framework) and currently stops at the
upstream `tests\note_tablet_check.cpp:45` assertion.

## AFW capture and the tuning record (2026-11-20, simulator)

`afw capture N` saves, per rebuilt held eye: `-rebuilt`, `-native` (the same eye's next native frame, one alternation later: the
reference, as on the Dishonored 1 AFW sheet), `-held` (what the held eye shows without AFW), `-fresh`, and the fresh eye's
`-freshdepth` (R32) and `-freshrig` (R8). `python tools/afw_capture_sheet.py <dir> <out.png> [--crop hand]` scores hand-region error
(MAE 0-255) and ghost fraction (pixels off by more than 48) for AFW and for the held frame. Runs: `vr_selftest.py ... --every
"afw capture 12"` (about 200 pairs per scenario). Pairs whose held-frame hand error is 15 or more are "motion".

Ratios are AFW / held frame (lower is better; 1.00 = as good as holding the old frame):

| Scenario | still hands | hands in motion | ghosting in motion | whole image |
|---|---|---|---|---|
| head sway | 0.84 | 0.65 | 0.55 | **0.77** |
| run (sword held) | 1.00 | 0.86 | 0.81 | **0.92** |
| walk | 0.97 | - | - | **0.95** |
| sword swing (two runs) | 1.00 / 1.00 | 0.93 / 0.91 | 0.90 / 0.87 | **0.98 / 0.97** |
| flick | 1.01 | - | - | **1.00** |

How it got there (swing, whole image): depth threshold only 1.30 -> rig mask 1.13 -> still-hands report 1.11 -> arms x5 ~1.15 (motion
0.65 in its best run) -> held sword x9 and per-pixel class by grip distance 0.97-0.98 with still hands exactly 1.00. Fast swings have a
floor: the reference is one tick later than the fresh eye, and the rebuild shows the fresh eye's pose (no extrapolation). A simulator is
not a headset: comfort, latency and the real compositor are untested.
