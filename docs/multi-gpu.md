# Multi-GPU: render on one card, display on another

If the **monitor is on GPU A** (often AMD — desktop / HDR / KWin) and you want the **game on GPU B** (often NVIDIA — DLSS), Vulkan and DXVK still default to the GPU that **has a display**. You then get RADV, no DLSS, and a launcher that never starts if you filtered to NVIDIA but that ICD is missing.

This is **PRIME / render offload**. HDR winewayland and SteamVR are separate; this page is only **which GPU draws the frame**.

## Env that works

Name the **render** GPU. Leave **all ICDs** loadable so the compositor GPU can still present.

```bash
# Substring of vulkaninfo deviceName (or a full name: … 4090 / 5090)
export DXVK_FILTER_DEVICE_NAME="NVIDIA GeForce RTX"
export VKD3D_FILTER_DEVICE_NAME="$DXVK_FILTER_DEVICE_NAME"

export WINE_HIDE_NVIDIA_GPU=0
export __NV_PRIME_RENDER_OFFLOAD=1
export __GLX_VENDOR_LIBRARY_NAME=nvidia
export __VK_LAYER_NV_optimus=NVIDIA_only
```

**Do not** set `VK_ICD_FILENAMES` to the NVIDIA JSON only. winewayland still needs the **display** GPU’s ICD (RADV/Mesa) to create a Wayland surface for KWin. Filter **after** enumeration (`DXVK_FILTER_*` / `VKD3D_FILTER_*`); do not delete the present ICD.

`vulkaninfo --summary` should list **both** cards. Filters cannot invent a missing NVIDIA kmod.

## Launcher “No adapters found”

RSI Launcher is **DXVK**. If the filter is NVIDIA and `vulkaninfo` only shows RADV, DXVK logs `No adapters found` and the window never appears. Load the NVIDIA driver first (`nvidia-smi` works), then launch. After a kernel update, DKMS / `kernel-devel` must match or the module will not load.

## DLSS

Only if the **render** adapter is NVIDIA. Typical dxvk-nvapi DRS (LUG NVIDIA notes):

```bash
export DXVK_ENABLE_NVAPI=1
export PROTON_ENABLE_NGX_UPDATER=1
export DXVK_NVAPI_DRS_NGX_DLSS_SR_OVERRIDE=on
export DXVK_NVAPI_DRS_NGX_DLSS_RR_OVERRIDE=on
# CIG: Smooth Motion / FG overrides → black screens / odd pacing
export DXVK_NVAPI_DRS_NGX_DLSS_FG_OVERRIDE=off
```

Confirm in `Game.log`: `Chosen Vulkan GPU Device (NVIDIA …)` and `DLSS initialized` — not only “DLSS Support = Available.”

## Checklist

1. `vulkaninfo --summary` lists the **render** GPU **and** the **display** GPU.  
2. `DXVK_FILTER_DEVICE_NAME` / `VKD3D_FILTER_DEVICE_NAME` match the **render** `deviceName`.  
3. `VK_ICD_FILENAMES` is **not** NVIDIA-only.  
4. `WINE_HIDE_NVIDIA_GPU=0`.  
5. `Game.log`: NVIDIA chosen; DLSS init if you expect it.  
6. Launcher “No adapters found” → NVIDIA kmod / filter first.

## Related

- [index.md](index.md) — HDR vs normal  
- [displays-resolutions.md](displays-resolutions.md) — which **monitor**, not which GPU  
- [steamvr.md](steamvr.md)  
- [troubleshooting.md](troubleshooting.md)  
- [LUG NVIDIA](https://wiki.starcitizen-lug.org/Troubleshooting/nvidia)  
