# IMU (GY-521)

The GY-521 breakout carries an MPU-6050: 3-axis accelerometer plus 3-axis
gyro, over i2c at address `0x68` (`0x69` if AD0 is pulled high).

**Wiring.** Four pins, nothing else needs connecting.

| GY-521 | Pi |
|---|---|
| VCC | 3.3 V (pin 1) |
| GND | GND (pin 6) |
| SDA | GPIO2 (pin 3) |
| SCL | GPIO3 (pin 5) |

**Enable i2c** once, then reboot:

```sh
sudo raspi-config nonint do_i2c 0
sudo reboot
```

**Check it.**

```sh
python3 tools/preflight.py --skip-board
```

Three things have to be true:

- `WHO_AM_I 0x68`. Anything else means the wrong chip or the wrong address.
  Nothing at all usually means SDA and SCL are swapped.
- `|a|` within 0.10 of 1.000 g. Hold the board still for this.
- Gyro bias under 10 deg/s on every axis.

If the check reports `Errno 121`, nothing acknowledged the address. Scan the
bus before touching any code:

```sh
i2cdetect -y 1
```

Expect `68` for the IMU. There is also a device at `5d` that shows as `UU`
once its driver has loaded: that is the screen's Goodix GT9271 touch
controller, on the same two pins. i2c is a shared bus and the two do not
clash. Nothing at `68` or `69` means the GY-521 is not powered or its leads
are not on pins 3 and 5; the touch controller answering proves the pins
themselves work.

If the check reports a percentage of failed reads, the chip answers but keeps
dropping off the bus. That is a physical contact, not a setting: the failure
rate is the same at every read length and is no better with a repeated-START
transaction. It is also not the power supply, however much it correlates with
running off the battery. Find the lead:

```sh
python3 tools/i2cwatch.py
```

It reads WHO_AM_I at 20 Hz and prints a rolling success rate. Wiggle one lead
at a time, VCC then GND then SDA then SCL. The one that moves the number is
the bad one. A sound connection sits at 100% and does not move when you flex
the loom. On a GY-521 the usual cause is the pin header being pushed through
the board but never soldered.

Keep going after the first bad contact. On this build there were two in
series, a crimp in the Grove cable and the i2c hub, and each masked the other:
fixing one moved the number without ever reaching 100%, which sent days of
work into filtering and grounding that was never needed. Anything short of
100% means you are not done.

The check is on the *magnitude* of the acceleration vector, not on z alone.
Gravity is 1 g whichever way the board is facing, and tilt is computed from the
ratio between axes rather than their absolute size, so the magnitude is the one
number that means "this sensor works" at any mounting angle. A board resting on
its edge reads almost all of its gravity on x, and that is fine.

On this build it read 0.980 g with a bias of -3.15, +1.33, -0.61 deg/s, on the
bench on 13 Sep 2026. Both are normal for a GY-521. The 2% shortfall is
scale-factor error and washes out of any tilt calculation.

On 29 Sep 2026, fitted in the body, it read 1.235 g (z -1.233 g). That is out
of spec and not yet explained; see
[docs/troubleshooting.md](troubleshooting.md). One position cannot tell an
offset from a scale error. `tools/imucal.py` can:

```sh
sudo systemctl stop hexapod
python3 tools/imucal.py            # six positions; --axes z for upright and upside down only
sudo systemctl start hexapod
```

It walks you through turning the robot so gravity lies along each body axis in
turn, then prints each chip axis's offset and scale against the datasheet
tolerance, and the `imu.axis_map` that matches how the board is mounted.

Two things about the bias. It moves with temperature, so measure it after the
electronics have been running ten minutes, not from cold. And prefer averaging
it at startup while the robot is known to be still over hardcoding a constant.

**In the server.** [hexapod/imu.py](../hexapod/imu.py) reads the chip with the
same ioctl calls as the preflight, averages the gyro bias for `imu.bias_seconds`
when the server starts (keep the robot still), and subtracts it from every
reading. Pitch and roll come from the accelerometer, positive nose up and right
side down, smoothed by `imu.smoothing`. The state poller reads it at 10 Hz.

**Bus errors are retried.** Each i2c transaction gets `imu.bus_retries` extra
attempts 2 ms apart; the state poller rides out the ones that get through and
reopens the chip after five in a row. `/api/state` reports both counts:
`imu.read_errors` is every bus error seen, `imu.read_failures` only the ones
the retries did not clear.

A sound loom sits at zero. This one ran at 5 to 10% errors through bring-up
and that was two bad contacts, not noise; with them fixed it holds 13,639
reads at 100% on battery, screen on the same two pins. So the retries are
insurance against vibration, not a way to live with a loom that needs
fixing. Any sustained `read_errors` is a contact to go and find. A rising
`read_failures` is the same thing, worse, and is what `/status` complains
about.

`imu.axis_map` in [config/hexapod.yaml](../config/hexapod.yaml) says how the
chip is mounted. Each entry is the chip axis pointing along the body axis
(+X right, +Y forward, +Z up), with an optional minus sign. Set it once the
board is bolted down: `tools/imucal.py` prints it. Then tip the robot nose up
and check the horizon on `/status` moves down; lean it right and check the
horizon tilts. The horizon cannot show the robot upside down: tilt comes from
ratios, so pitch and roll stay inside ±90°, and a robot on its back reads close
to level. If the IMU is missing the server still starts and the
panel says so.
