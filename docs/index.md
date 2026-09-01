# Star Citizen on Linux: making HDR actually work

> Community write-up: **SDR/X11 “normal” launch** vs **HDR/winewayland launch** vs **SteamVR/OpenVR**.  
> Environment reference: **KDE Plasma Wayland**, **NVIDIA**, **Wine (LUG-style prefix)**, **DXVK**, RSI Launcher.

---

## Why “normal” launch is not enough for HDR

On Plasma Wayland, if Wine keeps using **X11** (`DISPLAY` is set), the game usually goes through **XWayland**.

What we observed:

1. DXVK presents as **SDR / `VK_COLOR_SPACE_SRGB_NONLINEAR`** (or equivalent non-HDR path).
2. Toggling **HDR in the game** (or some Tech-Preview builds) can **freeze** or never look like true HDR.
3. Keyboard modifiers (Alt, etc.) are often **more reliable** on this path — which is why people stay on “normal” until they care about HDR.

**Real HDR** needs:

- Compositor **HDR enabled** on the output (System Settings → Display → HDR).
- A Vulkan/DXVK path that can use HDR color spaces.
- Wine talking to Wayland **natively** (`winewayland`), not only XWayland.

---

## The two launch paths (compare)

| | **Normal (SDR / keyboard-friendly)** | **HDR (true HDR / winewayland)** |
|--|--------------------------------------|----------------------------------|
| **Entry** | `./sc-launch.sh` | `./sc-launch-hdr.sh` → sets `SC_HDR=1` → `./sc-launch.sh` |
| **`DISPLAY`** | **Kept** (e.g. `:0`) | **Unset** (forces winewayland) |
| **SDL** | `unset SDL_VIDEODRIVER` (same) | same |
| **DXVK** | `DXVK_HDR=1` often still set | `DXVK_HDR=1` **required** |
| **`attributes.xml` HDR** | Forced **`HDR=0`** so leftover HDR flags don’t break SDR | Forced **`HDR=1`**, optional width/height pin (e.g. 3840×2160) |
| **IgnoreWindowFocus** | Prefer **`0`** (don’t keep reading mouse when alt-tabbed) | same |
| **Keyboard** | Better under Wine X11 | Can get **sticky Alt / modifiers** on winewayland |
| **Gamescope** | Optional separate script | Optional; pure HDR path does **not** need gamescope |
| **Steam / OpenVR** | **Hidden** (tmpfs over Steam dir) so RSI does not auto-start `vrserver` | Same hide — **HDR is not the VR path** |
| **In-game HDR toggle** | Often broken / freezes | **Works** — enable/disable between HDR and SDR freely once this path is correct |

### Mental model

```text
Normal path
  Plasma Wayland
       │
       ▼
  DISPLAY=:0  ──►  Wine X11  ──►  XWayland  ──►  mostly SDR
       │
       └── good for WASD / modifiers; bad for real HDR

HDR path
  Plasma Wayland (HDR on)
       │
       ▼
  DISPLAY unset  ──►  winewayland  ──►  real HDR presentation
       │
       └── DXVK_HDR=1 + attributes HDR=1
```

---

## Prerequisites (both paths)

1. **Install via [LUG Helper](https://github.com/starcitizen-lug/lug-helper)** (or equivalent Wine prefix with RSI Launcher).
2. **Plasma Wayland** session (or another compositor with working HDR).
3. **HDR enabled** on the monitor you play on (KDE: *System Settings → Display & Monitor → HDR*).
4. Recent enough **Wine** build used by LUG / your runner (winewayland + color management as available).
5. Check Wayland color protocol (optional):

   ```bash
   wayland-info | grep -i color
   ```

   Presence of something like `wp_color_manager_v1` is a good sign for modern HDR plumbing.

---

## HDR launch checklist (what we actually do)

These are the steps encoded in our `sc-launch-hdr.sh` + `sc-launch.sh` when `SC_HDR=1`.

### 1. Export HDR mode for the shared launcher

```bash
export SC_HDR=1
export SC_STEAMVR=0   # SteamVR keeps DISPLAY; skip for pure HDR (see steamvr.md)
# optional: remember X display for host tools only
export SC_X11_DISPLAY="${DISPLAY:-:0}"
```

### 2. Force winewayland (the critical bit)

```bash
# Do NOT keep DISPLAY for the Wine process tree when you want real HDR
unset DISPLAY
# Leave SDL_VIDEODRIVER unset so Wine/SDL can use Wayland
unset SDL_VIDEODRIVER
```

**If you leave `DISPLAY` set**, Wine tends to stay on X11/XWayland and HDR fails the way “normal” does.

### 3. Enable DXVK HDR

```bash
export DXVK_HDR=1
```

### 4. Patch `attributes.xml` **before** RSI Launcher starts the game

We still pre-set `HDR=1` so the client **boots** on the HDR path with a known-good flag (and normal launch forces `HDR=0` so leftover HDR sessions do not break SDR).

That is separate from the **in-game** checkbox: once you are already on a **proper HDR launch** (winewayland + compositor HDR + `DXVK_HDR=1`), the graphics-menu **enable/disable HDR** toggle works fine to switch presentation modes without needing a relaunch.

What does *not* work: relying on that same in-game toggle alone under the **normal / X11** path — that is where freezes and fake-SDR “HDR” showed up for us.

Star Citizen reads profile graphics flags from disk when the **game client** starts, for example:

```text
…/StarCitizen/<LIVE|PTU|…>/user/Client/0/Profiles/default/attributes.xml
```

Our launcher **rewrites that file for every channel** (LIVE, PTU, EPTU, …) *before* `RSI Launcher.exe` runs:

```text
sc-launch-hdr.sh
  → SC_HDR=1
  → sed/patch attributes.xml  (HDR=1, resolution, IgnoreWindowFocus)
  → unset DISPLAY + DXVK_HDR=1
  → start RSI Launcher
  → user hits Launch → client already sees HDR=1
```

| Attribute | HDR launch | Normal launch | Why force it |
|-----------|------------|---------------|--------------|
| **`HDR`** | **`1`** | **`0`** | Game-side HDR flag. Leftover `1` after an HDR session can break SDR launches; leftover `0` means the client may boot without HDR even if DXVK is ready. |
| **`Width` / `Height`** | e.g. **3840×2160** (optional pin) | leave / user choice | Stable fullscreen on a 4K HDR panel; override with `SC_ATTR_WIDTH` / `SC_ATTR_HEIGHT`. |
| **`IgnoreWindowFocus`** | **`0`** | **`0`** | If `1`, mouse look continues while alt-tabbed (ships spin on the desktop). |

Example fragment after an HDR pre-patch:

```xml
<Attr name="HDR" value="1"/>
<Attr name="HDRMaxBrightness" value="1000"/>
<Attr name="HDRRefWhite" value="250"/>
<Attr name="Width" value="3840"/>
<Attr name="Height" value="2160"/>
<Attr name="IgnoreWindowFocus" value="0"/>
```

**Safety:** keep a backup (e.g. `attributes.xml.hdr-bak`) on first edit. Patch **all channels you play** (LIVE and PTU are separate trees).

Deep dive: [attributes-hdr-patch.md](attributes-hdr-patch.md).

### 5. Hide the Wine System Tray (task icon) — do **not** close it

On HDR/winewayland especially, Wine often shows a floating **“Wine System Tray”** (extra panel/task icon).

| Do | Don’t |
|----|--------|
| Minimize / skip taskbar / opacity 0 via **KWin window rule** | Click the tray window’s **X** / destroy the window |
| Re-apply hide a few seconds after Electron starts (tray is late) | `kill` explorer / tray process in the prefix |

**Why:** Closing that tray window can destroy HWNDs Electron still uses → **RSI Launcher hard-crashes**. It can also tear down the whole session if you have **exit Star Citizen / RSI Launcher on close** enabled in the launcher settings. Hiding keeps the process and HWND alive; there is no sound.

We schedule delayed hide passes (e.g. 2s, 5s, 10s, …) after launch and keep a permanent KWin rule for title `Wine System Tray`.

Deep dive: [wine-system-tray.md](wine-system-tray.md).

### 6. Reduce input-method interference

Hold-to-compose / IME popups under Wine are painful in cockpit:

```bash
export GTK_IM_MODULE=
export QT_IM_MODULE=
export SDL_IM_MODULE=
export XMODIFIERS=
export GLFW_IM_MODULE=
```

### 7. Launch RSI Launcher with the same Wine prefix

Same as LUG: run `RSI Launcher.exe` from the prefix **after** env + attributes + tray hide setup. Game starts from the launcher as usual.

### 8. Confirm HDR in-game

- Desktop looks HDR-capable (bright UI, PQ/HDR curve on the panel).
- In SC graphics options, HDR is on and peak brightness matches your panel (e.g. ~1000 nits).
- **Toggle test:** with a proper HDR launch, you can **enable and disable HDR in the menu** and switch between true HDR and SDR presentation cleanly. If enable freezes or never looks right, you are almost certainly still on the normal/XWayland path.
- If the image looks like a flat SDR image with a “HDR” checkbox, re-check that **`DISPLAY` is unset** for the Wine tree.

---

## Normal launch (when you do *not* want HDR)

Use when:

- Sticky Alt / inverted walk under winewayland is worse than no HDR.
- You are debugging input, not picture.

```bash
./sc-launch.sh
# SC_HDR unset or 0 → DISPLAY kept → Wine X11 path
# attributes: HDR forced to 0
```

You can still export `DXVK_HDR=1` in the shared script; without winewayland + compositor HDR it will not give a correct HDR path.

---

## SteamVR / OpenVR (third path)

Default and HDR launches **hide the host Steam directory** from Wine (empty tmpfs bind). That stops RSI Launcher from loading `steamclient` and auto-starting **`vrserver`**, which has taken down the launcher on Plasma Wayland.

To **hook VR**:

1. Do **not** hide Steam.  
2. **Keep `DISPLAY`** — SteamVR’s `vrserver` still wants X11.  
3. Use a dedicated env (we use `SC_STEAMVR=1`) so this is not mixed with `unset DISPLAY` HDR.

That is incompatible with the winewayland HDR trick in the same process. Full write-up: **[steamvr.md](steamvr.md)**.

---

## Displays and resolution (mixed portrait + landscape)

Winewayland’s primary is often the output at **(0,0)** — frequently a **portrait** side panel — so the client only lists that panel’s modes. KWin `screen=0` also **reorders after reboot**.

Practical extras:

- Pick the **largest landscape** output by geometry (not connector name).  
- Set **`WAYLANDDRV_PRIMARY_MONITOR`** to that connector on the HDR path.  
- Do **not** rewrite `Width`/`Height` to panel native every launch (that wipes in-game res).  
- Keep **`AutoDetect=0`** after the client exits; Fast Shutdown writes `AutoDetect=1` and the next boot re-detects quality.  
- Exclusive fullscreen (`WindowMode=1`) binds 0,0; prefer **borderless** (`2`) on mixed setups.

Full write-up: **[displays-resolutions.md](displays-resolutions.md)**.

---

## Multi-GPU (render vs display)

If the **panel is on AMD** (or an iGPU) and you want **NVIDIA** for the game (DLSS), DXVK/Vulkan will still pick the **display** adapter unless you filter.

- Set `DXVK_FILTER_DEVICE_NAME` / `VKD3D_FILTER_DEVICE_NAME` to the **render** GPU’s `vulkaninfo` name.  
- **Do not** set `VK_ICD_FILENAMES` to NVIDIA-only — winewayland needs the **display** ICD to present.  
- If the NVIDIA kmod is missing, that filter makes **RSI Launcher** die with `DXVK: No adapters found`.

Full write-up: **[multi-gpu.md](multi-gpu.md)**.

---

## Optional: gamescope HDR (separate path)

Some setups use **gamescope** with `--hdr-enabled` (and friends) instead of pure winewayland.

Notes from our machine:

- Nested gamescope under KDE can be picky about **which output** (`-O DP-3`, display index).
- High supersample + HDR flags has crashed under KWin/NVIDIA for us.
- Pure **`sc-launch-hdr.sh`** remains the simpler “real HDR” path.
- `--force-grab-cursor` is a **gamescope** option only (cursor capture). The pure HDR script does **not** set it.

---

## Related desktop issues (not HDR-specific, but we hit them)

### Sticky Alt / WASD under winewayland

- Prefer **normal** launch for pure keyboard debugging.
- Or try `SC_WAYLAND=0 ./sc-launch-hdr.sh` (keeps HDR env but forces X11 — **HDR may freeze** again).
- Avoid injecting Alt via automation (anti-AFK should jiggle **mouse only**).

### Mouse keeps controlling the ship after alt-tab

- Set **`IgnoreWindowFocus=0`** (see above).
- With `1`, SC keeps reading mouse while unfocused.

### Wrong monitor (vertical side panel)

- KWin **screen index** is not “primary” and can **swap after reboot**.
- Prefer pinning by **geometry** (largest landscape output), not `screen=0`.
- Launchers (RSI) need not be forced; only the game client.

---

## Minimal reproduction matrix for other users

| Test | Expected if setup is correct |
|------|------------------------------|
| Normal launch, HDR off in game | Stable play, SDR |
| Normal launch, turn HDR on in game | Often freeze or no real HDR |
| HDR launch, compositor HDR on | Real HDR image |
| HDR launch, disable then re-enable HDR in menu | Works cleanly (true mode switch on winewayland) |
| HDR launch with `DISPLAY` still set | Behaves like normal (no real HDR / toggle may freeze) |
| SteamVR launch, Steam dir visible, `DISPLAY` set | OpenVR can init; `vrserver` can run |
| SteamVR launch but Steam still hidden | OpenVR “failed to locate module” / no runtime |
| HDR + SteamVR flags together | VR wins on `DISPLAY`; not a true winewayland HDR session |
| Alt-tab with `IgnoreWindowFocus=0` | Mouse does not turn ship |
| Alt-tab with `IgnoreWindowFocus=1` | Mouse still turns ship |

---

## Example launcher wrappers (conceptual)

**HDR:**

```bash
#!/usr/bin/env bash
export SC_HDR=1
export SC_STEAMVR=0
export SC_X11_DISPLAY="${DISPLAY:-:0}"
exec ./sc-launch.sh "$@"
```

**Inside shared `sc-launch.sh` (HDR branch only):**

```bash
if [[ "${SC_HDR:-0}" == "1" && "${SC_WAYLAND:-1}" != "0" ]]; then
  unset DISPLAY
fi
export DXVK_HDR=1
# patch attributes HDR=1 (and optional resolution)
# start RSI Launcher with Wine from the prefix
```

**SteamVR:**

```bash
#!/usr/bin/env bash
export SC_STEAMVR=1
# Do not unset DISPLAY. Do not hide ~/.local/share/Steam.
exec ./sc-launch.sh "$@"
```

Exact scripts live in your Wine prefix / LUG install; this repo documents **behavior**, not a full re-ship of proprietary game files.

---

## Further reading

- [Star Citizen LUG](https://wiki.starcitizen-lug.org/) — install, Wine, NVIDIA notes  
- [LUG Helper](https://github.com/starcitizen-lug/lug-helper)  
- DXVK HDR / Vulkan color space docs (upstream DXVK)  
- KDE Plasma HDR display settings  
- SteamVR / OpenVR on Linux (Valve runtime; keep `DISPLAY` for `vrserver`)  

---

## Contributing / scope

If you hit the same “normal works, HDR does not” pattern on another distro/compositor, useful reports include:

- Distro + compositor (Plasma/Hyprland/etc.) + GPU  
- Whether `DISPLAY` is set in the Wine process environment  
- Whether compositor HDR is enabled  
- Whether `attributes.xml` has `HDR=1`  
- Freeze vs black screen vs SDR-looking image  

See also: [normal-vs-hdr.md](normal-vs-hdr.md), [steamvr.md](steamvr.md), [displays-resolutions.md](displays-resolutions.md), [multi-gpu.md](multi-gpu.md), [troubleshooting.md](troubleshooting.md).
