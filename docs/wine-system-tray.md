# Hiding the Wine System Tray (taskbar icon / floating tray)

## What users see

When you run **RSI Launcher** (Electron) under Wine—especially on the **HDR / winewayland** path—Windows-style tray plumbing often shows up as:

- A floating window titled **“Wine System Tray”**
- An extra **taskbar / panel icon**
- Sometimes a tiny opaque rectangle that steals focus or looks like a second “app”

That is **not** the RSI Launcher window itself. It is Wine’s tray host (`explorer.exe` tray / notification area bridge).

## Why we hide it (and what we must *not* do)

| Approach | Result |
|----------|--------|
| **Close / kill the tray window** | Can destroy HWNDs Electron still needs → **RSI Launcher FATAL / instant exit**. Also risky if **exit Star Citizen / RSI Launcher on close** is enabled in launcher settings — a close can take down the whole session, not only the tray UI. |
| **Kill `explorer.exe` in the prefix** | Same class of breakage |
| **Hide only (minimize, skip taskbar, opacity 0)** | Tray process stays alive; UI clutter gone; launcher (and game) stay stable |

So the workaround is: **hide, never destroy**.

## How we hide it

### 1. Permanent KWin window rule (Plasma)

Match title **Wine System Tray** and force:

- Minimize  
- Skip taskbar / pager / switcher  
- No border  
- Below other windows  
- Opacity 0 (or near 0)  
- Do **not** “close window” on match  

Example rule ideas (KDE *System Settings → Window Management → Window Rules*):

- Window title: exact / contains `Wine System Tray`
- Force: minimized, skip taskbar, skip pager, no titlebar, low opacity  

On our machine this lives in `~/.config/kwinrulesrc` under a rule described as *Hide Wine System Tray (keep process/HWND — never close)*.

### 2. Delayed hide helper after launch

The tray often appears **a few seconds after** Electron starts—not at t=0. The launch script schedules several delayed passes, e.g. at 2s, 5s, 10s, 20s, 35s:

```bash
# Conceptual — do not close the window
for delay in 2 5 10 20 35; do
  sleep "$delay"
  hide-wine-tray-safe.sh   # minimize / re-apply KWin hide only
done &
```

Helper responsibilities:

- Ask KWin to reconfigure / re-run a small “hide tray” script  
- **Never** `wmctrl -c` / windowclose on that title  
- Exit quietly if KWin APIs are missing  

Optional env to skip entirely:

```bash
SUPPRESS_RSI_TRAY=0 ./sc-launch-hdr.sh   # leave tray visible
```

Default for us: **suppress on** (hide).

### 3. Why this shows up more on HDR

With **`DISPLAY` unset** (winewayland), tray / desktop integration can surface differently than under XWayland. Users on the HDR path often notice the tray more; the same hide rule still applies.

## User-facing FAQ

**Q: Is it safe to click the X on “Wine System Tray”?**  
**A:** Avoid it. Closing that window has crashed RSI Launcher for us (HWND teardown). If you also have **exit Star Citizen / RSI Launcher on close** enabled in RSI Launcher settings, a close can exit the launcher and game entirely—not just the tray. Use the hide rule instead.

**Q: Does hiding the tray disable RSI notifications?**  
**A:** Tray UI is suppressed; the launcher process remains. You still use the main RSI Launcher window.

**Q: Does this affect the game client?**  
**A:** No. It only targets the Wine tray window title/class, not `StarCitizen.exe`.

---

See also: [index.md](index.md) (full HDR checklist), [troubleshooting.md](troubleshooting.md).
