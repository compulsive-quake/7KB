# Reading HID Joysticks (winmm)

7DTD uses Unity's **legacy** Input Manager, so `Input.GetAxis` only sees axes
baked into the shipped `InputManager.asset` — a mod cannot add axes. To read a
DirectInput/HID joystick (e.g. a RadioMaster transmitter in "USB Joystick
(HID)" mode), bypass Unity and P/Invoke the Win32 multimedia joystick API in
`winmm.dll`: `joyGetNumDevs` / `joyGetPosEx` / `joyGetDevCapsW`. This exposes
up to 6 analog axes (X,Y,Z,R,U,V), 32 buttons, and a POV hat. Channel order
varies per radio/firmware, so bind axes by detection ("move the stick"), not a
fixed mapping. Working implementation: `FPV/src/FPVGamepad.cs`.

## Gotcha: device name

`JOYCAPS.szPname` is the **driver** string — for any generic HID device it
reads "Microsoft PC-joystick driver", not the product name the Windows Game
Controllers dialog shows. The friendly name lives in the registry:

1. `HK{CU,LM}\System\CurrentControlSet\Control\MediaResources\Joystick\<szRegKey>\CurrentJoystickSettings`,
   value `Joystick<id+1>OEMName` → an OEM subkey name like `VID_1209&PID_4F54`.
2. `HK{CU,LM}\System\CurrentControlSet\Control\MediaProperties\PrivateProperties\Joystick\OEM\<that>\OEMName`
   → the friendly name (e.g. "Radiomaster Pocket Joystick").

If step 1 yields nothing, build the subkey directly from `JOYCAPS.wMid`/`wPid`
as `VID_{mid:X4}&PID_{pid:X4}`. Check **HKCU first, then HKLM** — DirectInput
writes OEM keys per-user on non-admin installs. Some entries have an empty
`OEMName` value, so fall back to `szPname` when the lookup comes up blank.
This is the same lookup GLFW/SDL use; see `ResolveDeviceName` in
`FPV/src/FPVGamepad.cs`.

## Gotcha: winmm cannot see HID Ry ("Y Rotation") or the 8th axis

`joyGetPosEx` exposes only 6 of the 8 HID analog axes, and the mapping is not
what the field names suggest. Per the EdgeTX manual (USB joystick "classic"
mode, CH1-8 → HID X, Y, Z, Rx, Ry, Rz, Slider, Dial):

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

So a switch mixed onto CH5 (or CH8) shows up and moves in the Windows Game
Controllers dialog (DirectInput sees all axes) but is **invisible** to winmm —
no movement in any of the 6 polled axes, no bind capture. Fixes, radio side:
move the mix to CH6/CH7, or better to **CH9+** where it reports as a plain HID
*button*; EdgeTX's USB joystick "Advanced" mode can also set a channel's mode
to `Btn` explicitly. Code side, the only real fix is Raw Input / hid.dll
instead of winmm. Source:
<https://manual.edgetx.org/edgetx-how-to/joystick-mapping-information-for-game-developers>

## Gotcha: transmitter switches can report as axes, not buttons

Depending on the radio's channel/mixer setup, a 2/3-position aux switch may be
mapped onto one of the 6 analog axes instead of a button bit — a press-to-bind
button capture will never see it. Treat "axis as switch": at bind time record
the axis index plus its **rest position** (normalized 0..1 value when capture
started), then consider the action held while `|value - rest| > 0.25`. The
rest-relative delta self-calibrates regardless of which way the switch throws
or whether it idles at 0, 0.5, or 1. Exclude axes already bound to flight
channels from this capture so brushing a gimbal doesn't steal the bind.
Implementation: `TryGetCapturedActionAxis` in `FPV/src/FPVControls.cs`.
