# Star Citizen on Linux: making HDR actually work

> Community write-up: **SDR/X11 “normal” launch** vs **HDR/winewayland launch**.  
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
| **In-game HDR** | Often broken / freezes | Works when compositor + winewayland are correct |

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
export SC_STEAMVR=0   # SteamVR keeps DISPLAY; skip for pure HDR
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

### 4. Patch `attributes.xml` before launch (all channels you use)

Star Citizen stores the HDR toggle in the profile, e.g.:

```text
…/StarCitizen/<LIVE|PTU|…>/user/Client/0/Profiles/default/attributes.xml
```

On **HDR** launch we set:

| Attribute | Value | Why |
|-----------|--------|-----|
| `HDR` | `1` | Game-side HDR flag |
| `Width` / `Height` | e.g. `3840` / `2160` | Stable fullscreen (optional; override with env) |
| `IgnoreWindowFocus` | `0` | Stop mouse look when alt-tabbed (ships spinning on desktop) |

On **normal** launch we set:

| Attribute | Value | Why |
|-----------|--------|-----|
| `HDR` | `0` | Leftover `HDR=1` from an HDR session can break SDR launches |
| `IgnoreWindowFocus` | `0` | Same alt-tab mouse issue |

Example snippets:

```xml
<Attr name="HDR" value="1"/>
<Attr name="Width" value="3840"/>
<Attr name="Height" value="2160"/>
<Attr name="IgnoreWindowFocus" value="0"/>
```

Optional env overrides we use:

```bash
SC_ATTR_WIDTH=3840 SC_ATTR_HEIGHT=2160 ./sc-launch-hdr.sh
SC_IGNORE_WINDOW_FOCUS=0 ./sc-launch-hdr.sh   # default
```

### 5. Reduce input-method interference

Hold-to-compose / IME popups under Wine are painful in cockpit:

```bash
export GTK_IM_MODULE=
export QT_IM_MODULE=
export SDL_IM_MODULE=
export XMODIFIERS=
export GLFW_IM_MODULE=
```

### 6. Launch RSI Launcher with the same Wine prefix

Same as LUG: run `RSI Launcher.exe` from the prefix after the env/attributes steps. Game starts from the launcher as usual.

### 7. Confirm HDR in-game

- Desktop looks HDR-capable (bright UI, PQ/HDR curve on the panel).
- In SC graphics options, HDR is on and peak brightness matches your panel (e.g. ~1000 nits).
- If the image looks like a flat SDR image with a “HDR” checkbox, you are probably still on XWayland — re-check that **`DISPLAY` is unset** for the Wine tree.

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

### Scroll wheel dead after closing the game

- Desktop “game mouse mode” (disable middle-button scroll tricks for Corsair) must only apply while **`StarCitizen.exe`** runs — **not** while only RSI Launcher is open.
- After client exit, restore desktop scroll even if the launcher stays up.

---

## Minimal reproduction matrix for other users

| Test | Expected if setup is correct |
|------|------------------------------|
| Normal launch, HDR off in game | Stable play, SDR |
| Normal launch, turn HDR on in game | Often freeze or no real HDR |
| HDR launch, compositor HDR on | Real HDR image |
| HDR launch with `DISPLAY` still set | Behaves like normal (no real HDR) |
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

Exact scripts live in your Wine prefix / LUG install; this repo documents **behavior**, not a full re-ship of proprietary game files.

---

## Further reading

- [Star Citizen LUG](https://wiki.starcitizen-lug.org/) — install, Wine, NVIDIA notes  
- [LUG Helper](https://github.com/starcitizen-lug/lug-helper)  
- DXVK HDR / Vulkan color space docs (upstream DXVK)  
- KDE Plasma HDR display settings  

---

## Contributing / scope

If you hit the same “normal works, HDR does not” pattern on another distro/compositor, useful reports include:

- Distro + compositor (Plasma/Hyprland/etc.) + GPU  
- Whether `DISPLAY` is set in the Wine process environment  
- Whether compositor HDR is enabled  
- Whether `attributes.xml` has `HDR=1`  
- Freeze vs black screen vs SDR-looking image  

See also: [normal-vs-hdr.md](normal-vs-hdr.md), [troubleshooting.md](troubleshooting.md).
