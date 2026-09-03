# BCMCH Tremor Monitor v5.1.1 — readability + backlight + CSV download fix

Diff on top of v5.1.0 (tagged `R1`–`R3` in the `.ino`, alongside the
existing `T1`–`T10` tags from the previous update — `grep -n "R[0-9]"` or
`grep -n "T[0-9]"` to find them).

## 1. LCD readability (R2, R3)

**Before**: a bare number ("23.4") on the panel — precise, but not
glanceable, especially for a patient checking their own wrist mid-tremor.

**Now**: the panel leads with a large, plain-language, color-coded status
word:

| Word | Color | Range (deg/s, tunable) |
|---|---|---|
| CALM | green | < 25 |
| MILD | yellow-green | 25–60 |
| MODERATE | orange | 60–100 |
| STRONG | red | > 100 |

The precise number ("23.4 deg/s") is still shown, smaller, underneath —
so nothing clinically useful is lost, it's just no longer the primary
thing you have to read. "Min"/"Max" became "Lowest"/"Highest" (plainer
English), each on more generously spaced text.

**Important**: those four thresholds are a reasonable starting point, not
a clinically validated scale. Tune `TREMOR_BAND_MILD_DPS` /
`_MODERATE_DPS` / `_STRONG_DPS` near the top of the DSP section against
real recorded sessions before relying on the color/word for anything
beyond a rough at-a-glance indicator — the same caution that applied to
the raw severity number in the first place.

**On font size**: the status word opportunistically uses the largest
Montserrat size actually compiled into your `lv_conf.h` (48→40→32→28,
checked via LVGL's own `LV_FONT_MONTSERRAT_*` feature-flag macros), and
falls back to the default theme font if none of those are enabled — so
this **cannot fail to compile** regardless of your `lv_conf.h`, but you
may not get a dramatically bigger word if your project only has the
default font size built in. If the status word looks the same size as
before after flashing, check `lv_conf.h` for `LV_FONT_MONTSERRAT_32` (or
similar) set to `1`, and rebuild.

## 2. Backlight timing (R1)

`BACKLIGHT_TOUCH_ON_MS` (on-time after each touch) changed from **3000
→ 15000**. Worth noting: your original description said "5 seconds,"
but the actual firmware constant was 3000ms — I flagged this discrepancy
back when I first reviewed the sketch. I also bumped
`BACKLIGHT_BOOT_ON_MS` (initial on-time after boot) from 5000 → 15000 to
match, so the behavior is consistent whether the screen just turned on
from boot or from a touch. Change either constant independently if you'd
rather they differ.

## 3. CSV download fix

I found (and fixed) two real bugs in `index.html`, plus added
diagnostics so if it's *still* broken after this, the Log panel will
tell us exactly where:

**Bug 1 — concurrent GATT operations.** The page polls the `STATUS`
characteristic every 3 seconds in the background. Web Bluetooth throws
an error if two read/write operations on the same device overlap in
flight — so if that background poll happened to fire at the same moment
as the download's own write+read sequence, one of them could fail
outright. Fixed by routing **every** GATT read/write through a single
serializing queue (`gattOp()`), so nothing ever overlaps, and by having
the status poll skip itself entirely while a download is active.

**Bug 2 — unsafe buffer slicing.** The notification handlers did
`new Uint8Array(event.target.value.buffer)`, which takes the *entire*
underlying `ArrayBuffer` rather than just the bytes belonging to that
specific notification (`DataView.buffer` isn't automatically clipped to
`byteOffset`/`byteLength`). In practice this usually happens to work
because the browser typically allocates a fresh, exactly-sized buffer
per notification — but it's not guaranteed, and any case where it doesn't
would corrupt the downloaded CSV silently. Fixed to always respect
`byteOffset`/`byteLength` explicitly.

**Also added**: a 6-second stall watchdog (if no chunk arrives for 6s
mid-download, it gives up cleanly with a clear log message instead of
leaving the UI stuck with a disabled button forever), and much more
detailed logging at every step of the download (request sent, byte count
reported by device, each stall/failure reason) so a future failure is
diagnosable from the on-page Log panel alone.

## If it's still not working after this

Please try again and paste **both**:
1. The Log panel contents from the page (Connect → Start → Stop →
   Download, whatever sequence you tried)
2. The Serial monitor output from the device over the same window,
   particularly any `[BLE][XFER]` lines

That'll tell us definitively whether it's failing on the write, the info
read, or partway through the chunk transfer — and whether the problem is
even still in the software at all versus, e.g., the device being out of
BLE range or the CSV buffer genuinely being empty (check the "CSV
buffer" line in the device-info panel before downloading — if it says
"0 bytes / 0 rows," record something first).

## Files
- `W146_v5_1_0_tremor.ino` — v5.1.0 firmware + the R1–R3 diff (readability, backlight)
- `index.html` — v5.1.0 web page + the CSV download fixes
