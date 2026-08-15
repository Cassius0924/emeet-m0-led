# eMeet M0 LED control

Control the LED ring and call-button light of the **eMeet M0 USB conference
speakerphone** from Linux — no vendor software required.

Reverse-engineered from the device's HID report descriptor. The M0 exposes a
HID interface (besides its audio interfaces) whose **output reports** drive the
lights. The protocol is not documented by eMeet.

![device](https://emeet.com/cdn/shop/files/officecore-m0.png) <!-- placeholder: replace with real photo -->

## Why this exists

The eMeet M0's pretty ring light is normally only controlled by the device
firmware (green = connected, red = muted, etc.). But the HID interface
(`0483:5730`, "eMeet Tech eMeet M0") turns out to accept LED output reports.
This tool lets you set the ring color and the call-button LED from the command
line — handy for status indicators ("on air" light, meeting-in-progress
signals, notification light) or just for fun.

## Supported effects

| Effect       | Ring light            | Call button    |
|--------------|-----------------------|----------------|
| `red`        | 🔴 solid red          | off            |
| `blue`       | 🔵 solid blue         | off            |
| `green`      | 🟢 flashing green     | 🟢 solid green |
| `btn-red`    | off                   | 🔴 solid red   |
| `all-red`    | 🔴 solid red          | 🔴 solid red   |
| `blue-red`   | 🔵 solid blue         | 🔴 solid red   |
| `off`        | off                   | off            |

No RGB: the HID descriptor only exposes single-bit LEDs (usage page 0x08, LED
usages 0x09/0x17/0x18/0x20), so colors are limited to what the firmware knows.

Not supported (hardware): green button with red/blue ring, and flashing green
ring with red button.

## Install

```sh
git clone https://github.com/Cassius0924/emeet-m0-led.git
sudo install -m 0755 emeet-m0-led/emeet-led /usr/local/bin/emeet-led
```

Permissions: `/dev/hidrawN` is root-only by default. Either run with sudo
(the script auto re-execs via `sudo`), or install the udev rule so any user in
the `plugdev` group can access it:

```sh
sudo install -m 0644 emeet-m0-led/udev/99-emeet-m0.rules /etc/udev/rules.d/
sudo udevadm control --reload && sudo udevadm trigger
```

## Usage

```sh
emeet-led blue              # blue ring
emeet-led all-red           # red ring + red call button
emeet-led btn-red           # call button red, ring off
emeet-led off               # everything off
emeet-led demo              # cycle through the effects
emeet-led <ring> <btn>      # arbitrary combo: ring=red|blue|green|off btn=green|red|off
emeet-led -p /dev/hidraw2   # explicit device (auto-detected by default)
emeet-led -d 5 blue         # show for 5 s, then off (blocking, decimals ok: -d 0.5)
emeet-led -h                # help (--help works too)
```

The device is auto-discovered by VID:PID (`0483:5730`) under
`/sys/class/hidraw/`, so the script works regardless of the `hidrawN` number.

Unsupported combos (green button with red/blue ring, green ring with red
button) are rejected by the tool with an error.

## How it works

See [docs/protocol.md](docs/protocol.md) for the full HID report descriptor
analysis and the combination rules.

## Home Assistant

A Home Assistant integration (light entity + automation examples) is in
[`ha/`](ha/) — see [ha/README.md](ha/README.md).

## License

MIT
