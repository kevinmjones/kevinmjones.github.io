---
title: "Reverse-engineering a Qidi 3D printer to water my plants"
description: >-
  How I went from a closed-firmware Qidi X-CF Pro and a dead Windows-only
  toolchain to a Linux CLI that uploads sliced .gcode over a proprietary UDP
  protocol — and used it to print a parametric Gatorade-bottle drip spout
  that survives 5-8 days of unattended watering.
pubDate: 2026-04-30
tags: [3d-printing, reverse-engineering, openscad, python, hardware]
---

I own a [Qidi X-CF Pro](https://qidi3d.com/products/qidi-tech-x-cf-pro-industrial-grade-3d-printer) — an older industrial-grade FDM printer with a 350°C hotend, closed-source firmware, and Wi-Fi that talks only to QIDI's own desktop slicer. I wanted two things out of it that the stock workflow doesn't really offer:

1. **Drive it from Linux.** Without buying additional hardware (no Sonic Pad, no Klipper Pi).
2. **Solve a real problem with it.** Specifically: build a slow-drip plant waterer that fits on a 20 fl oz Gatorade bottle, so my plants survive while I'm out of town.

This is the story of doing both. By the end I had a parametric OpenSCAD model, a Python CLI for the printer, a Wine-mediated compression pipeline, and a small ridiculous-looking thing in soil that drips water on schedule.

## The reconnaissance

The X-CF Pro pings fine on my LAN at its local address. Let's see what services it's running:

```bash
nmap -Pn --top-ports 200 <printer-lan-ip>
```

Nothing. Every standard 3D-printer port is closed: no 80 / 443 (no web UI), no 7125 (Moonraker), no 8080 (OctoPrint), no 8899 / 3000 over TCP. Just ICMP echo replies.

That's because the X-CF Pro predates the new Klipper-based Qidi line. Older Qidi hardware uses a proprietary UDP protocol on **port 3000**, with M-code commands. A few abandoned-but-working community projects implement it — most usefully [yandreev3/3dWiFiSend](https://github.com/yandreev3/3dWiFiSend), a Python script targeting the X-Pro / X-Max with the same protocol family.

A first attempt:

```bash
echo "M4001" | nc -u <printer-lan-ip> 3000 -w2
```

I get back:

```
ok X:0.010611 Y:0.010611 Z:0.002490 E:0.007300 T:0/303/255/302/1 U:'UTF-8' B:1
```

That's the printer's calibration data — `mm-per-step` for each axis, then `T:` packs the machine type and bed dimensions (303×255×302 mm), then UTF-8 encoding. The printer is alive and chatty over UDP.

## The protocol, briefly

The X-CF Pro firmware (V4.2.14.20, with a "CBD make it" board signature — Chitu-family) uses a small set of M-codes over UDP/3000. After spending a few minutes probing with nothing more than a Python `socket`, I had this map:

| M-code | What it does |
|--------|--------------|
| `M4001` | Handshake; returns calibration and bed dimensions |
| `M4000` | **Combined status** — bed/nozzle temp & target, X/Y/Z, `D:done/total/state` progress |
| `M4002` | Firmware version |
| `M4006` | Currently or last-printed filename |
| `M20` | List files on the printer's USB drive (multi-packet response) |
| `M28 NAME` / `M29 NAME` | Begin / end file write |
| `M30 NAME` | Delete file |
| `M6030 ":NAME" I1` | Start print |
| `M114` | Live temps + heater power *(NOT position — that's `M115` here, which is non-standard)* |

A few firmware quirks that bit me:

- **Errors are signaled by the literal substring `Error:` or `fail`** in the reply. Successful `M28` and `M29` return `Done saving file!`, **not** `ok`. Detecting failures explicitly rather than checking for `ok` is the only way to avoid false positives.
- **The printer locks the session to `(IP, source-port)`** after first contact. If you open a fresh socket per request, the printer refuses subsequent commands with `Error:IP is connected by IP:X.X.X.X already!`. Reusing a single socket across the whole session fixes it.
- **Stale UDP packets in the OS receive buffer.** The printer can be slow under print load. A request that times out on our end may have its response arrive *seconds later*, sit in the kernel's UDP buffer, and get matched against the *next* request. Result: the next M4001 returns M4000-format data and your handshake parser explodes. Solution: drain the socket non-blocking before each send.

That last one took me an embarrassing amount of head-scratching. UDP, man.

## The file format problem

Listing my printer's USB drive showed prior prints with the extension `.gcode.tz`. The bottle: the X-CF Pro firmware doesn't accept raw `.gcode`. It expects a vendor-specific compressed format produced by a closed-source binary named `VC_compress_gcode`, which:

- Ships with QIDI Print (the Windows / Mac slicer)
- Takes the printer's per-axis calibration values as command-line arguments
- Bakes those values into the compressed output, so a `.tz` made for one printer might not match another

The kicker — [QIDI's own open-source repo](https://github.com/QIDITECH/Qidi-Print) at `python/qidi/Wifi/CBDConnect.py` has this code path:

```python
elif Platform.isLinux():
    exePath = "./VC_compress_gcode_linux.linux"
```

A Linux build clearly exists internally. They've never shipped it.

No public reverse-engineering of the `.tz` format exists. Every community project either runs the Windows `.exe` under Wine or uses the Mac binary unpacked from QIDI Print. So Wine it is:

```bash
sudo pacman -S wine
wine VC_compress_gcode.exe input.gcode \
  0.010611 0.010611 0.002490 0.007300\
  /tmp/out 303 255 302 0
```

The arguments after the input file are X/Y/Z/E `mm-per-step`, output dir, then `X_max` / `Y_max` / `Z_max` / `machine_type` extracted from the `M4001` handshake. The result is `input.gcode.tz` — a binary blob the printer will accept.

I emailed QIDI to ask if they'd share the Linux build. Filed under "if it ever arrives, swap to native; until then, Wine." But I haven't gotten around to actually emailing them yet.

## qidi-send

Wrapped the whole thing in a Python CLI that:

1. Opens one UDP socket and keeps it for the session.
2. Drains stale packets before every send.
3. Handshakes with `M4001` to get calibration.
4. Runs `wine VC_compress_gcode.exe` to produce `<file>.gcode.tz`.
5. Uploads via the chunked protocol — 1280-byte chunks framed with `data + LE seek-pos(4) + XOR-checksum(1) + 0x83 terminator`, sent one at a time, each ACK'd before the next.
6. Retries each chunk up to 3× on UDP timeout.
7. Optionally fires `M6030 ":NAME" I1` to start the print.

```text
$ qidi-send my_part.gcode --print

[compress] wine VC_compress_gcode.exe → /tmp/qidi-…/my_part.gcode.tz
[compress] ok, 79,104 bytes
[upload]   M28 my_part.gcode.tz (79,104 bytes)
[upload]    25%  (20,480/79,104)
[upload]   ! chunk @20480 timeout, retry 1/3
[upload]    50%  (40,704/79,104)
[upload]   100%  (79,104/79,104)
[upload]   M29
[upload]   Done saving file!
[print]    M6030 ":my_part.gcode.tz" I1
[print]    ok N:1
```

There's also `--watch` for a `top`-style live monitor, `--list` / `--delete` for managing the USB drive, and `--status` for one-shot snapshots. The whole script is around 400 lines.

## Now what to print

The X-CF Pro is back under control. Time to use it for something. The brief:

> Design and print a watering spout that fits on a 20 fl oz Gatorade bottle. Fill the bottle, screw the cap on, flip it upside-down into a plant, and have it slowly water the plant while you're away.

Three iterations later, this works.

### Iteration 1 — the threads

Gatorade's 20 fl oz bottle uses a non-standard 38mm 2-start "sport finish" — *not* a PCO 1810 / 1881 standard. There's no published spec; community measurements converge on:

- Major Ø ≈ 38 mm
- Pitch ≈ 3.6 mm per helix, 2-start (so 7.2 mm of axial travel per turn)
- Thread depth ≈ 1 mm

Modeled it in OpenSCAD using [BOSL2](https://github.com/BelfrySCAD/BOSL2)'s `threaded_rod(internal=true)`, subtracted from a cylinder to make the cap. Printed a thread-only test gauge first (~10 g, ~50 min) before committing to the full ~1.5-hour spout print.

**It didn't fit.** The bottle's threads wouldn't even *enter* the cap — the bore was too small. PETG shrinkage plus optimistic spec assumptions stacked the wrong way.

### Iteration 2 — bigger bore

Bumped `THREAD_DIAM` from 38.0 to 39.5 mm. That's 0.75 mm more radial clearance, which is generous for printer slop. Re-rendered, re-sliced, re-uploaded:

```bash
openscad -D 'target="thread_test"' -o thread_test.stl spout.scad
prusa-slicer --export-gcode --load qidi-xcf-pro-petg.ini \
  --output thread_test.gcode thread_test.stl
qidi-send thread_test.gcode --print
```

This time the bottle screwed in cleanly with a satisfying turn or two of resistance. We had a working thread.

### Iteration 3 — actual usefulness

The first complete spout was a 50 mm stake with a single 1.2 mm orifice at the tip. Worked, but had two practical flaws:

- **It tipped over.** A thin stake stuck in soil with a top-heavy water bottle on it doesn't stay put.
- **One hole isn't ideal.** If it clogs (bound to happen with tap water sediment), the whole thing stops draining.

So I added:

- **4 anchor fins** at the upper portion of the stake, 18 mm radial, triangular (wide at the cap end, tapering to a point 25 mm down). They bury into soil with the stake and lock the bottle from tilting or rotating.
- **4 distributed orifices** (1.0 mm each) along the lower stake at 50/60/70/80% of length, at 0/90/180/270° rotational angles. Total flow ≈ 2× the single 1.2 mm hole, but distributed across the soil contact zone for redundancy.
- **80 mm stake length** (was 50 mm) for deeper soil insertion.

The whole thing is parametric. The complete `spout.scad` is ~120 lines. Adjusting any dimension is a one-variable change, re-render, re-slice, ship.

```scad
// Anchor fins on the upper stake
FIN_COUNT      = 4;
FIN_RADIAL     = 18;
FIN_THICKNESS  = 2.0;
FIN_HEIGHT     = 25;

// Multiple radial orifices
ORIFICE_DIAM      = 1.0;
ORIFICE_COUNT     = 4;
ORIFICE_FRAC_LOW  = 0.50;
ORIFICE_FRAC_HIGH = 0.80;
```

## What I learned

- **UDP loss is real, even on a quiet LAN.** Anything reading a multi-packet response (file lists, status under print load) has to handle truncation and retry. Single-shot M-code request/reply is mostly fine, but anything chunked needs explicit retry logic plus a stale-packet drain.
- **Closed protocols often aren't deeply hidden.** `tcpdump` on the QIDI Print app for ten minutes would have given me the entire wire format. The open-source community projects already did the work years ago.
- **Iterate cheaply.** A 10-minute thread-test print cost 5 g of filament and saved me a 90-minute paperweight. When dimensions are uncertain, *always* print the smallest verifying piece first. This generalizes far beyond 3D printing.
- **Soil capillarity is the real flow regulator.** I was overthinking orifice sizing. Once the spout is staked in moist soil, the soil's capillary saturation determines flow rate, not the hole math. A single 1.2 mm hole and four 1.0 mm holes deliver roughly the same total water per day in practice — the multi-hole version just spreads it across more contact points.

## The artifact

The final spout:

- 39.5 mm threaded cap
- 80 mm tapered hollow stake
- 4 anchor fins (Ø49 mm anchored footprint)
- 4 distributed 1.0 mm orifices
- ~2 hours print time, ~17 g PETG

A 590 mL Gatorade bottle inverted on this delivers slow watering for roughly 5–8 days into a small-to-medium pot. My plants survive. This is, in the broadest sense, the goal.

Code, `spout.scad`, the `qidi-send` CLI, and the PrusaSlicer profile live at [github.com/kevinmjones/qidi-research](https://github.com/kevinmjones/qidi-research). The Claude skill that documents the whole pipeline (so I or any agent can remember how to drive this thing six months from now) is at `~/.claude/skills/qidi-3d-print/`.

Useful for: anyone with the same printer who wants out of QIDI's GUI; anyone reverse-engineering proprietary Wi-Fi hardware; anyone whose plants are sad while they're away.
