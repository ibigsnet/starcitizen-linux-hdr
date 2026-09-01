# Displays and resolution

Mixed **portrait + landscape** is common. winewayland’s primary is often the output at compositor **(0,0)** — frequently a portrait side panel — so the client only lists that panel’s modes. KWin `screen=0` also reorders after reboot. Connector names (`DP-3`, `HDMI-A-1`) change when you replug.

This is the **monitor** counterpart to [HDR](index.md) and [SteamVR](steamvr.md). Which **GPU** draws the frame is [multi-gpu.md](multi-gpu.md).

## What works

1. Among **enabled** outputs, pick the **largest landscape** panel (`width > height`, then biggest pixel area). Do not hardcode `DP-*` or `screen=0`.
2. On the **HDR / winewayland** path, set Wine’s primary to that connector:

   ```bash
   export WAYLANDDRV_PRIMARY_MONITOR="$MAIN_OUTPUT"
   ```

   Mode enumeration and which output is primary are separate from the resolution you last saved in-game.

3. Prefer **borderless** (`WindowMode=2`) over exclusive fullscreen (`1`). Exclusive binds Wine (0,0) — often the portrait panel.
4. Do **not** rewrite `Width` / `Height` to panel native every launch unless you opt in. That wipes in-game resolution.
5. Keep **`AutoDetect=0`**. `1` re-runs hardware detect and overwrites `SysSpec_*` / quality.

`HDR` and `IgnoreWindowFocus` still belong in the pre-launch patch ([attributes-hdr-patch.md](attributes-hdr-patch.md)). Do not rewrite `SysSpec_*` unless you are restoring a snapshot on purpose.

RSI Launcher does not need to live on the play panel. Only the **game client** does. Place by connector / geometry, not `screen=0`.

## Dual GPU

If **displays are on GPU A** and the game **renders on GPU B**, that is a **device filter** problem. Short version: `DXVK_FILTER_DEVICE_NAME` / `VKD3D_FILTER_DEVICE_NAME` name the **render** GPU; do **not** set `VK_ICD_FILENAMES` to that GPU only. Full write-up: **[multi-gpu.md](multi-gpu.md)**.

## Related

- [index.md](index.md) — HDR vs normal  
- [attributes-hdr-patch.md](attributes-hdr-patch.md)  
- [steamvr.md](steamvr.md)  
- [multi-gpu.md](multi-gpu.md)  
- [troubleshooting.md](troubleshooting.md)  
