# Forcing HDR in `attributes.xml` *before* the game launches

## Why patch before launch?

Star Citizen stores many graphics flags in the profile file:

```text
<WinePrefix>/drive_c/Program Files/Roberts Space Industries/StarCitizen/
  <LIVE|PTU|EPTU|TECH-PREVIEW|…>/user/Client/0/Profiles/default/attributes.xml
```

Important behaviors we hit:

1. On the **normal / X11** path, the in-game **HDR** checkbox is not enough (and toggling it can freeze). The presentation stack is wrong even if the box is ticked.
2. On a **proper HDR launch** (winewayland, compositor HDR on, `DXVK_HDR=1`), that same **enable/disable** control works fine mid-session to switch between HDR and SDR modes. Pre-patching is about boot defaults and path hygiene, not “never use the menu.”
3. The file is read when the **client** starts. Changing attributes on disk after the game is already up often does nothing until restart.
4. If you last used **HDR launch** (`HDR=1`) and next time you use **normal/SDR launch** without clearing the flag, SDR sessions can misbehave. So **both** paths should rewrite the file deliberately.

Therefore the launcher script **edits `attributes.xml` on disk before** RSI Launcher starts the game.

## What we force

### HDR launch (`SC_HDR=1`, e.g. `sc-launch-hdr.sh`)

| Attribute | Forced value | Purpose |
|-----------|--------------|---------|
| `HDR` | `1` | Game-side HDR enabled at boot |
| `Width` / `Height` | **Optional** — only if you opt in | Forcing panel native every boot **wipes** in-game resolution; see [displays-resolutions.md](displays-resolutions.md) |
| `WindowMode` | If `1`, coerce to `2` on HDR | Exclusive FS binds Wine (0,0) (often portrait) |
| `AutoDetect` | `0` | `1` re-detects and overwrites quality on next start |
| `IgnoreWindowFocus` | `0` (default) | Don’t keep mouse-look when alt-tabbed |

Also set in the environment (not in XML):

```bash
export DXVK_HDR=1
unset DISPLAY   # winewayland — see main guide
```

### Normal / SDR launch (`SC_HDR` unset or `0`)

| Attribute | Forced value | Purpose |
|-----------|--------------|---------|
| `HDR` | `0` | Clear leftover HDR session flag |
| `IgnoreWindowFocus` | `0` | Same alt-tab mouse fix |

`DXVK_HDR` may still be exported in a shared script; without winewayland + compositor HDR it won’t give a correct HDR path.

## How the patch works (conceptually)

```text
start sc-launch-hdr.sh
        │
        ▼
  SC_HDR=1
        │
        ▼
  for each channel LIVE, PTU, EPTU, …:
        edit Profiles/default/attributes.xml
           HDR=1, optional Width/Height, IgnoreWindowFocus=0
        │
        ▼
  unset DISPLAY, DXVK_HDR=1, …
        │
        ▼
  start RSI Launcher.exe
        │
        ▼
  user clicks Launch → client already sees HDR=1
```

Pseudo-logic:

```bash
sc_patch_attributes() {
  local file="$1"
  [[ -f "$file" ]] || return 0
  # backup once
  cp -n "$file" "$file.hdr-bak" 2>/dev/null || true

  if [[ "${SC_HDR:-0}" == "1" ]]; then
    # set Width/Height if you pin resolution
    sed -i -E 's/(name="HDR" value=")[^"]*/\11/' "$file"
    # or insert <Attr name="HDR" value="1"/> if missing
  else
    sed -i -E 's/(name="HDR" value=")[^"]*/\10/' "$file"
  fi

  # Always prefer focused-window input only
  sed -i -E 's/(name="IgnoreWindowFocus" value=")[^"]*/\10/' "$file"
}
```

Apply to **every channel folder you use** (LIVE and PTU often have separate trees). On some Windows installs, PTU may symlink to EPTU—patch the real directory.

## Optional environment overrides

| Env | Meaning |
|-----|---------|
| `SC_ATTR_WIDTH` / `SC_ATTR_HEIGHT` | Resolution written on HDR launch (default 3840×2160 for us) |
| `SC_IGNORE_WINDOW_FOCUS` | Default `0`; set `1` only if you want background mouse input |
| `SC_HDR` | `1` = HDR attribute + winewayland path |

## What this does *not* replace

- **Compositor HDR** must still be enabled on the monitor.  
- **`unset DISPLAY`** (winewayland) is still required for real HDR presentation.  
- Patching XML alone while staying on XWayland is **not** the full fix.

## Safety tips for users copying the idea

1. **Backup** `attributes.xml` before first automation (`attributes.xml.hdr-bak`).  
2. Don’t run two launchers that fight over the same file simultaneously.  
3. After a game patch, re-check that the attributes keys still exist (CIG can rename options).  
4. Peak brightness (`HDRMaxBrightness`, `HDRRefWhite`) can stay user-tuned in-game; we mainly force the on/off flag and resolution pin.

## Example resulting fragment

```xml
<Attr name="HDR" value="1"/>
<Attr name="HDRMaxBrightness" value="1000"/>
<Attr name="HDRRefWhite" value="250"/>
<Attr name="Width" value="3840"/>
<Attr name="Height" value="2160"/>
<Attr name="IgnoreWindowFocus" value="0"/>
<Attr name="WindowMode" value="1"/>
```

---

See also: [index.md](index.md), [normal-vs-hdr.md](normal-vs-hdr.md).
