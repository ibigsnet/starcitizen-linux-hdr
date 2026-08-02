# Troubleshooting (Linux SC + HDR)

## HDR not applying

1. Confirm Plasma (or DE) **HDR is on** for that monitor.  
2. Confirm the **Wine process** has **no** `DISPLAY` in its environment when using the HDR launcher.  
3. Confirm `DXVK_HDR=1`.  
4. Confirm `attributes.xml` has `HDR=1` **before** starting the client (see [attributes-hdr-patch.md](attributes-hdr-patch.md)).  
5. Avoid “enable HDR only inside the game” on the **normal** X11 path — that is where freezes showed up for us.

## Floating “Wine System Tray” / extra task icon

- **Hide** it (KWin rule: minimize, skip taskbar, opacity 0).  
- **Do not close** it — can crash RSI Launcher (HWND).  
- Details: [wine-system-tray.md](wine-system-tray.md).

## Freezes when enabling HDR in-game

Classic sign of **XWayland + HDR toggle**. Use the HDR launcher (winewayland) instead of flipping HDR on a normal session.

## Sticky modifiers (Alt walk inverted, etc.)

- Prefer mouse-only anti-AFK (no synthetic Alt).  
- Fully quitting the client (not only alt-tab) clears Wine key state.  
- Fall back to normal launch if winewayland input is unusable that day.

## Mouse still looks / ships after alt-tab

`IgnoreWindowFocus` must be `0`.  
If it is `1`, SC intentionally keeps input while unfocused.

## Scroll wheel dead after closing SC

Desktop “game mouse” tweaks (e.g. Corsair middle-button scroll) must apply only while **`StarCitizen.exe`** runs — not while only **RSI Launcher** is open.  
Restore desktop scroll when the client exits even if the launcher stays up.

## Wrong monitor

Do not rely only on KWin `screen=0` / `screen=1` (order changes after reboot).  
Prefer “largest **landscape** output” or an explicit connector name you verify each session (`kscreen-doctor -o`).

## Gamescope + HDR

Treat as a **third** path. Nested HDR under KWin can crash depending on resolution/output. If pure winewayland HDR works, prefer it for reliability.
