# eMeet M0 HID LED protocol

Reverse-engineered from the device's HID report descriptor
(`sudo usbhid-dump -d 0483:5730 -e desc`).

## Device

```
Bus 001 Device 004: ID 0483:5730 STMicroelectronics Audio Speaker
iManufacturer: eMeet Tech
iProduct:      eMeet M0
```

Interfaces:

| Interface | Class | Driver |
|-----------|-------|--------|
| 0         | Audio (Control)       | snd-usb-audio |
| 1         | Audio (Streaming)     | snd-usb-audio |
| 2         | Audio (Streaming)     | snd-usb-audio |
| 3         | Human Interface Device | usbhid |

The device shows up as `/dev/hidrawN` (any N — use the sysfs match below).

## Report descriptor (annotated)

```
05 0C        Usage Page (Consumer)
09 01        Usage (Consumer Control)
A1 01        Collection (Application)
  85 01      Report ID 1
  09 E9 09 EA   Volume Increment / Volume Decrement
  95 02 75 01   two 1-bit inputs (volume buttons)
  ...
C0           End Collection

05 0B        Usage Page (Telephony)
09 05        Usage (Phone Mute)
A1 01        Collection (Application)
  85 02      Report ID 2
  09 20      Usage (Mute)
  81 22      Input (Data,Var,Abs)     <- mute button state
  ...
C0           End Collection

05 08        Usage Page (LEDs)          <-- the interesting part
  85 03      Report ID 3
  09 18      Usage (Battery Good)
  91 22      Output (Data,Var,Abs)     <- ring: flash green + btn: green
  85 04      Report ID 4
  09 09      Usage (Mute)
  91 22      Output                     <- ring: solid red
  85 05      Report ID 5
  09 17      Usage (Battery Full)
  91 22      Output                     <- btn: solid red
  85 06      Report ID 6
  09 20      Usage (Generic Indicator)
  91 22      Output                     <- ring: solid blue
```

All LED usages are **1-bit** (Report Size 1, Logical Min 0 / Max 1) — no RGB
channels. The firmware maps each report to a fixed color/behaviour.

## Observed mapping (tested on hardware)

| Report ID | Usage       | Ring light        | Call button   |
|-----------|-------------|-------------------|---------------|
| 3         | Battery Good | 🟢 flashing green | 🟢 solid green |
| 4         | Mute        | 🔴 solid red      | —             |
| 5         | Battery Full | —                 | 🔴 solid red  |
| 6         | Generic Indicator | 🔵 solid blue | —          |

## Combination rules

- Reports 3/5 (button) and 4/6 (ring) are independent: e.g. `report 6` +
  `report 5` gives blue ring + red button (verified).
- Report 3 controls both ring and button together.
- Unsupported on this hardware: green button with red/blue ring, and
  flashing green ring with red button.

## Writing reports

Send the report ID as the first byte of an output report to `/dev/hidrawN`:

```python
with open("/dev/hidraw0", "wb") as f:
    f.write(bytes([4, 1]))   # report 4 = ring red, value 1 = on
```

Auto-discovery of the device node:

```sh
grep -l "HID_ID=0003:00000483:00005730" /sys/class/hidraw/*/device/uevent
```

## Future work / unknowns

- Whether any hidden vendor control transfers can reach an RGB mode
  (the standard descriptor shows none; firmware likely has no RGB path).
- Exact power-on default state and whether `green` (solid) ring is reachable
  from the host (the device shows solid green ring on connect by itself).
