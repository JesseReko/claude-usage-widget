# Claude Usage Widget — Claude Context

## What this is

A Windows 11 always-on-top floating widget that shows Claude.ai usage stats (session tokens, weekly tokens, API credits). Written in Python, distributed as a PyInstaller exe. Forked from an open-source widget and maintained under `JesseReko/claude-usage-widget` on GitHub (public repo). Upstream: `niccolo-sabato/claude-usage-widget`.

Built/customized June 2026.

---

## Stack

- **Language:** Python, single file `src/widget.pyw`
- **UI:** Tkinter with `overrideredirect(True)` — no OS window chrome
- **Packaging:** PyInstaller → `ClaudeUsage-Setup.exe`
- **Config:** `%LOCALAPPDATA%\Claude Usage\config.json`
- **Logs:** `%LOCALAPPDATA%\Claude Usage\widget.log` and `crash.log`
- **Auto-update:** checks `JesseReko/claude-usage-widget` GitHub releases API

---

## Coordinate system — critical

This system runs Windows 11 with a 4K display at 150% DPI scaling. `SetProcessDpiAwareness(2)` is set, which means:

- **Tkinter uses PHYSICAL pixel coordinates** — same unit as `SHAppBarMessage`
- `dpi_scale = 1.501` exists in the code for **UI element sizing only** (font sizes, padding, widget dimensions)
- **Never apply dpi_scale to window coordinates or taskbar position values**
- `SHAppBarMessage(ABM_GETTASKBARPOS)` returns physical pixels: taskbar at `y=2088`, height=72px
- Pass those values directly to Tkinter geometry — no conversion

This burned us once: dividing SHAppBarMessage values by dpi_scale moved the widget to logical coordinates (~y=1391) and it "disappeared" off the visible screen area.

---

## Modes

| Mode | Description |
|------|-------------|
| Normal/expanded | Full widget with title bar, section names, all bars |
| Essential | Compact bar-only view, no title bar — triggered by double-clicking the ochre dot |
| Collapsed (essential + docked) | Bars only, transparent background blends into taskbar |
| Docked | Snapped to the taskbar via `SHAppBarMessage` |

State flags: `_docked`, `_essential`, `_expanded`

---

## Key layout constants

```python
DEF_W   = 160    # default width
MIN_W   = 120    # minimum width
BAR_H   = 16     # progress bar height
TITLE_H = 28     # title bar height
BG      = '#262624'  # background — also used as -transparentcolor when collapsed
```

---

## Transparency / click-through

- `-transparentcolor BG` makes exact BG-colored pixels invisible AND click-through
- Only applied in `_enter_ess_collapsed()`, removed in `_exit_ess_collapsed()`
- Side effect: frame backgrounds matching BG exactly are also click-through — dragging only works by clicking on bar canvas pixels (the filled pill areas), not the background
- Don't try to make "invisible" frames with off-by-one colors — the rendering artifacts are visible and awful

---

## Known quirks

- `_auto_height()` is guarded with `if self._docked: return` — without this, any layout change (bar count change, language toggle, etc.) fires `_auto_height()` and resets height to content height (~49px), overriding the 72px taskbar-snapped height
- Hamburger menu was missing on startup in docked+essential mode because `_enter_ess_collapsed()` ran at `after(100)` when `_docked=False`. Fixed in `_restore_docked()` by adding `elif not self._expanded: self._enter_ess_collapsed()`
- Pace suffix is intentionally short (`"65% session"` / `"65% week"`) — longer text forced minimum widget width by expanding the sub-label

---

## Gradient bars

`pill_gradient()` function draws progress bars as top-to-bottom linear gradients using `_lerp_hex()`. Both the background track and the filled portion use gradients. The function iterates row by row, clipping to the pill shape geometry.

---

## Update check

Points to `JesseReko/claude-usage-widget` GitHub releases. Checks on startup after a 10s delay, then every 24h. The repo has no releases, so the check logs a 404. That is harmless. The widget runs from source via the Startup shortcut, not the installer.

## Credentials

- `session_key` only accepts a real `sk-ant-` value from a `Set-Cookie` header (`rotated_session_key`). The server ends sessions with `sessionKey=""`, which used to overwrite a working key.
- All curl calls go through `run_curl`, because a default timeout message contains the full command, credentials included. `wlog` also redacts `sk-ant-` values.
- Expired session while docked or in essential mode: `session_banner` covers the strip and opens the renew dialog on click, because the error panel falls outside the locked docked height.
- The credits API call only runs with an `sk-ant-admin` key (`_has_admin_key`).

---

## Config keys (config.json)

| Key | Purpose |
|-----|---------|
| `session_key` | Claude.ai session cookie |
| `org_id` | Claude.ai org ID |
| `admin_api_key` | Anthropic Admin API key, `sk-ant-admin…` (for API spend) |
| `taskbar_docked` | Whether widget is docked to taskbar |
| `dock_x` | X position when docked |
| `essential` | Whether essential mode is active |
| `essential_bars` | Which bars show in essential mode |
| `width` / `height` / `x` / `y` | Last window geometry |

---

## Repo

`C:\Users\Dell XPS 1530\Apps\claude-usage-widget\`
GitHub: `https://github.com/JesseReko/claude-usage-widget` (public)
