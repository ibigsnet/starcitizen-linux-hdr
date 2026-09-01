# SteamVR / OpenVR: how the game is allowed to see Steam

Star Citizen on Linux is usually launched from a **Wine prefix + RSI Launcher**, not from Steam. The launcher (and the client) will still **probe Steam** if they can see a Steam install: `steamclient`, OpenVR, and **SteamVR’s `vrserver`**.

That probe is useful when you **want** VR. It is harmful when you do **not**: RSI Launcher can load `steamclient` and auto-start `vrserver`, which has crashed the launcher (Wine “Web Thread” / `select()` style faults) on Plasma Wayland.

So the **default** launch path **hides Steam**. VR is an **opt-in** third path next to **normal** and **HDR**.

This is **not** an official CIG, Valve, or LUG guide. Adapt paths to your prefix.

---

## Three paths (where VR sits)

| Path | Env | Steam install | `DISPLAY` | Intent |
|------|-----|---------------|-----------|--------|
| **Normal** | (unset) | **Hidden** | **Kept** (Wine X11 / XWayland) | Flat play, stable keyboard |
| **HDR** | `SC_HDR=1` | **Hidden** | **Unset** → winewayland | Real HDR |
| **SteamVR** | `SC_STEAMVR=1` | **Visible** | **Kept** (vrserver wants X11) | OpenVR / SteamVR |

**HDR and SteamVR fight over `DISPLAY`.** VR mode wins if both flags are set: keep `DISPLAY`, skip the winewayland HDR trick. Do not expect “true HDR winewayland + SteamVR” in one process tree without extra work.

---

## Why default launch hides Steam

RSI Launcher is a Windows Electron app. If `~/.local/share/Steam` (or your Steam library path) is visible to Wine, it can:

1. Load **`steamclient`**.
2. Start **`vrserver`** even though you only wanted 2D.
3. Crash the launcher (Web Thread / compositor interaction) on some NVIDIA + Plasma Wayland setups.

The workaround is a **bind mount that replaces Steam’s directory with an empty tmpfs** for the Wine tree only. Host Steam is untouched.

Conceptual (bubblewrap or equivalent):

```bash
# Default / non-VR: Wine must not see Steam
bwrap \
  --dev-bind / / \
  --tmpfs "$HOME/.local/share/Steam" \
  -- \
  wine "C:\\Program Files\\Roberts Space Industries\\RSI Launcher\\RSI Launcher.exe"
```

Any equivalent namespace (`unshare`, Firejail, a private bind) is fine. The requirement is: **the Wine process cannot stat a real Steam install**.

---

## What VR actually needs

When you **do** want the headset:

| Need | Why |
|------|-----|
| **Real Steam directory** | OpenVR runtime, `steamclient`, SteamVR binaries |
| **SteamVR installed and able to start** | Headset compositor (`vrserver`) |
| **`DISPLAY` set** | `vrserver` / SteamVR still expect X11 on typical desktop Linux, even inside a Wayland session |
| **No Steam tmpfs hide** | Otherwise OpenVR init fails (“could not open registry”, “Failed to locate module” — those lines in a *non-VR* log are **expected**) |

The game talking to OpenVR is then the same as on Windows: enable VR in Star Citizen after SteamVR is up. Some CIG/EAC bootstrap lines still include `-no-vr`; treat in-game VR enable + a SteamVR-visible prefix as the hook, not a hand-edit of anti-cheat command lines.

---

## Opt-in: `SC_STEAMVR=1`

Shared launcher (same script as HDR) should branch **before** Wine starts:

```bash
# Display: SteamVR needs X11. Do this *before* any HDR unset DISPLAY.
if [[ "${SC_STEAMVR:-0}" == "1" ]]; then
  export DISPLAY="${DISPLAY:-:0}"
elif [[ "${SC_HDR:-0}" == "1" ]]; then
  unset DISPLAY   # winewayland HDR path — skip when using VR
fi

# Steam visibility
if [[ "${SC_STEAMVR:-0}" == "1" ]]; then
  STEAM_BLOCK=()          # no bwrap hide
else
  STEAM_BLOCK=(bwrap --dev-bind / / --tmpfs "$HOME/.local/share/Steam" --)
fi

"${STEAM_BLOCK[@]}" wine ".../RSI Launcher.exe"
```

Wrapper / `.desktop` (generalized):

```bash
#!/usr/bin/env bash
export SC_STEAMVR=1
# Do not set SC_HDR=1 here; VR keeps DISPLAY.
exec ./sc-launch.sh "$@"
```

```ini
[Desktop Entry]
Name=RSI Launcher (SteamVR)
Comment=Star Citizen with Steam/OpenVR visible; DISPLAY kept for vrserver
Exec=env SC_STEAMVR=1 /path/to/sc-launch.sh
Categories=Game;VR;
```

Start **SteamVR first** (or have it able to auto-start), then this launcher, then enable VR in the client.

---

## Mental model

```text
Default / HDR (no headset)
  Wine prefix
       │
       ▼
  tmpfs over ~/.local/share/Steam
       │
       └── RSI Launcher cannot load steamclient → no vrserver surprise

SteamVR path
  SteamVR (host) ── vrserver needs DISPLAY (X11)
       │
  Wine prefix ── sees real Steam + OpenVR
       │
       └── Star Citizen can initialize VR
```

---

## Tradeoffs

| | Default hide | SteamVR visible |
|--|----------------|-----------------|
| Launcher stability | Better (no auto vrserver) | You must tolerate SteamVR + steamclient in the Wine tree |
| Accidental headset init | Blocked | Possible |
| HDR winewayland | Compatible (Steam still hidden) | **Not** combined: `DISPLAY` stays set |
| Keyboard / X11 | Normal path | Same as normal (X11), not HDR winewayland |

Wine **System Tray** hide helpers that walk windows can confuse SteamVR overlays; it is reasonable to **skip tray hiding** on the VR path.

---

## Checklist

1. SteamVR runs on the host and the headset is seen there.  
2. Launch with **`SC_STEAMVR=1`** (or equivalent: **no** Steam tmpfs, **`DISPLAY` kept**).  
3. Confirm the Wine process **can** list `$HOME/.local/share/Steam` (or your Steam library).  
4. Confirm **`DISPLAY` is set** on that process (`tr '\\0' '\\n' < /proc/<pid>/environ | grep DISPLAY`).  
5. Do **not** use the HDR wrapper (`unset DISPLAY`) for the same session.  
6. Enable VR **in game** after the client is up.

If OpenVR still fails: SteamVR runtime bitness, `VR_OVERRIDE` / `XR_RUNTIME_JSON`, and whether Wine can execute the host SteamVR binaries are the next knobs — those are Steam/Wine issues, not Star Citizen–specific.

---

## Related

- [index.md](index.md) — HDR vs normal  
- [normal-vs-hdr.md](normal-vs-hdr.md) — why `DISPLAY` matters  
- [troubleshooting.md](troubleshooting.md) — HDR, tray, monitors  
