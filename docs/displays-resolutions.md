# Displays, resolution, and why the game picks the wrong panel

Mixed **portrait + landscape** (ultrawide, 4K, a vertical Discord/chat screen) is a common Linux SC setup. Wine, KWin, and Star Citizen each have a different idea of “primary.” If those disagree, you get:

- A **1440×2560** (or similar) mode list when the big panel is 4K / 32:9  
- Exclusive fullscreen stuck on the **leftmost** output  
- In-game resolution **reset** every launch  
- Quality presets **wiped** after a “successful” quit  

This page is the **display/resolution** counterpart to [HDR](index.md) and [SteamVR](steamvr.md). Same stack: Wine prefix + RSI Launcher, Plasma Wayland. Adapt connector names; do **not** hardcode `DP-3` vs `DP-9`.

---

## What goes wrong

| Actor | What it treats as “main” | Failure mode |
|-------|--------------------------|--------------|
| **KWin** | `screen=0` / priority that **reorders after reboot** | Pin/place scripts send the game to the portrait panel |
| **winewayland** | Output at compositor **(0,0)** if unset | Portrait is often leftmost → only that panel’s modes |
| **Star Citizen** | `attributes.xml` `Width` / `Height` / `WindowMode` / `AutoDetect` | Autodetect rewrites quality; exclusive FS binds 0,0 |
| **You** | “The ultrawide I play on” | None of the above unless you pick by **geometry** |

Connector names (**DP-3**, **HDMI-A-1**) also change when you replug or the GPU re-probes. Scripts that match `DP-3` will silently follow the wrong cable.

---

## Rule: pick the largest **landscape** output

Among **enabled** outputs, prefer `width > height`, then **largest pixel area**. That selects an ultrawide or 16:9 play panel over a tall side monitor without naming the port.

Conceptual (KDE: `kscreen-doctor -o`; other DEs: `wlr-randr`, `hyprctl monitors`, …):

```text
enabled outputs
  → keep landscape (W > H)
  → sort by W×H descending
  → that connector is MAIN
```

Export something like `MAIN_OUTPUT`, `MAIN_W`, `MAIN_H`, `MAIN_X`, `MAIN_Y`. Optional override: force a connector if you must.

Use this for:

1. **`WAYLANDDRV_PRIMARY_MONITOR`** (Wine)  
2. Optional **KWin send-to-output** / place window  
3. Optional **gamescope `-O`** / display index — still prefer geometry, not a remembered index  

---

## Wine: `WAYLANDDRV_PRIMARY_MONITOR`

On **winewayland** (HDR path: `DISPLAY` unset), Wine’s primary is **not** “KDE primary.” If unset, it often uses the output at **(0,0)**.

If that is a **portrait** panel, the client only sees that panel’s modes (e.g. 1440×2560 / 2560×1440) even though the play screen is 3840×2160 or 7680×2160.

```bash
# Connector name from the landscape pick above (e.g. DP-9 this boot, DP-3 next)
export WAYLANDDRV_PRIMARY_MONITOR="$MAIN_OUTPUT"
```

Set this on **HDR / winewayland** launches even if you do **not** rewrite `Width`/`Height`. Mode **enumeration** and **which output is primary** are separate from the resolution you last saved in-game.

---

## `attributes.xml`: what to force vs what to leave

Profile path (per channel: LIVE, PTU, …):

```text
…/StarCitizen/<channel>/user/Client/0/Profiles/default/attributes.xml
```

| Attr | Recommendation | Why |
|------|----------------|-----|
| `HDR` | `1` on HDR launch, `0` on SDR | Leftover HDR flag breaks the other path; see [attributes-hdr-patch.md](attributes-hdr-patch.md) |
| `IgnoreWindowFocus` | `0` | `1` keeps mouse-look when alt-tabbed |
| `WindowMode` | If `1` (exclusive FS) on HDR, coerce to **`2` (borderless)** | Exclusive FS binds Wine **(0,0)** — often the portrait panel |
| `AutoDetect` | Keep **`0`** | `1` re-runs hardware detect and **overwrites `SysSpec_*` / quality** |
| `Width` / `Height` | **Do not** force panel native every boot unless you opt in | Writing 7680×2160 (or 4K) every launch **wipes** in-game resolution |

Optional: `SC_ATTR_WIDTH` / `SC_ATTR_HEIGHT` for an explicit size, or a flag to pin **native landscape** once you know you want that.

The client **rewrites `AutoDetect=1` in-session** and on **Fast Shutdown** (`CSystem::Quit` / `ExitOnQuit`). A pin only at **launcher start** is not enough if you hit Play again from a still-open RSI Launcher. A small watcher that sets `AutoDetect=0` while `StarCitizen.exe` runs and again **after** the process exits (quit write lands late) is what actually makes quality stick.

Do **not** rewrite `SysSpec_*` unless you are restoring a snapshot you took on purpose.

---

## Exclusive fullscreen vs borderless

`WindowMode`:

- `0` — windowed  
- `1` — **exclusive fullscreen**  
- `2` — **borderless**

On winewayland + a portrait output at (0,0), **exclusive** often fullscreens the **wrong** panel. **Borderless** lets the compositor place a 32:9 or 16:9 window on the landscape output.

If you pin/place with KWin, skip fighting a window that is **already** fullscreen on the landscape monitor (avoids fullScreen true/false thrash and “snaps back to borderless”).

---

## Placing the window (optional)

RSI Launcher does not need to live on the play panel. Only the **game client** does.

- Prefer **connector / geometry**, not `screen=0`.  
- Place at the profile’s `Width`×`Height` **centered on the landscape output**, not stretched to fill a 32:9 panel when the game is 16:9.  
- Caption/class filters: do not match `gamescope`, Discord, or anti-AFK helper titles.

---

## Dual GPU (PRIME), briefly

If **displays are on GPU A** and the game **renders on GPU B**, Vulkan device filters (`DXVK_FILTER_DEVICE_NAME` / `VKD3D_FILTER_DEVICE_NAME`) must name the **render** GPU. Winewayland still needs the **display** GPU’s ICD to present. Do not set `VK_ICD_FILENAMES` to the render GPU only.

That is independent of “which monitor,” but it is why a 4K/ultrawide mode list can appear while the compositor is scanning out on the other card.

---

## Checklist

1. List enabled outputs; pick **largest landscape** (not `DP-*`, not `screen=0`).  
2. HDR path: `WAYLANDDRV_PRIMARY_MONITOR=<that connector>`.  
3. Do not force `Width`/`Height` to native every launch unless you opted in.  
4. `AutoDetect=0` at start, **and** after the client exits (Fast Shutdown write).  
5. Avoid exclusive FS (`WindowMode=1`) on mixed portrait+landscape winewayland.  
6. Confirm in-game resolution after one clean quit + relaunch.

---

## Related

- [index.md](index.md) — HDR vs normal  
- [attributes-hdr-patch.md](attributes-hdr-patch.md) — `HDR=1` / `HDR=0`  
- [steamvr.md](steamvr.md) — `DISPLAY` kept for VR  
- [troubleshooting.md](troubleshooting.md)  
