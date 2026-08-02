# Normal vs HDR launch (detail)

## Environment flags

| Variable | Normal | HDR |
|----------|--------|-----|
| `SC_HDR` | unset / `0` | `1` |
| `DISPLAY` | set | **unset** |
| `SC_X11_DISPLAY` | optional | save before unset (host tools only) |
| `SC_STEAMVR` | `0` or `1` | usually `0` (VR path keeps DISPLAY) |
| `SC_WAYLAND` | default keep X11 | default Wayland when HDR; `0` forces X11 (breaks HDR often) |
| `DXVK_HDR` | may be `1` | **`1`** |
| `SDL_VIDEODRIVER` | unset | unset |
| IME modules | cleared | cleared |

## attributes.xml

| Attribute | Normal | HDR |
|-----------|--------|-----|
| `HDR` | force `0` | force `1` |
| `Width` / `Height` | leave or your choice | often pin 4K for stability |
| `IgnoreWindowFocus` | `0` recommended | `0` recommended |
| `HDRMaxBrightness` / `HDRRefWhite` | user taste | match panel (e.g. 1000 / 250) |

## Failure modes

| Symptom | Likely path | Fix direction |
|---------|-------------|-----------------|
| HDR toggle freezes game | X11/XWayland (normal path) | unset DISPLAY, winewayland HDR launch |
| Image never looks HDR | XWayland or compositor HDR off | enable HDR in DE; confirm Wine env |
| Sticky Alt / weird walk | winewayland | try normal launch or careful focus habits |
| Ship spins while alt-tabbed | `IgnoreWindowFocus=1` | set to `0` |
| SDR launch broken after HDR session | leftover `HDR=1` in attributes | normal launch must force `HDR=0` |

## In-game enable / disable (once HDR launch is correct)

With a proper **HDR launch**, the graphics-menu **HDR on/off** control works as expected: you can switch between true HDR and SDR presentation without relaunching. That is a good smoke test that winewayland + compositor HDR are actually wired up.

What fails is using that checkbox as a substitute for the HDR **launcher** while still on normal/X11.

## Why both paths exist

HDR is a **presentation + Wine backend** choice, not a single checkbox.  
Normal path optimizes for **input reliability**.  
HDR path optimizes for **correct color/brightness pipeline** (and then the in-game toggle is free to use).

Many of us keep **both** launchers:

- Daily / competitive feel → normal  
- Pretty space + OLED/min-LED panels → HDR (toggle HDR off in-menu if you want SDR without leaving the session)
