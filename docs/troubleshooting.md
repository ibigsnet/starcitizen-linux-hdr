# Troubleshooting (Linux SC + HDR)

## HDR not applying

1. Confirm Plasma (or DE) **HDR is on** for that monitor.  
2. Confirm the **Wine process** has **no** `DISPLAY` in its environment when using the HDR launcher.  
3. Confirm `DXVK_HDR=1`.  
4. Confirm `attributes.xml` has `HDR=1` **before** starting the client (see [attributes-hdr-patch.md](attributes-hdr-patch.md)).  
5. Avoid “enable HDR only inside the game” on the **normal** X11 path — that is where freezes showed up for us. On a **proper HDR launch**, the same enable/disable control works fine.

## Floating “Wine System Tray” / extra task icon

- **Hide** it (KWin rule: minimize, skip taskbar, opacity 0).  
- **Do not close** it — can crash RSI Launcher (HWND teardown), and can fully exit launcher/game if **exit Star Citizen / RSI Launcher on close** is enabled in launcher settings.  
- Details: [wine-system-tray.md](wine-system-tray.md).

## Freezes when enabling HDR in-game

Classic sign of **XWayland + HDR toggle** on the **normal** path. Use the HDR launcher (winewayland) instead.

Once you are already on that HDR launch, **enable/disable in the graphics menu is fine** and is a good way to switch modes without restarting.

## Sticky modifiers (Alt walk inverted, etc.)

- Prefer mouse-only anti-AFK (no synthetic Alt).  
- Fully quitting the client (not only alt-tab) clears Wine key state.  
- Fall back to normal launch if winewayland input is unusable that day.

## Mouse still looks / ships after alt-tab

`IgnoreWindowFocus` must be `0`.  
If it is `1`, SC intentionally keeps input while unfocused.

## Wrong monitor / wrong resolution list

Do not rely only on KWin `screen=0` / `screen=1` or a remembered `DP-*` name.  
Prefer the largest **landscape** output. On winewayland set `WAYLANDDRV_PRIMARY_MONITOR` to that connector or you only get the portrait panel’s modes.  
Do not force `Width`/`Height` to native every launch. Keep `AutoDetect=0`.  
Details: [displays-resolutions.md](displays-resolutions.md).

## Gamescope + HDR

Treat as a **separate** path. Nested HDR under KWin can crash depending on resolution/output. If pure winewayland HDR works, prefer it for reliability.

## Wrong GPU / no DLSS / launcher “No adapters found”

Displays on AMD, game should be NVIDIA: if `Game.log` chose RADV, the device **filter** is missing or the NVIDIA driver is not loaded.  
Do **not** set `VK_ICD_FILENAMES` to NVIDIA-only (winewayland still needs RADV to present).  
RSI Launcher is DXVK: a NVIDIA-only filter with no NVIDIA ICD → `Failed to initialize DXVK`.  
Details: [multi-gpu.md](multi-gpu.md).

## SteamVR / OpenVR

- **Launcher crash / Web Thread** on a 2D launch: Steam is visible to Wine. Hide `$HOME/.local/share/Steam` (tmpfs bind) unless you opted into VR.  
- **Headset not seen / OpenVR failed to locate module:** you hid Steam, or `DISPLAY` is unset (HDR winewayland). Use the SteamVR path: Steam visible + `DISPLAY` kept.  
- **Want HDR and VR together:** not the same process. HDR unsets `DISPLAY`; `vrserver` wants it set. See [steamvr.md](steamvr.md).  
- Start **SteamVR on the host** before (or able to start with) the RSI launcher.
