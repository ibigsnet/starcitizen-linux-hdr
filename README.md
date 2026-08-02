# Star Citizen on Linux: HDR launch workarounds

Practical notes from running **Star Citizen** under **Wine (LUG / XL-style prefix)** on **Fedora/Nobara + KDE Plasma Wayland**, with **NVIDIA** and a real HDR display.

This is **not** an official CIG or LUG guide. It documents what broke for us when “normal” (SDR/X11) launch looked fine but **in-game HDR did not**, and what we changed so the **HDR path** works.

| | |
|--|--|
| **Audience** | Linux SC players who want real HDR (not just a bright SDR image) |
| **Stack tested** | KDE Plasma Wayland, NVIDIA, Wine-tkg LUG runner, DXVK, RSI Launcher |
| **Companion** | Compare **normal launch** vs **HDR launch** side by side |

→ **Full write-up for GitHub Pages:** [docs/index.md](docs/index.md)

## Quick summary

**Real HDR on Plasma Wayland needs Wine’s Wayland driver (`winewayland`), not XWayland.**

| Path | How | HDR result |
|------|-----|------------|
| **Normal** (`sc-launch.sh`) | Keep `DISPLAY` → Wine X11 / XWayland | Stable keyboard; **in-game HDR often wrong or freezes** |
| **HDR** (`sc-launch-hdr.sh`) | `SC_HDR=1` → **unset `DISPLAY`** → winewayland | **Real HDR** if compositor HDR is on; keyboard can be stickier |

## Repo layout

```text
starcitizen-linux-hdr/
  README.md              ← you are here
  docs/
    index.md             ← main GitHub Pages article
    normal-vs-hdr.md     ← comparison table (deep)
    troubleshooting.md   ← sticky keys, alt-tab mouse, monitors
  LICENSE
```

## Enabling GitHub Pages (later)

1. Create a GitHub repo and push this folder.
2. **Settings → Pages → Build from branch**
3. Source: `/docs` (or root if you prefer).
4. Optional: use theme via `docs/_config.yml` (included).

## Status

Living notes from one machine (Nobara / Plasma / RTX 5090 / MSI 4K HDR). Adapt paths and display names to your setup.
