# Star Citizen on Linux: HDR (and SteamVR) launch workarounds

Practical notes from running **Star Citizen** under **Wine (LUG / XL-style prefix)** on **Fedora/Nobara + KDE Plasma Wayland**, with **NVIDIA** and a real HDR display.

This is **not** an official CIG or LUG guide. It documents what broke for us when “normal” (SDR/X11) launch looked fine but **in-game HDR did not**, how **SteamVR/OpenVR** is hooked without letting RSI auto-start `vrserver` on every 2D launch, and what we changed so those paths work.

| | |
|--|--|
| **Audience** | Linux SC players who want real HDR (not just a bright SDR image), or SteamVR without launcher crashes |
| **Stack tested** | KDE Plasma Wayland, NVIDIA, Wine-tkg LUG runner, DXVK, RSI Launcher, SteamVR |
| **Companion** | Compare **normal** vs **HDR** vs **SteamVR** launch |

→ **Full write-up for GitHub Pages:** [docs/index.md](docs/index.md)

## Quick summary

**Real HDR on Plasma Wayland needs Wine’s Wayland driver (`winewayland`), not XWayland.**

**SteamVR needs the opposite of the default 2D launch:** Wine must **see** a real Steam install, and **`DISPLAY` must stay set** (`vrserver` still wants X11). Default launch **hides** `~/.local/share/Steam` so RSI does not auto-start SteamVR and crash.

| Path | How | Result |
|------|-----|--------|
| **Normal** (`sc-launch.sh`) | Keep `DISPLAY` → Wine X11 / XWayland; **Steam hidden** | Stable keyboard; **in-game HDR often wrong or freezes**; no surprise `vrserver` |
| **HDR** (`sc-launch-hdr.sh`) | `SC_HDR=1` → **unset `DISPLAY`** → winewayland; Steam still hidden | **Real HDR** if compositor HDR is on; keyboard can be stickier |
| **SteamVR** (`SC_STEAMVR=1`) | Keep `DISPLAY`; **do not hide Steam** | OpenVR can init; **not** combined with winewayland HDR |

## Repo layout

```text
starcitizen-linux-hdr/
  README.md                 ← you are here
  docs/
    index.md                ← main GitHub Pages article
    normal-vs-hdr.md        ← comparison table (deep)
    steamvr.md              ← Steam hide vs SteamVR/OpenVR opt-in
    attributes-hdr-patch.md ← force HDR=1 in attributes.xml before launch
    wine-system-tray.md     ← hide Wine tray icon (never close it)
    troubleshooting.md      ← sticky keys, alt-tab mouse, monitors, VR
  LICENSE
```

## Topics covered for other users

1. **Normal vs HDR launch** — keep `DISPLAY` (X11) vs unset `DISPLAY` (winewayland).  
2. **Pre-launch `attributes.xml` patch** — force `HDR=1` (and clear it on SDR) for a clean boot; on a proper HDR launch, the **in-game enable/disable** toggle then works fine for mode switching.  
3. **Wine System Tray** — hide the floating tray / task icon; **never close** it (HWND crash risk, and full exit if RSI “exit on close” is on).  
4. **Alt-tab mouse** — `IgnoreWindowFocus=0`.  
5. **Sticky Alt / keyboard** — tradeoff of winewayland vs X11.  
6. **SteamVR / OpenVR** — hide Steam by default (launcher crash); opt in with `SC_STEAMVR=1` (keep `DISPLAY`, no Steam tmpfs). HDR winewayland and SteamVR do not share one process tree.

## Enabling GitHub Pages (later)

1. Create a GitHub repo and push this folder.
2. **Settings → Pages → Build from branch**
3. Source: `/docs` (or root if you prefer).
4. Optional: use theme via `docs/_config.yml` (included).

## Status

Living notes from one machine (Nobara / Plasma / RTX 5090 / MSI 4K HDR). Adapt paths and display names to your setup.
