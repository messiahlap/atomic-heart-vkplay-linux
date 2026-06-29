# Atomic Heart (VK Play / GameCenter) on Linux — Lutris + UMU + Proton

**English** | [Русский](README.ru.md)

How to get the **VK Play (VKGC / GameCenter)** edition of **Atomic Heart** running on Linux through
**Lutris + umu-run + Proton**, fixing the instant startup crash, the broken video, and a
720p/blurry render, and getting native 4K.

> **TL;DR:** The game crashes at launch because Proton's Media Foundation backend
> (**winedmo**, and also **winegstreamer**) cannot play the intro movie
> `Launch_FHD_60FPS_PC_Steam.mp4`. **Rename/remove the `Launch_*.mp4` files** and force the
> GStreamer media backend with `PROTON_MEDIA_USE_GST=1`. Then fix the resolution in
> `GameUserSettings.ini`. HDR is **not** exposed by this build of the game. Details below.

---

## Tested environment

| | |
|---|---|
| OS | Bazzite (Fedora atomic), kernel 7.0.x |
| DE / session | KDE Plasma 6.6.5, kwin 6.6.5, **Wayland** |
| GPU / driver | NVIDIA RTX 4090, proprietary driver 610.43.02 |
| CPU | Ryzen 7 7800X3D |
| Launcher | Lutris 0.5.22, runner `umu-run` (umu-launcher 1.4.0) |
| Proton | GE-Proton10-34 |
| Game | Atomic Heart, **VK Play edition** (via GameCenter.exe), UE4 4.27.2 |

Launch chain: **Lutris → umu-run → GameCenter.exe (VKGC) → AtomicHeart-Win64-Shipping.exe**.

---

## Symptom 1 — instant crash on startup

The game crashes immediately at launch (`SecondsSinceStart=0`), crash dump like:

```
C:\users\steamuser\AppData\Local\AtomicHeart\Saved\Crashes\UE4CC-Windows-XXXX_0000
Unhandled Exception: EXCEPTION_ACCESS_VIOLATION reading address 0x0000000000000000
```

In the Proton log (`PROTON_LOG`):

```
winedmo_demuxer_create url L"../../../AtomicHeart/Content/Movies/Launch_FHD_60FPS_PC_Steam.mp4"
demuxer_create Failed to open input, error -22 (Invalid argument).
demuxer_create Failed to open input, error 1 (Error number 1 occurred).
winedmo_demuxer_create demuxer_create failed, status 0xc0000001
```

### Root cause

The startup movie `Launch_FHD_60FPS_PC_Steam.mp4` (H264 + AAC) is played through **Media Foundation**,
which Proton implements with **winedmo** (an FFmpeg backend). winedmo has a flawed byte-stream
read/`Seek()` implementation and cannot read the stream (a known bug that breaks startup videos in
other games too; videos are also broken on the Steam edition of Atomic Heart). Media Foundation
never gets a media source → `CreatePresentationDescriptor` returns NULL → NULL dereference → crash.

Switching to the other backend (`winegstreamer`) removes the winedmo crash, **but winegstreamer itself
crashes** in the `IMFByteStream → wg_parser` glue layer. The file itself is **fine** (it decodes with
host gstreamer without errors). In other words, both of Proton's MF backends fail on this movie.

### Fix — skip the startup movie

The proven workaround (same as on the Steam edition): remove the startup movies, the game boots
straight to the menu.

```bash
# path to Movies inside the VK Play prefix (adjust to yours)
MOV="$HOME/Games/atomic-heart-vk/pfx/drive_c/VK Play/Atomic Heart/AtomicHeart/Content/Movies"

# rename ALL Launch_* (PC_Steam + PS4/PS5/Xbox — so the game won't pick another variant)
for f in "$MOV"/Launch_*.mp4; do mv -v "$f" "$f.bak"; done
```

Revert (restore the movies):

```bash
for f in "$MOV"/Launch_*.bak; do mv "$f" "${f%.bak}"; done
```

> ⚠️ **VK Play GameCenter** verifies the GUP manifest (size + MD5) on check/update and will mark the
> renamed files as "missing" → re-download them. A normal launch usually leaves them alone.
> If it re-downloads on every start, bypass GameCenter's verification / launch
> `AtomicHeart-Win64-Shipping.exe` directly.

In-game videos (difficulty-selection cartoons, etc.) will just be **black** on Proton, but they do
**not** crash the game.

### Also — force the GStreamer backend

Add to the environment:

```
PROTON_MEDIA_USE_GST=1
```

Disables winedmo and routes video through winegstreamer. Useful for in-game videos and generally safer
on this stack. (This is the same variable GE-Proton sets in protonfixes for games with broken video.)

---

## Symptom 2 — blurry / 720p render, cursor clipped to the corner

After the logo, a black **1280×720** window appears, the picture is blurry, and in the menu the cursor
only moves within the top-left corner (the 720p zone over a 4K screen).

### Root cause

A mismatch in `GameUserSettings.ini`: `ResolutionSizeX/Y` is 4K, but the actual window size is taken
from `DesiredScreenWidth/Height=1280×720`.

### Fix

File:
```
<prefix>/drive_c/users/steamuser/AppData/Local/AtomicHeart/Saved/Config/WindowsNoEditor/GameUserSettings.ini
```

Set all resolution fields to native (example for 4K):

```ini
ResolutionSizeX=3840
ResolutionSizeY=2160
LastUserConfirmedResolutionSizeX=3840
LastUserConfirmedResolutionSizeY=2160
DesiredScreenWidth=3840
DesiredScreenHeight=2160
bUseDesiredScreenHeight=True
LastUserConfirmedDesiredScreenWidth=3840
LastUserConfirmedDesiredScreenHeight=2160
```

The key fields were `DesiredScreenWidth/Height`. After this: native 4K, cursor across the whole screen.

---

## HDR

**This build of the game has no HDR setting**, so UE4 never activates an HDR swapchain and no HDR is
emitted, regardless of which variables you set.

If a game *does* support HDR, then on KDE Plasma 6.6 + Wayland + NVIDIA (driver newer than 595.58.03)
HDR without gamescope is enabled like this (for GE-Proton):

```
PROTON_ENABLE_WAYLAND=1   # Wine Wayland driver
PROTON_ENABLE_HDR=1       # = DXVK_HDR=1 in GE-Proton
DXVK_HDR=1
```
plus enabling HDR in the game itself (`bUseHDRDisplayOutput=True`). On drivers older than 595.58.03 you
also need `vk-hdr-layer` + `ENABLE_HDR_WSI=1`.

> **gamescope** is not a viable HDR route here: it crashes GameCenter (VKGC) itself, and gamescope-HDR
> on NVIDIA has been broken since Plasma 6.5.

---

## Final Lutris configuration (env section)

```yaml
system:
  disable_runtime: true
  env:
    GAMEID: '0'
    LC_ALL: ''
    PROTONPATH: /home/<user>/.local/share/Steam/compatibilitytools.d/GE-Proton10-34
    PROTON_MEDIA_USE_GST: '1'        # winegstreamer instead of broken winedmo
    DXVK_HDR: '1'                    # HDR-ready (no effect in-game — no setting)
    PROTON_ENABLE_HDR: '1'
    PROTON_ENABLE_WAYLAND: '1'
    WINEDLLOVERRIDES: sl.interposer=
    STEAM_COMPAT_CLIENT_INSTALL_PATH: /home/<user>/.local/share/Steam
    STEAM_COMPAT_DATA_PATH: /home/<user>/Games/atomic-heart-vk
  gamescope: false
wine:
  runner_executable: /usr/bin/umu-run
```

For debugging video/MF you can temporarily add:
```
PROTON_LOG: -all,+loaddll,+module,+warn,+err,+mfplat,+quartz,+dmo
PROTON_LOG_DIR: /home/<user>
GST_DEBUG: '2'
```

---

## From-scratch checklist

1. Install the VK Play game via GameCenter (in Lutris through umu-run / GE-Proton).
2. Add `PROTON_MEDIA_USE_GST=1` to the environment.
3. Rename `Content/Movies/Launch_*.mp4` → `.bak`.
4. Launch → reach the menu → set resolution; if blurry/720p, fix `GameUserSettings.ini`
   (`DesiredScreenWidth/Height`).
5. Play.

---

## Sources / credits

- ProtonDB — Atomic Heart (app 668580): reports about broken videos and skipping startup movies.
- CachyOS proton-cachyos CHANGELOG — description of the winedmo `Seek()` bug.
- GE-Proton (GloriousEggroll) — the `PROTON_MEDIA_USE_GST` variable / protonfixes.
- ValveSoftware/Proton issue #6554 (Atomic Heart, videos don't render).

Diagnosis done by reverse-engineering `winedmo.so` and parsing Proton logs.

---

*If this helped, a star is appreciated. PRs confirming other hardware/drivers are welcome.*
