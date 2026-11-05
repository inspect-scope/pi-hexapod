# A Raspberry Pi brain for a six-legged robot

By Antoine Boxho and Fin Warman

[VIDEO: youtube link]

We built a MakeYourPet Chica hexapod and replaced the two Android apps that
drive it with one Python process on a Raspberry Pi 5. The Pimoroni Servo2040
that drives the 18 servos keeps its stock firmware. Everything above it now
runs on the Pi: inverse kinematics, gaits, safety trips, the camera, the IMU, a
web client and a status screen on the robot's roof.

The code, the printable body, the config and a 1,000-line bring-up log are at
[github.com/C00kieMonsta/pi-hexapod](https://github.com/C00kieMonsta/pi-hexapod)
under the MIT license. You can run the whole stack on a laptop with no robot
attached.

## What the robot does

- Walks with the Chica gait set: tripod, triple, ripple and wave. Three stance
  modes (normal, speed, offroad) and tricks: flex, axis leans, spin, dance,
  bounce and jump.
- Drives from any browser on the LAN. The left pad steers like a car, the
  right pad slides sideways. Once you click the pad, W and S walk, Q and E
  turn, A and D slide. Space is e-stop from anywhere on the page.
- Shows the commanded pose in 3D, and a top-down view of the feet.
- Streams the robot camera as MJPEG and saves full-resolution stills, each with
  a JSON file of the robot's state at that moment.
- Senses ground contact through a micro switch in each foot. A stance foot
  that has not touched down is lowered up to 15 mm until its switch closes.
- Follows your hand through a laptop camera. In lean mode the palm moves the
  body. In pincer mode your index and middle fingers drive the two front legs.
- Lets you pose a leg from the 3D view, one joint at a time or by grabbing a
  foot or a knee. Joints slew at 40 deg/s toward the target, so a grab never
  makes a servo jump.
- Runs a 480x320 status page on the roof screen: battery voltage, servo
  current, the URL to open on your phone, a horizon from the GY-521 IMU, the six
  foot switches and a STOP button.

## How it fits together

```
browser on the LAN  --wifi-->  Pi  --USB serial-->  Servo2040  -->  18 servos
joystick, sliders   /          |   --i2c-------->  GY-521 IMU
480x320 panel   ---            |   --CSI--------->  camera
                        one FastAPI process: IK, gait, state, MJPEG
```

One process owns the hardware. The serial port, the i2c bus and the camera can
each be opened by one process at a time, so `hexapod serve` holds all three.
The operator page and the roof panel are plain HTTP clients of it.

The control loop runs at 50 Hz on its own thread, not on the web server's event
loop, so a stalled websocket cannot stall the robot. Velocity commands come
from named sources, each with a priority, and expire after 0.5 s. The web page
is one source. A vision behaviour would be another. If the page freezes or the wifi drops, the command
expires and the robot stops walking. One layer down, the board's IO thread cuts
torque if the control loop stops sending servo frames.

We read the Servo2040 protocol off the firmware source in
[EddieCarrera/chica-servo2040-simpleDriver](https://github.com/EddieCarrera/chica-servo2040-simpleDriver).
Values are 14 bits split into two 7-bit bytes. The command byte is the only
byte with bit 7 set, which is how the parser resynchronises. The codec is
[hexapod/protocol.py](../hexapod/protocol.py): pure functions, no IO.

Hand tracking runs in the browser. MediaPipe's WASM hand landmarker finds 21
points per frame and posts them to the server.
[hexapod/hand.py](../hexapod/hand.py) maps them onto a body pose inside the
operator limits: 15 mm of shift and 8 degrees of roll, pitch and yaw. Browsers
only give camera access to https pages or localhost, so `serve` mints a
self-signed certificate.

## What went wrong during bring-up

We kept a log of every hardware problem in
[docs/troubleshooting.md](troubleshooting.md). When a diagnosis turned out wrong, we left it
in the file next to the one that replaced it. These are the five entries we
would want to read before building one.

### The i2c fault we misdiagnosed for weeks

The IMU kept dropping off the bus with `Errno 121`. Success rates moved between
93% and 100% depending on the configuration, and collapsed to 0% within about
15 seconds when the Pi ran from the battery's UBEC. We blamed switching noise
from the UBEC, wrote that down as fact, and planned to buy a 2S-to-USB-C buck
module.

The cause was two bad contacts in series: the VCC crimp in the Grove cable,
and an i2c hub. Either one alone is intermittent. Together they masked each
other. Every fix moved the number without reaching 100%, so each one looked
like a partial fix that needed one more change on top: filtering, extra
grounds, a slower bus, a capacitor. Each of those changes involved unplugging
something, which reseated the bad crimp and made the change look like it
helped.

With a new cable and the hub out of the loom, the bus held 13,639 reads at
100% on battery, with the screen's touch controller on the same two pins. The
log keeps a table of the six conclusions this retracts.

[tools/i2cwatch.py](../tools/i2cwatch.py) finds this kind of fault. It reads the
IMU at 20 Hz and prints a rolling success rate while you wiggle one lead at a
time. The lesson we took: a fix that only moves the number means you have not
found the fault.

### Left and right legs did opposite things

Pressing Torque or Stand made one side tuck up while the other stretched out
flat. [tools/servolog.py](../tools/servolog.py) showed every leg receiving the
same pulse. Left and right legs are mirror images, so the same shaft rotation
moves them opposite ways.

Chica's `chica-config-2040.txt` has no direction field. The Android app applied
the mirror itself, from the L or R in each servo's name. We copied the pin
numbers and calibration values and missed that behaviour. The fix was
`direction: -1` on the femur and tibia of one side.

The config ended up with `-1` on all three joints of the right legs, hip
included. The robot stood and sat correctly. On its first walk on the floor,
pushing the stick to spin right made it walk forward. Simulating a wrong sign
on each joint group pointed at the three right hips. The coxa part is the same
print on all six legs, rotated around the body, so the hip does not mirror. A
hip with the wrong sign is invisible at sit and stand, because the coxa barely
moves between the two. It only shows when walking.

### Three femur servos died on one day

On 26 Sep 2026 we lost three femur servos while the robot sat on blocks with
its legs in the air. One was already silent when first probed. One peaked at
4.4 to 5.4 A over five probe runs while the others read 1.3 to 3.7 A, then
died. The third read normal in a sweep. About 40 minutes later the operator saw
the total current pass 10 A, and about a minute after that it was dead.

We still do not know why. The log lists five hypotheses with the evidence for
each, and the claims we later retracted.

One fact holds for all three. Nothing in our stack acted on current or
voltage. Chica's config has warning thresholds for both, but the Android app
enforced them. The Servo2040 firmware only reports current and voltage when
asked. Replacing the app with the Pi dropped the cuts.

The board's IO thread now latches e-stop on a total over 10 A, 1.5 A at a held
sit, 5.5 A at a held stand, under 6.0 V, or 2 s without telemetry. The board
reports one total for all 18 servos, so [tools/poke.py](../tools/poke.py)
`--probe` wiggles one header at a time and watches the current sensor. It
flagged one of the dead servos as the hungriest on every run before it failed.
It read another as normal about 40 minutes before it died. A clean probe does
not clear a servo.

### The relay never opened

Every e-stop and current trip ends in "torque off", which should open the relay
that feeds the servo rail. A meter on the Servo2040's power terminals read
7.79 V with torque on and with torque off. The relay module's LEDs never
changed.

The Servo2040 drove its control pin correctly: 0.22 V for off, 3.3 V for on.
The module was set to low trigger, and the RP2040's 3.3 V high was not enough
to switch a 5 V low-trigger module off. The relay had been closed the whole
time, and torque off only stopped the servo PWM. Moving the module's H/L jumper
to H fixed it: the rail then read 0.08 V with torque off and 7.63 V with torque
on.

### Two processes on one serial port

Before we added a lock, two processes could open the serial port at once and
both send servo frames. On the robot that looks like random twitching. The port is now opened exclusively, so a
second opener fails at once with "Could not exclusively lock port".

That lock exposed a second bug. The server treated a missing or locked board as
fatal. While `poke.py --centre` held the port, systemd restarted the service
every 2 seconds, and it crash-looped eleven times behind the operator. The
board is now optional at startup: the status line says it is offline, the
server retries every 5 seconds, and the board refuses torque until it is back.

## Try it without a robot

```sh
git clone https://github.com/C00kieMonsta/pi-hexapod && cd pi-hexapod
python3 -m venv .venv && .venv/bin/pip install -e '.[dev]'
tools/fetch-hand-assets.sh
.venv/bin/hexapod --dry-run serve
```

Open the URL it prints and accept the self-signed certificate. `/` is the
operator page and `/status` is the roof panel. The dry run fakes the board, a
rocking IMU and a camera, so every page and endpoint works. Allow the laptop
camera to try hand following. The test suite also runs with nothing attached:
`.venv/bin/pytest`.

## Build your own

The parts we used:

- MakeYourPet Chica frame and legs, printed
- Pimoroni Servo2040 running the stock Chica firmware
- 18 Feetech FT5330M servos
- Raspberry Pi 5
- GY-521 (MPU-6050) IMU
- Raspberry Pi Camera Module 3 Wide
- Waveshare 3.5" RPi LCD (F), an ST7796S panel mounted sideways
- 2S LiPo, a UBEC set to 5.2 V for the Pi, and a 5 V relay module on the servo
  rail
- A micro switch in each foot

[TODO: total parts cost]
[TODO: print time and filament for the body and legs]

[assets/print-me/](../assets/print-me/) has the leg parts and a stacked
electronics body that sits on the Chica frame: a battery floor, a Pi and
Servo2040 floor, and a roof that holds the screen. A CadQuery script generates
the body, so you can edit its parameters and regenerate every STL.

Servo channels, calibration, geometry and safety thresholds live in
[config/hexapod.yaml](../config/hexapod.yaml). If you already calibrated your
servos with Chica, the two microsecond values per servo copy straight across.
[tools/preflight.py](../tools/preflight.py) checks the environment, config,
IMU, camera and board in one run. [docs/build.md](build.md) walks through
bring-up in order, from finding the board to the first walk.

## What is not done

- A USB drop mid-session is not reconnected. The board is only retried until
  it first opens.
- Walking current has not been measured. The relay's contacts are rated 7 A
  resistive, below the 10 A trip.
- In the body, the IMU read 1.235 g where it should read 1.000 g. The cause is
  open. [tools/imucal.py](../tools/imucal.py) runs a six-position check that
  tells an offset from a scale fault.
- WebRTC for the live view over a weak link is not built. It is MJPEG.

## Credits

- [MakeYourPet](https://github.com/MakeYourPet/hexapod) designed the Chica
  frame and legs and published the STLs and `chica-config-2040.txt` under MIT.
  Our servo channels and calibration are ported from that file.
- [Eddie Carrera's chica-servo2040-simpleDriver](https://github.com/EddieCarrera/chica-servo2040-simpleDriver)
  is the Servo2040 firmware we run unchanged, and the source of the serial
  protocol. MIT.
- [MediaPipe](https://github.com/google-ai-edge/mediapipe) provides the hand
  landmarker and its WASM runtime. Apache-2.0.

Issues and pull requests are welcome. If you build one, we would like to see
it.
