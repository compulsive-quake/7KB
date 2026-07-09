# Reading HID Joysticks (hid.dll; formerly winmm)

7DTD uses Unity's **legacy** Input Manager, so `Input.GetAxis` only sees axes
baked into the shipped `InputManager.asset` — a mod cannot add axes. To read a
DirectInput/HID joystick (e.g. a RadioMaster transmitter in "USB Joystick
(HID)" mode), bypass Unity and read the device directly. Working
implementation: `FPV/src/FPVGamepad.cs` (direct hid.dll, all 8 axes).

## Current approach: setupapi + hid.dll + overlapped ReadFile

Parse HID input reports yourself; winmm is a 6-axis dead end (see below).
Recipe, all plain P/Invoke with no extra DLLs shipped:

1. **Enumerate**: `HidD_GetHidGuid` → `SetupDiGetClassDevs(DIGCF_PRESENT |
   DIGCF_DEVICEINTERFACE)` → `SetupDiEnumDeviceInterfaces` →
   `SetupDiGetDeviceInterfaceDetail` for the device path. The detail struct's
   `cbSize` is the *fixed* part only: **8 on x64** (the game is 64-bit only).
2. **Filter**: `CreateFile(GENERIC_READ, share RW, FILE_FLAG_OVERLAPPED)` —
   keyboards/mice refuse GENERIC_READ, which filters them for free. Then
   `HidD_GetPreparsedData` + `HidP_GetCaps`; keep devices with
   `UsagePage 0x01` and `Usage` 0x04 (joystick) / 0x05 (gamepad) / 0x08
   (multi-axis). If several qualify, prefer the one with the most axes.
3. **Friendly name**: `HidD_GetProductString` returns the same name the Game
   Controllers dialog shows — this **replaces** the winmm-era registry
   OEMName crawl entirely.
4. **Axes**: `HidP_GetValueCaps` (mind `IsRange`) → the 8 analog usages are
   `0x30..0x37` = X, Y, Z, Rx, Ry, Rz, Slider, Dial. Record
   `LogicalMin/Max/BitSize` per usage; if `LogicalMin < 0` the raw value from
   `HidP_GetUsageValue` is two's-complement in `BitSize` bits — sign-extend
   before normalizing.
5. **Read loop** (no background thread needed): keep one overlapped `ReadFile`
   pending; each frame drain completed reports via
   `GetOverlappedResult(wait:false)` until `ERROR_IO_INCOMPLETE`, re-issuing
   the read after each report. Allocate the OVERLAPPED and report buffer in
   unmanaged memory (`Marshal.AllocHGlobal`); on x64 `hEvent` sits at
   OVERLAPPED offset 24. Any other read error = device unplugged.
6. **Parse**: `HidP_GetUsageValue` per axis usage and `HidP_GetUsages`
   (usage page 0x09; button usages are 1-based) per report.
   `HIDP_STATUS_SUCCESS` is `0x00110000`; a non-success status usually means
   the usage lives in a different report ID — keep the previous value, don't
   zero it. Report buffer length passed to `HidP_*` is
   `Caps.InputReportByteLength` (includes the report-ID byte).
7. **Teardown**: if a read is pending, `CancelIo` + `GetOverlappedResult(wait:
   true)` *before* freeing the buffers, or the driver can write into freed
   memory.

Channel order still varies per radio/firmware, so bind axes by detection
("move the stick"), not a fixed mapping. Slots the device doesn't report
should read as centered (0.5) so a stale bind can't peg a channel.

## Why not winmm (`joyGetPosEx`): it silently drops 2 of 8 axes

winmm (`joyGetNumDevs` / `joyGetPosEx` / `joyGetDevCapsW`) has only six axis
slots and the HID→slot mapping is not what the field names suggest. Measured
with EdgeTX USB joystick "classic" mode (CH1-8 → HID X, Y, Z, Rx, Ry, Rz,
Slider, Dial):

| EdgeTX channel | HID axis | JOYINFOEX field |
|---|---|---|
| CH1 | X | dwXpos |
| CH2 | Y | dwYpos |
| CH3 | Z | dwZpos |
| CH4 | Rx | **dwVpos** (not dwRpos!) |
| CH5 | Ry ("Y Rotation") | **not visible** |
| CH6 | Rz | dwRpos |
| CH7 | Slider | dwUpos |
| CH8 | Dial | **not visible** |
| CH9-32 | buttons | dwButtons |

So a switch mixed onto CH5 (or CH8) moves in the Windows Game Controllers
dialog (DirectInput sees all axes) but is **invisible** to winmm — no
movement in any polled axis, no bind capture. This bit the FPV mod for real;
the direct-HID rewrite above was the fix. Radio-side workarounds (if stuck
with winmm): move the mix to CH6/CH7, or to **CH9+** where it reports as a
plain HID *button*; EdgeTX's USB joystick "Advanced" mode can also set a
channel's mode to `Btn` explicitly. Source:
<https://manual.edgetx.org/edgetx-how-to/joystick-mapping-information-for-game-developers>

Also, `JOYCAPS.szPname` is the **driver** string ("Microsoft PC-joystick
driver"), not the product name; getting the real name under winmm requires a
registry crawl through `MediaResources\Joystick\<szRegKey>` →
`MediaProperties\...\Joystick\OEM\<VID_xxxx&PID_xxxx>\OEMName` (HKCU first,
then HKLM). With hid.dll, `HidD_GetProductString` makes all of that obsolete.

## Gotcha: transmitter switches can report as axes, not buttons

Depending on the radio's channel/mixer setup, a 2/3-position aux switch may be
mapped onto an analog axis instead of a button bit — a press-to-bind button
capture will never see it. Treat "axis as switch": at bind time record the
axis index plus its **rest position** (normalized 0..1 value when capture
started), then consider the action held while `|value - rest| > 0.25`. The
rest-relative delta self-calibrates regardless of which way the switch throws
or whether it idles at 0, 0.5, or 1. Exclude axes already bound to flight
channels from this capture so brushing a gimbal doesn't steal the bind.
Implementation: `TryGetCapturedActionAxis` in `FPV/src/FPVControls.cs`.

## Gotcha: throttle channel direction varies per radio — a reversed one reads "stuck"

A transmitter's throttle channel can be reversed (radio-side channel reverse,
mixer weight -100, or just how the model was set up), so stick-bottom reports
raw **1.0** and stick-top **0.0**. A non-centering throttle read as `raw 0..1`
then fails in a deceptively quiet way that presents as "the craft won't take
off" (note: a real FPV-mod stuck-on-ground bug turned out to be resting-contact
sweep freeze instead — see *Unity Sweep Casts & Resting Contact* — so confirm
with the live axis readout before blaming the radio):

- Stick at idle reads high → any "drop throttle to idle" arming safety never
  clears (or clears only at *full* stick).
- Once armed, pushing the stick up commands ~0% thrust, so the craft never
  lifts; the motor audio still responds to stick movement (mirrored), which
  makes the input path look healthy.

Diagnosis trick: if motor audio pitch is driven by commanded throttle (FPV's
loop sweeps pitch 0.52→1.85 over throttle 0→1), the periodic `audio dbg` log
lines are a readout of what the flight model actually received — pitch pinned
near the bottom of the sweep while the pilot holds full stick proves the
command is ~0 without any extra instrumentation. Fix: per-binding `Invert`
flag (`controls.json` → `Throttle.Invert`, or Options → Controls), or reverse
the channel on the radio — not both. The mod now also logs the live stick %
while the throttle safety holds, and prints it in the "Disarmed!" HUD prompt.
