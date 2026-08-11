# SharkBot — Web Bluetooth Robot Car Dashboard

A single-page dashboard that connects **directly from the browser** to an Adafruit
Feather 32u4 Bluefruit LE over BLE and drives a four-motor robot car. No server, no
app, no build step — it replaces the Adafruit "Bluefruit Connect" mobile app.

```
Chrome / Edge  ──Web Bluetooth──▶  nRF51822 (NUS UART)  ──SPI──▶  ATmega32u4  ──I²C──▶  Motor Shield v2 ──▶ MOTOR_A..D
   index.html                          Feather 32u4 Bluefruit LE          firmware/robot_control.ino
```

| File | Purpose |
|---|---|
| `index.html` | The whole dashboard — HTML/CSS/JS inline, framework-free |
| `firmware/robot_control.ino` | Arduino sketch for the Feather 32u4 |
| `README.md` | This file |

---

## 1. Flash the firmware

**Arduino IDE setup (once):**

1. `File → Preferences → Additional Boards Manager URLs`, add:
   `https://adafruit.github.io/arduino-board-index/package_adafruit_index.json`
2. `Tools → Board → Boards Manager` — install **Adafruit AVR Boards**, then select
   **Adafruit Feather 32u4**.
3. `Tools → Manage Libraries` — install:
   - **Adafruit Motor Shield V2 Library**
   - **Adafruit BluefruitLE nRF51**

**Upload:**

1. Open `firmware/robot_control.ino`. The IDE will offer to move it into a folder
   named `robot_control/` — accept (Arduino requires the sketch folder to match the
   sketch name).
2. Select the Feather's serial port and click Upload.
3. **If the upload fails or the port vanishes:** double-tap the Feather's reset
   button — the red LED pulses and the bootloader port appears for ~8 seconds.
   Click Upload again while it's pulsing.

**Pin assumptions** (top of the sketch): five IR sensors on `IR1..IR5` = `A0..A4`
(marked `// TODO: confirm pin` — confirm the physical left→right order matches),
four motors on shield ports 1–4 (`MOTOR_A..MOTOR_D`), BLE on hardware SPI with
CS=8, IRQ=7, RST=4. `MOTOR_A`/`MOTOR_B` drive the **right** wheel pair and
`MOTOR_C`/`MOTOR_D` drive the **left** pair — recovered from the chassis's
original app sketch. If turns come out mirrored, swap those two pairs; if a
single wheel spins backwards, swap that motor's two leads at the shield
terminal instead of touching the code.

## 2. Serve the page

Web Bluetooth only runs in a **secure context** — serve over `http://localhost`
or HTTPS. Behaviour on `file://` is inconsistent across Chrome versions, so don't
rely on double-clicking the file:

```bash
cd project-folder
python3 -m http.server 8000
# then open http://localhost:8000
```

To share it, GitHub Pages works as a free HTTPS host — push the repo, enable
Pages, done. Nothing else changes; the BLE connection is always browser→robot,
never through a server.

## 3. Browser support (read before debugging "it won't connect")

| Browser | Works? |
|---|---|
| Chrome / Edge — Windows, macOS, ChromeOS, Android | ✅ |
| Chrome — Linux | ✅ (older versions may need `chrome://flags/#enable-experimental-web-platform-features`) |
| Safari — any platform, incl. **all iPhones/iPads** | ❌ Web Bluetooth not implemented |
| Firefox — any platform | ❌ Web Bluetooth not implemented |

iOS cannot run this at all — every iOS browser is Safari's engine underneath.
Opening the page in an unsupported browser shows a plain explanation instead of
a dead dashboard.

## 4. Connect and drive

1. Power the robot (4×AA for motors, USB for logic). It advertises as **SharkBot**.
2. Open the page in Chrome/Edge, click **CONNECT** (the chooser only opens from a
   real click — that's a Web Bluetooth rule).
3. Pick *SharkBot* in the chooser. Status flips to CONNECTED and the LINK STATUS
   panel starts updating at ~10 Hz — that's your proof the link is live.
4. **Hold** a direction button (or W/A/S/D / arrows) to drive; release to stop.
   SPACE or ESC is an immediate stop. The speed slider sends `V<n>`; the MODE
   switch toggles Manual / Line Follow.

## 4½. Dashboard extras

- **Route recorder** — REC captures your driving (direction and speed changes,
  with timing); PLAY replays it through the same 150 ms heartbeat pipeline, so
  every failsafe still applies. Any manual input, STOP, mode change, hidden
  tab, or disconnect aborts playback instantly. Manual-mode only — start a
  recording or playback before switching to Line Follow.
- **⚙ display settings** — the gear icon opens a panel with four color themes
  (Cyan / Amber / Violet / Mint; only the "connected/live" accent changes —
  fault-red stays constant so its meaning never shifts) and three layouts:
  **Standard**, **Focus** (a larger drive pad), and **Compact** (single
  column). Both are saved per-device and apply instantly, no reload, no BLE
  connection needed. Below 880 px the page always uses the compact stack
  regardless of the layout choice — there isn't room for anything else.

## 5. The protocol (both sides implement exactly this)

Plain ASCII, newline-terminated — debuggable from a serial monitor.

**Browser → robot** (each + `\n`; unknown commands are silently ignored):

| Command | Meaning |
|---|---|
| `F` `B` `L` `R` | drive forward / back, pivot left / right (hold-to-drive) |
| `S` | stop — all four motors released |
| `V<0-255>` | set speed, e.g. `V180` |
| `M0` / `M1` | manual / line-follow mode |

**Robot → browser:**

| Line | Cadence |
|---|---|
| `T,<irL>,<irR>,<speed>,<mode>` | 10 Hz — the two outer IR readings (raw, 0–1023), confirmed speed, confirmed mode |
| `E,DEADMAN` | the firmware stopped itself — shown red in the event log |

Notifications fragment at 20 bytes, so the page buffers bytes and only parses
on `\n` — never assume one notification is one line.

## 6. How it stays safe — the part worth explaining out loud

A dropped BLE connection while the motors are running is an **uncommanded-motion
hazard**. So the system never trusts a single "go" command:

- **150 ms heartbeat (browser):** while a drive button is held, the page re-sends
  the command every 150 ms. While line-follow mode is active it re-sends `M1` on
  the same cadence. Releasing everything sends `S`.
- **400 ms deadman (firmware):** every received command refreshes a timestamp. If
  the robot is moving — or is autonomous in line-follow mode — and 400 ms pass
  with no command, the firmware releases all four motors and drops to manual on
  its own. Two missed heartbeats are survivable; a dead link is not, by design.

Closing the tab, walking out of range, or killing the browser all look identical
to the firmware: silence → stop within ~400 ms. Hiding the tab also releases the
controls page-side, and a lost GATT link locks the UI with **no silent
auto-reconnect** — reconnecting is a deliberate human action.

## 7. Line follow

Five sensors (`IR1`–`IR5` = `A0`–`A4`) read every loop. Any sensor over `VAL`
(default `100` — the value from the original working sketch; retune it for
your lighting/track with a serial monitor if it seems off) is treated as
"sees black":

- Either left sensor (`IR1`/`IR2`) sees black → turn left
- Else either right sensor (`IR4`/`IR5`) sees black → turn right
- Else the middle sensor (`IR3`) sees black → drive forward
- Else (nothing sees the line) → stop

Any manual drive command (`F`/`B`/`L`/`R`/`S`) received while in line-follow
mode drops the robot back to manual immediately.

## 8. Bench test without the web page

The sketch accepts the same commands over the USB serial monitor (115200 baud,
newline line-ending): type `F` and the car drives — then watch it stop ~400 ms
later. That's not a bug; that's the deadman working (the monitor doesn't send a
heartbeat). `V60` then `F` repeatedly is a gentler way to test on the bench.
`M1` puts it in line-follow. Received commands echo as `> F`, and a deadman
trip logs itself.

## 9. Acceptance checklist

1. Page on `http://localhost:8000` in Chrome → disabled dashboard, working CONNECT. ✅
2. CONNECT opens the chooser; *SharkBot* appears. ✅
3. On connect, telemetry numbers update ~10×/sec. ✅
4. Holding Forward drives; releasing stops. ✅
5. Speed slider changes motor speed; telemetry `speed` field tracks it. ✅
6. **Failsafe:** while driving, kill robot power or walk out of range → motors stop
   within ~400 ms; after re-powering, the car must not move on its own. ✅
7. Closing the tab mid-drive stops the car (same deadman path). ✅
8. Safari shows the unsupported-browser message, not a broken dashboard. ✅

## 10. Troubleshooting

| Symptom | Likely cause / fix |
|---|---|
| Robot not in the chooser | Robot unpowered, out of range, or OS Bluetooth off. On Linux, try the Chrome flag above. Nearby: check the module's blue LED. |
| `SecurityError` on `getPrimaryService` | `optionalServices: [UART_SERVICE]` missing from `requestDevice` — it's mandatory even though the service is in `filters`. |
| Connects but nothing happens, no errors | The classic RX/TX swap. The browser must **write to `…0002`** and **subscribe to `…0003`** — they're named from the peripheral's side. Getting it backwards is a silent no-op. |
| Upload fails / port missing | Double-tap reset to enter the bootloader, upload while the LED pulses. |
| Compiles but behaves erratically at runtime | RAM exhaustion. Keep the 2.5 KB rule: no `String`, literals in `F()`, telemetry printed field-by-field. |
| A wheel spins backwards | Swap that motor's two leads at the shield terminal. |
| One whole side never moves | That side's pair isn't being driven — check `MOTOR_A/B` (right) and `MOTOR_C/D` (left) actually match how the motors are plugged in. |
| L and R turn the wrong way | Right/left pairs swapped — the geometry needs `MOTOR_A/B` on the opposite side. |
| Line mode turns the wrong way / never stops | Sensor polarity or threshold — retune `VAL`, or check whether your sensors read low-over-black instead of high-over-black (the comparisons in `lineFollow()` assume high = black). |
| Robot works fine tethered to your laptop, breaks the instant you unplug USB | Almost always a **power** issue, not a code issue — brownout from motor current sagging the logic rail once USB's stiff 5V supply is gone. Try fresh AA batteries first; then confirm motor power and logic power are actually on separate rails as wired (motors → AA pack via the shield's motor terminal, logic → USB or a healthy LiPo — never both off the same weak source). |
| Telemetry shows STALE while connected | Notifications died without a disconnect event — disconnect and reconnect; check robot power. |

## 11. Hardware recap (fixed)

| Part | Detail |
|---|---|
| Board | Adafruit Feather 32u4 Bluefruit LE (ATmega32u4 + nRF51822, 2.5 KB SRAM) |
| Motors | 4× DC via Adafruit Motor Shield **v2** (I²C) — right pair MOTOR_A/B, left pair MOTOR_C/D |
| IR sensors | 5-sensor analog reflectance array on `A0`–`A4` (assumed left→right — confirm!); line-follow reads all five |
| BLE | Hardware SPI — CS 8, IRQ 7, RST 4 |
| Power | 4×AA motors, USB logic |

Motor shield is I²C, BLE is SPI — no pin conflict.
