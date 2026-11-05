# Build and bring-up

Install on the Pi, check every device, then bring the robot up one step at a time.
Hardware problems hit along the way are in [troubleshooting.md](troubleshooting.md).

One process owns the hardware. Do not run `poke.py`, `preflight.py` or
anything else that touches a device while the server is up. The serial port is
opened exclusively, so a second opener fails at once with "Could not
exclusively lock port"; stop `hexapod.service` first. Before that lock existed,
two processes could share the port and both drive the servos, which looks like
random twitching and is not a hardware fault.

## Install

On the Pi:

```sh
git clone https://github.com/C00kieMonsta/pi-hexapod.git && cd pi-hexapod
python3 -m venv --system-site-packages .venv && .venv/bin/pip install -e '.[dev]'
```

`--system-site-packages` matters: `picamera2` is apt-installed
(`sudo apt install python3-picamera2`) and a plain venv cannot import it.

On a laptop, to try it without hardware:

```sh
python3 -m venv .venv && .venv/bin/pip install -e '.[dev]'
.venv/bin/hexapod --dry-run serve
```

`serve` opens a laptop-camera Hand modal. Lean maps the palm onto chassis pose.
Pincer maps index and middle onto L1/R1. Toggle in the modal. The robot live
view stays the robot camera. Pose stays inside the operator limits and does
not change ride height. `--no-hand` turns the modal off. The laptop camera
needs https (http only works on localhost), so `serve` mints a self-signed
cert. Accept the browser warning once. `--no-tls` is plain HTTP. Fetch the
MediaPipe wasm and landmarker once (`tools/fetch-hand-assets.sh`); they are
gitignored.

```sh
tools/fetch-hand-assets.sh
.venv/bin/pip install -e '.[dev]'
.venv/bin/hexapod --dry-run serve
```

Open the URL it prints. `/` is the client, `/status` is the panel page. The
dry run fakes the board, a rocking IMU and a placeholder camera, so every page
and endpoint works with nothing attached.

## Preflight

One command that checks everything the robot needs, on the Pi:

```sh
python3 tools/preflight.py
```

It checks the Python environment, that no file got zero-filled by a bad
shutdown, that the config loads and its two clamps agree, how often the gait
would need to clamp a joint, that the accelerometer answers and reads 1 g,
that `picamera2` imports and sees a camera, and that the Servo2040 replies
over serial. Exit code is 0 if nothing failed. Stop `hexapod.service` first;
the checks open the same devices.

Every check is independent and catches its own errors, so missing hardware
reports as one bad line instead of a traceback. Skip what is not attached:

```sh
python3 tools/preflight.py --skip-board        # no Servo2040 on this Pi
python3 tools/preflight.py --skip-imu          # no IMU wired yet
python3 tools/preflight.py --skip-camera       # no camera fitted
python3 tools/preflight.py --imu-seconds 10    # longer gyro average
```

Run it before every session once the robot is assembled. Run the numbered
bring-up below the first time, and after changing geometry or calibration.

## Bring-up

Do these in order. Each step must work before you start the next. Keep the robot on a stand until step 6.

**1. Find the board.**

```sh
hexapod ports
```

Put the device path in `serial.port` in [config/hexapod.yaml](../config/hexapod.yaml).

**2. Check the link.** No servo movement. Reads voltage, current and the touch pins.

```sh
python3 tools/poke.py /dev/ttyACM0
```

This script imports nothing from this repo. If it works, the wiring, firmware and port are fine.

What the voltage means depends on how the board is powered:

| State | Expected |
|---|---|
| USB only, *Separate USB and Ext. Power* trace intact | ~5.0 V |
| Trace cut, no battery on the terminals | ~0 V. The rail is genuinely dead, which is the point of cutting it |
| 2S pack on the terminals | 7.4 to 8.4 V |

The firmware scales the reading by 1024/3.3 before sending it. Measured 5.08 V on USB before the trace was cut, which rules out the codec having grabbed the firmware's other constant (`b1024_5V_RATIO` would have shown 7.7 V or 3.35 V). It has still not been checked against a meter on battery. If it is off, fix `_B1024_3V3_RATIO` in [hexapod/protocol.py](../hexapod/protocol.py).

**3. Check the maths.** Offline. No board needed.

```sh
hexapod check
```

It prints joint travel over the full command envelope and flags any limit the gait would hit.

**4. Centre the servos.** Legs off the ground.

```sh
python3 tools/poke.py /dev/ttyACM0 --centre
```

Every servo goes to 1500 us. This is your mechanical zero, and it is the pose
the horns must be fitted in, each joint at its `geometry.*_attach_angle`.
The fitting pose is drawn in
[docs/servo-fitting.svg](servo-fitting.svg), with every servo's number:
leg **straight out along its mount line** (coxa 0), thigh **35° above
horizontal**, knee **folded 68° from straight** (112° inside angle, a fairly
open knee). Chica's coxa value is -8, but the coxa angle is not mirrored here,
so -8 would turn the right legs back and the left legs forward; 0 is the same
on both sides. Refitted this way on 5 Oct 2026; the tibias were at 134° fold
before. All six identical, and
one value covers all six legs, so a replacement servo must match it. Do not fit
the arms in a pose that "looks like standing" or "looks like sitting". The
attach angles must describe the pose you actually fitted; check `hexapod check`
for lost range.

**5. Fix the joint directions.** One leg at a time, still elevated.

```sh
hexapod jog
```

Two checks per leg:

- `L1 0 35 68` (the three `geometry.*_attach_angle` values; use whatever the config says) must put all three servos back at 1500 us. Jog opens at the sit pose, so this is also how you centre a leg.
- `L1 0 0 30` must point the leg straight out sideways, thigh horizontal, knee nearly straight (30° fold is the joint limit).

If a joint moves the wrong way, flip `direction` from `1` to `-1` for that servo in the config. If a joint is centred but at the wrong angle, correct its attach angle.

The hip is the one to get right first, and a big swing reads better than a
small one: `R2 coxa 45` must point the leg diagonally towards the **camera
end**, `L2 coxa 45` towards the **rear**. Both sides share the same coxa sign,
because the hip turns about the vertical axis and the coxa part is the same
print on all six legs; only femur and tibia mirror. A hip the wrong way round
does not show at sit or stand. It shows on the floor: a turn command walks
straight and a forward command turns. See the 8 Oct 2026 entry in
[docs/troubleshooting.md](troubleshooting.md).

Femur and tibia are easier to read as a sit-to-stand move than as a single
angle. These are the neutral poses the solver produces at `sit_height` 40 and
`ride_height` 105, the same for all six legs to within 0.3°:

| joint | sit | stand | sit → stand |
|---|---|---|---|
| coxa | 0.1° | 0.1° | nothing |
| femur | 78.1° | 27.3° | drops 50.8° |
| tibia | 132.0° | 106.8° | knee opens 25.2° |

In jog, `R3 0 78.1 132.0` then `R3 0 27.3 106.8`. The femur must rotate **down**
and the knee must **open**, pushing the foot further below the body. A femur
that rises is an inverted femur, a knee that closes is an inverted tibia, and
both can be wrong at once. Coxa barely moves between the two poses, so check it
on its own: `R3 coxa 20` swings the leg counter-clockwise seen from above.

`tibia` is the fold at the knee, 0 = tibia in line with the femur. The interior
femur-to-tibia angle you would put a protractor on is 180 minus that, so 48°
sitting and 73° standing. Same movement, and the two conventions run opposite
ways, so be clear which one a number is in before acting on it.

Flipping `direction` does not move the servo's centre. `servo_angle =
direction * (joint_angle - attach_angle)`, so the servo still sits at 1500 us
when the joint is at its attach angle whichever sign `direction` has; only the
travel either side of it mirrors. Nothing needs remounting after a flip. A horn
that is a spline tooth out shows up as the wrong *angle* in the attach-angle
check above, not as the wrong direction.

Do all six legs. Left and right are mirror images, so the signs are not
guaranteed to match across a pair, and copying R3's answer to the other five is
how you get one leg walking backwards.

**Watch what actually went to the servos.** With `hexapod serve` running, on
the Pi or a laptop:

```sh
python3 tools/servolog.py                    # localhost
python3 tools/servolog.py --host hexapod.local --csv stand.csv
```

Then press Stand, Sit or Torque on `/`. It prints one line per channel that
changed, with the leg and joint, the SERVO header number, the joint angle the
solver asked for and the pulse that went out:

```
14:02:11.318  R3 femur   SERVO  2 (ch  1)    77.8 ->   27.3 deg   1976 -> 1414 us (-562)
```

Ctrl-C prints a summary: start, end, net travel and range for every channel,
which ones never moved, which hit the pulse clamp, and whether every leg was
asked for the same joint change (for Stand and Sit they should be). It cannot
see which way the leg physically turned, so read it next to the robot: if
SERVO 2 went +122 us and the knee went up when the table says the femur should
drop, that is the servo whose `direction` flips.

**If a commanded joint does not move at all, take a census.** The log says
which header was driven; the census says what is on it, with the config out of
the loop. Stop the service first, legs off the ground:

```sh
sudo systemctl stop hexapod
python3 tools/poke.py /dev/ttyACM0 --census
```

**Is a servo dead, suspect, or fine?** Same tool, no eyes needed:

```sh
sudo systemctl stop hexapod
python3 tools/poke.py /dev/ttyACM0 --probe
```

Each header is wiggled from where it is and the board's current sensor is read
the whole time. One line per servo with the peak while moving and what it
settled to a second later, then a summary. Read it like this:

| reading | means | do |
|---|---|---|
| peak 0.5 A or more over idle, settles to idle | moved; healthy FT5330Ms have read 0.9 to 3.7 A | nothing |
| peak 0.15 to 0.5 A over idle | **weak**: two or three ADC counts, not a real move | check the plug, then swap it with a neighbour |
| peak under 0.15 A over idle | **dead**, or unplugged, or lead broken | swap its plug with a neighbour: fault follows the plug = servo or lead |
| peak over 4.5 A (above the FT5330M's 3.9 A stall spec) | **suspect**: an external bind, or internal damage | torque off, turn the joint by hand: a catch is a bind to fix first; free, suspect the servo |
| still pulling 0.8 A+ a second after the move | **stalled or bound** | something is in the way, find it before it cooks |
| "well above the pack" | 40%+ hungrier than its siblings | watch it; re-run after a session and compare |

The HIGH line is set for the FT5330M. A healthy DS3235 PRO peaks up to 4.4 A,
close to it.

In five probe runs one afternoon, SERVO 17 (L1 femur) peaked at 4.4 to 5.4 A
while the others read 1.3 to 3.7 A, and it was dead by evening. SERVO 8 read
normal about 40 minutes before it died, so a clean probe does not clear a
servo. Torque comes on at the `--base` pose (1500 us without it); the probe
then stops before wiggling any header if the robot already pulls more than
0.5 A at rest, because a servo stalled at rest hides inside every per-header
reading. It also stops the run, relay off, when a header is still pulling a
second after it returns, and when the board stops answering for 2 s. Run it
legs free, at the start of a session and after any mechanical change.

**Mapping headers to joints** is `--census` instead of `--probe`, and does
need eyes. Each of the 18 headers wiggles ±200 us in turn and you type what moved as
position and joint, `RR femur`, `FL coxa`. Positions are FL FR ML MR RL RR;
front is the camera end, left and right are the robot's own. At the end it
prints the header-to-joint table and a `servos:` block to paste into
[config/hexapod.yaml](../config/hexapod.yaml). Because you name legs by where
they are, not by what the config calls them, this also settles whether R1 is
really at the front.

**Or let the board tell you which headers have a live servo.** No eyes needed:

```sh
P=$(curl -k -s https://localhost:8000/api/state | python3 -c 'import json,sys; print(json.dumps(json.load(sys.stdin)["pulses"]))')
sudo systemctl stop hexapod
python3 tools/poke.py /dev/ttyACM0 --probe --base "$P"
```

It wiggles each header from where the servo already is and watches the current
sensor. A servo that moves pulls 1 to 5 A; a header with nothing on it, or a
dead servo, pulls nothing. On this robot that found SERVO 2 silent while the
other 17 answered, which was the whole of "only the tibia moves".

**6. Neutral stance.** Still elevated. Then lower it onto the ground.

```sh
hexapod neutral --height 80
```

**7. Walk.**

```sh
hexapod serve
```

It lists every address it is reachable at, and the panel page is `/status` on
any of them. Use the one on your wifi or ethernet interface (`en0`, `wlan0`, `eth0`). If you are on a VPN, the tunnel address is listed too and marked. It will not work from other devices.

## Deploy

On the Pi, once the venv exists:

```sh
deploy/install.sh
```

It installs [deploy/hexapod.service](../deploy/hexapod.service) as a system
service (`After=network-online.target`, `Restart=always`, binds `0.0.0.0:8000`
so the laptop can reach it) and
[deploy/hexapod-kiosk.service](../deploy/hexapod-kiosk.service) as a user service
that runs [deploy/kiosk.sh](../deploy/kiosk.sh): wait for the display and the
server, then Chromium in kiosk mode on `/status`. It also adds the user to
`dialout`, `i2c` and `video` and enables lingering so the kiosk starts at boot.

```sh
journalctl -u hexapod -f               # server
journalctl --user -u hexapod-kiosk -f  # panel
```

If the kiosk unit does not start on your image, put the same line in the
compositor's autostart instead: `~/.config/labwc/autostart` on current
Raspberry Pi OS, `[autostart]` in `~/.config/wayfire.ini` on 2023 images. Panel
rotation is a display setting, not ours.

This was written against the Bookworm desktop and has not been run on the
robot's Pi yet. Expect to adjust the display detection in `kiosk.sh`.

## Configuration

Everything lives in [config/hexapod.yaml](../config/hexapod.yaml).

Servo channels and calibration are ported from MakeYourPet's `chica-config-2040.txt`. The `P` numbers in that file are the same integers the firmware uses as channel indices, so `P15` is `channel: 15`. If you already calibrated your servos with Chica, copy the two microsecond values per servo straight across.

The stance block is ours, not a copy of Chica's `MODE_STANDARD`. It puts every foot about 138.5 mm out from its own coxa, which is what Chica's radius, corner angle and elongation constants work out to.
