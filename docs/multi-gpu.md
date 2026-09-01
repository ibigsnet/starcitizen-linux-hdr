# Multi-GPU: render on one card, display on another

Many Linux SC boxes plug the **monitor into GPU A** (often AMD, because it drives the desktop / HDR / KWin) and want the game to **render on GPU B** (often NVIDIA, for DLSS / CUDA / nvapi).

Vulkan and DXVK default to a GPU that **has a display**. If that is not the NVIDIA card, you get:

- **RADV / AMD** as the adapter  
- **No DLSS** (or nvapi features missing)  
- RSI Launcher **failing to start** if you filtered to NVIDIA but the NVIDIA ICD is not in the instance  

This is **PRIME / render offload**, not “two games at once.” HDR winewayland, SteamVR, and monitor picking are separate; this page is only **which GPU draws the frame**.

---

## Roles

| Role | Typical | Job |
|------|---------|-----|
| **Display GPU** | AMD dGPU or iGPU with DP/HDMI plugged in | KWin / winewayland present, mode list, HDR scanout |
| **Render GPU** | NVIDIA dGPU, often **no** cables | Star Citizen + DXVK/Vulkan + DLSS |
| **iGPU** | On-CPU AMD/Intel | Ignore for the game unless it is your only display |

`vulkaninfo --summary` should list **all** of them. If NVIDIA is missing, the driver/kmod is down — filters cannot invent it.

---

## What we set (conceptual)

Name the **render** GPU. Leave **all ICDs** loadable so the compositor GPU can still present.

```bash
# Render device (substring of vulkaninfo deviceName)
export DXVK_FILTER_DEVICE_NAME="NVIDIA GeForce RTX"   # or a full name: … 4090 / 5090
export VKD3D_FILTER_DEVICE_NAME="$DXVK_FILTER_DEVICE_NAME"

export WINE_HIDE_NVIDIA_GPU=0
export __NV_PRIME_RENDER_OFFLOAD=1
export __GLX_VENDOR_LIBRARY_NAME=nvidia
export __VK_LAYER_NV_optimus=NVIDIA_only
```

**Do not** set `VK_ICD_FILENAMES` to the NVIDIA JSON only. winewayland still needs the **display** GPU’s ICD (RADV/Mesa) to create a Wayland surface for KWin. Filter **after** enumeration (`DXVK_FILTER_*` / `VKD3D_FILTER_*`), don’t delete the present ICD.

Optional: `SC_NVIDIA_RENDER=0` (or skip the block) if you actually want to render on the display GPU.

---

## Mental model

```text
Star Citizen / DXVK / native Vulkan
        │  DXVK_FILTER_DEVICE_NAME = NVIDIA …
        ▼
   Render GPU (NVIDIA)  ── shaders, DLSS, framebuffer
        │
        │  PRIME copy (PCIe)
        ▼
   Display GPU (AMD / iGPU)  ── KWin, winewayland, HDR scanout
        │
        ▼
   Physical panel (largest landscape — see displays-resolutions.md)
```

Every frame is copied over PCIe to the card that owns the connector. That is expected. It is **not** the same as “the NVIDIA card is idle.”

---

## Why the launcher died when NVIDIA was missing

RSI Launcher is **DXVK** (D3D11 → Vulkan). If `DXVK_FILTER_DEVICE_NAME` is `NVIDIA GeForce RTX 4090` and `vulkaninfo` only shows RADV + llvmpipe, DXVK logs:

```text
Found device: AMD Radeon …   Skipping: Device filter
DXVK: No adapters found
Failed to initialize DXVK
```

and the launcher never appears. That is a **filter + missing kmod** problem, not HDR.

Fail **closed** if you require NVIDIA: check `nvidia-smi` before Wine. After a kernel update, DKMS/`kernel-devel` must match or the module will not load.

The **game client** on current LIVE is **native Vulkan** as well as DXVK for the launcher; both need the NVIDIA physical device visible.

---

## DLSS / nvapi (Linux)

Star Citizen’s Vulkan renderer can still use NGX via **dxvk-nvapi**-style DRS overrides (LUG NVIDIA notes). Typical:

```bash
export DXVK_ENABLE_NVAPI=1
export PROTON_ENABLE_NGX_UPDATER=1
# Super-res / ray reconstruction on; driver frame-gen override off
# (CIG: Smooth Motion / FG overrides → black screens / odd pacing)
export DXVK_NVAPI_DRS_NGX_DLSS_SR_OVERRIDE=on
export DXVK_NVAPI_DRS_NGX_DLSS_RR_OVERRIDE=on
export DXVK_NVAPI_DRS_NGX_DLSS_FG_OVERRIDE=off
```

These only matter if the **render** adapter is NVIDIA. On RADV they do nothing useful.

Confirm in `Game.log`: `Chosen Vulkan GPU Device (NVIDIA …)` and `DLSS initialized` — not only “DLSS Support = Available.”

---

## Overlay GPU order

MangoHud 0.8 indexes GPUs by sorted DRM `renderD*` nodes, which **shuffle** when NVIDIA binds late. Label strings (`gpu_text`) must follow a **stable order** (e.g. NVIDIA dGPU → other dGPU → iGPU), with `gpu_list` remapped to those indices at session start. See your overlay docs; do not assume index `0` is NVIDIA.

---

## Checklist

1. `vulkaninfo --summary` lists the **render** GPU **and** the **display** GPU.  
2. `DXVK_FILTER_DEVICE_NAME` / `VKD3D_FILTER_DEVICE_NAME` match the **render** `deviceName`.  
3. `VK_ICD_FILENAMES` is **not** NVIDIA-only.  
4. `WINE_HIDE_NVIDIA_GPU=0`.  
5. HDR path still uses winewayland + display ICD; filters only pick **who draws**.  
6. `Game.log`: NVIDIA chosen; DLSS init if you expect it.  
7. Launcher: if it fails with “No adapters found,” NVIDIA kmod/filter first.

---

## Related

- [index.md](index.md) — HDR vs normal  
- [displays-resolutions.md](displays-resolutions.md) — which **monitor**, not which GPU  
- [steamvr.md](steamvr.md) — Steam hide vs VR  
- [troubleshooting.md](troubleshooting.md)  
- [LUG NVIDIA](https://wiki.starcitizen-lug.org/Troubleshooting/nvidia)  
