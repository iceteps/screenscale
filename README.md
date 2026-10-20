# screenscale

Per-monitor display scaling for **Hyprland on Omarchy 4**, for laptops that meet
a different external screen in every room.

A ~200 PPI laptop panel and a ~100 PPI desk monitor cannot share one scale: the
value that makes one comfortable makes the other unusable. Hyprland scales per
output, so the compositor is not the problem — keeping a per-output choice
*applied* on Omarchy is. `screenscale` shows the density maths, lets you tune
each screen live, and makes the choice stick.

```
$ screenscale
OUTPUT      MONITOR                     MODE              PPI  SCALE  EFF-PPI  LOGICAL
---------   -------------------------   -------------   -----  -----  -------  ----------
eDP-2       EDO EF10QBC64.C             2560x1600@165     203    1.6      127  1600x1000  *
HDMI-A-1    RIS R22L R22L12051079       1920x1080@60      102      1      102  1920x1080

  * = focused.  PPI = the panel's real pixel density.  EFF-PPI = how dense it looks.
  Two screens feel the same size when their EFF-PPI match.
```

> **Status: v0.1, and honest about it.** This works on the author's machine and
> is in daily use, but it is young, and parts of it exist purely to work around
> current Omarchy behaviour (see [Why this is a workaround](#why-this-is-a-workaround)).
> Issues and PRs very welcome — especially from anyone with a dock, three
> monitors, or a vertical screen, none of which this has been tested against.

## Install

Requires `hyprctl`, `jq`, `awk`, `socat` — all present on a stock Omarchy.

```bash
git clone https://github.com/iceteps/screenscale
install -Dm755 screenscale/screenscale ~/.local/bin/screenscale
```

Then in `~/.config/hypr/autostart.lua`:

```lua
o.launch_on_start("screenscale watch")
```

And optionally rebind Omarchy's scaling keys in `~/.config/hypr/bindings.lua`:

```lua
hl.unbind("SUPER + SLASH")
hl.unbind("SUPER + ALT + SLASH")
o.bind("SUPER + SLASH", "Monitor scale up", "screenscale up")
o.bind("SUPER + ALT + SLASH", "Monitor scale down", "screenscale down")
```

## Use

```
screenscale                    show every monitor with its PPI maths
screenscale set <scale> [MON]  apply a scale live       (default: focused)
screenscale up|down [MON]      step through presets     (default: focused)
screenscale ppi <target> [MON] scale to a target apparent density
screenscale mode <WxH[@HZ]|preferred> [MON]
                               run below the preferred resolution
screenscale save               persist current scales into monitors.lua
screenscale apply              re-apply saved scales to what is connected now
screenscale watch              daemon: re-apply on hotplug (autostart this)
screenscale forget <MON|all>   drop a saved entry
screenscale log [n]            recent actions
```

`MON` is an exact output name (`eDP-2`; an exact match wins, so `DP-1` never
also picks `eDP-1`), any substring of the name or of the monitor
description (`hdmi`, `r22l`), `focused`, or `all`.

**There is no save step.** `set` / `up` / `down` / `ppi` persist as they apply,
and `watch` adopts changes made from Omarchy's own display panel. That is not
laziness — an unsaved change cannot survive Omarchy's clamshell watcher, which
re-reads `monitors.lua` every ~2 seconds while docked and puts back whatever it
finds there.

### At a new desk

```bash
screenscale                 # what is this screen, actually?
screenscale ppi 105         # or nudge with SUPER+/ until it feels right
```

Nothing else. The scale is stored against the monitor's **EDID description**,
not its connector, so the same screen gets the same scale next time you plug it
in — a different port, a different dock, a different city.

### A TV that stutters

Many TVs advertise 4K@30 as their preferred mode, and over HDMI that is what
Hyprland picks — every animation and scroll then runs at 30 fps.

```bash
screenscale mode 1920x1080@60 sony    # smooth again
screenscale mode preferred sony       # back to what the TV asks for
```

The scale travels with the mode so the screen keeps its apparent size, as
near as a legal scale allows: 4K at scale 2 becomes 1080p at scale 1. It never
goes below 1. The mode must be one
the monitor offers (`hyprctl monitors`); `@HZ` is optional and matches within
half a hertz, so `@60` finds `59.94`. The choice is saved against the monitor's
EDID description like its scale, so it sticks to that TV. On hotplug the
watcher puts the saved mode back, and a monitor that is off its saved mode is
never adopted into the block.

Internal panels are the exception: Omarchy's clamshell watcher re-enables the
laptop panel with `mode = "preferred"`, so a mode set on `eDP-*` does not
survive closing and opening the lid.

## How it stores things

`screenscale save` maintains a marker-delimited block at the end of
`~/.config/hypr/monitors.lua`:

```lua
-- >>> screenscale: managed block — edit with `screenscale save` >>>
hl.monitor({ output = "eDP-2", mode = "preferred", position = "0x0", scale = 1.6 })
hl.monitor({ output = "desc:RIS R22L R22L12051079", mode = "preferred", position = "1600x0", scale = 1 })
-- <<< screenscale <<<
```

`mode` stays `"preferred"` unless `screenscale mode` set it; every later save
keeps the mode already recorded for that monitor rather than capturing the live
one.

Internal panels are keyed by **connector name**, on a single line. Externals are
keyed by **EDID description**. Both details are load-bearing — see below.

Positions are written out explicitly. With `position = "auto"`, changing the
laptop's scale changes its logical width and shoves every monitor after it
sideways, so retuning one screen visibly rearranged the other. The first
external monitor is the anchor and never moves; the internal panel is re-seated
flush against it on whichever side it currently sits.

## Why this is a workaround

Three Omarchy behaviours, all currently open upstream, are the reason this tool
is shaped the way it is:

| Behaviour | Upstream |
|---|---|
| `omarchy-hyprland-monitor-scaling` applies per monitor but persists into the **catch-all** rule, so a per-screen tweak becomes global on the next reload | [#7505](https://github.com/basecamp/omarchy/issues/7505), [#8103](https://github.com/basecamp/omarchy/issues/8103), [#6673](https://github.com/basecamp/omarchy/issues/6673) |
| `omarchy-hyprland-monitor-clamshell` can only read a **single-line, connector-keyed** rule, so `desc:` and multi-line rules are invisible to it and the panel is driven from the fallback every ~2s | [#7084](https://github.com/basecamp/omarchy/issues/7084), [#8123](https://github.com/basecamp/omarchy/issues/8123), [#8335](https://github.com/basecamp/omarchy/issues/8335) — fix proposed in [PR #8808](https://github.com/basecamp/omarchy/pull/8808) |
| Both paths re-apply `position = "auto"`, discarding any arrangement | [#7326](https://github.com/basecamp/omarchy/issues/7326), [#4785](https://github.com/basecamp/omarchy/issues/4785) |

A fourth is not an Omarchy bug but bites anything that tries to enforce scales:
**`hyprctl eval hl.monitor(...)` emits a `configreloaded` event.** Every scale
change does, including the user's own — so anything that "restores saved scales
on reload" will instantly undo the change the user just made. `screenscale`
reacts only to `monitoraddedv2` / `monitorremovedv2` for that reason.

If upstream lands fixes for these, most of this tool becomes unnecessary, and
that is a good outcome.

## Known issues

- Adoption of changes made from Omarchy's display panel races the clamshell
  watcher's 2-second poll. `screenscale` polls at 0.6s and usually wins, but
  "usually" is doing real work in that sentence.
- Only the first external monitor is used as the layout anchor. Three-monitor
  setups will lay out, but the arrangement logic has not been exercised.
- `GDK_SCALE` must be `1`. It is process-wide and cannot vary per output, so a
  few legacy XWayland-only apps look small on a HiDPI panel. That is an X11
  limitation, not something this can fix.
- Rotated (`transform`) outputs are preserved by Hyprland but not managed here.
- An Omarchy update that rewrites `monitors.lua` drops the managed block. The
  tell is the laptop panel snapping to scale 2 every couple of seconds; re-run
  `screenscale save`.

## Prior art

If you are not on Omarchy, you probably want one of these instead:
[way-displays](https://github.com/alex-courtis/way-displays) (DPI-based auto
scale, hotplug, persistence, blessed for Hyprland),
[nwg-displays](https://github.com/nwg-piotr/nwg-displays),
[kanshi](https://sr.ht/~emersion/kanshi), or
[HyprDynamicMonitors](https://github.com/fiffeek/hypr-dynamic-monitors). For an
Omarchy-flavoured TUI, see
[omarchy-monitor-settings](https://github.com/ryanyogan/omarchy-monitor-settings).

`screenscale` exists for the narrow case those do not cover: staying applied
against Omarchy's own monitor watchers.

## License

MIT
