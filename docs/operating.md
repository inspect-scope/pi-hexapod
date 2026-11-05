# Operating

The browser client, the roof panel, the camera and the safety layer.

## Web interface

- Live view top left, from the camera's low-resolution stream. **Capture** (or `C`) saves a full-resolution still and shows the file name.
- A 3D commanded-pose view sits next to the top-down stance. orbit / world / follow. drag to look around. green pads are closed foot switches.
- Left pad walks and turns like a car: up is forward, down is back, sideways
  turns, and a diagonal walks forward on a gentle curve. Right pad slides
  sideways without turning. Click the pad block to capture the keyboard:
  `W S` walk, `Q E` turn, `A D` slide sideways, `Esc` releases. Those keys do nothing until then. `J` twice jumps, same as the button.
- Space is e-stop from anywhere. So is the red button, and `POST /stop` if the page is wedged.
- Stand, Sit and Torque are separate. Sit before you cut torque. Jump is two clicks. Bounce loops until off.
- Gait, tricks and stance-mode dropdowns sit on that row. walks are the Chica set (tripod / triple / ripple / wave). tricks are flex, axis leans, spin (ripple in place) and dance. modes are normal / speed / offroad.
- The top-down view shows live foot positions. Green is on the ground, hollow blue is mid swing, yellow means that leg hit a joint limit.
- Posture and gait sliders tune ride height, body tilt, cycle time, step lift and top speed while it walks.
- The attitude tile shows pitch and roll from the IMU and the turn rates.

## Status panel

`/status` is built for the screen on the roof: battery voltage large enough to
read from across the room, servo current, the control URL to type into any
phone or laptop, an artificial horizon from the IMU, the robot's state, and a
STOP button. No other controls. The URL is the robot's own address with the
scheme and port this page was served on, so on the hotspot it reads
`https://10.42.0.1:8000`. STOP posts to `/stop`, which latches e-stop; clear it
from the client page.

A frame round the screen edge shows the six foot switches. Each leg owns the
stretch of edge on its side of the robot, green while its switch is closed,
grey when open, dashed while there is no data. `panel.front` in
[config/hexapod.yaml](../config/hexapod.yaml) says which screen edge faces the
robot's front; on this robot it is `right`, so L1 to L3 run along the top and
R1 to R3 along the bottom.

The screen is a 320x480 ST7796S over SPI, mounted on its side, so the page runs
at 480x320. [deploy/kiosk.sh](../deploy/kiosk.sh) rotates the output with
`wlr-randr --transform 90` before it starts Chromium, because Chromium sizes
itself to the output once and does not follow a later rotation. Set
`HEXAPOD_PANEL_TRANSFORM=270` if the panel is mounted the other way up, or
`normal` for a portrait screen. `HEXAPOD_PANEL_OUTPUT` overrides the connector
name if yours is not `SPI-1`.

The page has both layouts. Landscape puts the readings on the left and the
horizon on the right; portrait stacks them. It switches on the media query, so
the dry run in a browser window works either way.

The panel is Chromium in kiosk mode pointed at `https://localhost:8000/status`.
Same server, same stack, nothing else to maintain; see [Deploy](build.md#deploy).

## Camera

[hexapod/camera.py](../hexapod/camera.py) owns `Picamera2` and runs two streams
off the same sensor frames: a 640x360 one that the encoder turns into the
MJPEG live view, and the full 4608x2592 one that stills are saved from. Both
run all the time, so a capture never interrupts the stream. Sizes are in the
`camera` block of the config.

Stills go to `camera.survey_dir/<server start time>/<capture time>.jpg`, one
directory per server run, with a `.json` beside each holding the full state
snapshot (voltage, pitch, roll, position of every foot) at the moment of
capture. `POST /capture` returns the path.

The live view is MJPEG over `multipart/x-mixed-replace`. It is fine on a LAN or
a tether. Over a bad link WebRTC would do better; that is not built.

The camera is mounted upside down, so `camera.rotation` is `180` and both
streams get a libcamera `Transform(hflip=1, vflip=1)`. Only 0 and 180 are
allowed; a sensor can flip but not transpose. The device tree can do the same
job for every tool on the Pi, but note the `imx708` overlay's default is
already `rotation=180`, so the flip there is `rotation=0`. Use one or the
other, not both; two flips cancel.

Pi 4 and earlier encode the stream in hardware. Pi 5 has no hardware JPEG
encoder, so it uses a software one on an RGB low-resolution stream; the module
picks based on the platform. The full-resolution stream costs about 35 MB of
CMA per buffer (`camera.buffers`, default 2). If the camera fails to start with
an allocation error, lower `still` or add `cma=320M` to the kernel command line.

## Safety

- `limits.pulse_us` is the hard clamp. Nothing reaches a servo outside it.
- A velocity command expires 0.5 s after it arrives, whether it came over `/ws` or `/move`. A dropped websocket, a frozen page or a client that stops posting means the robot stops walking. In a steel tank the link is the thing most likely to fail.
- The board's IO thread cuts torque if it goes `control.watchdog_ms` without a fresh servo frame. A hung control loop or a crashed server parks the robot.
- E-stop latches. Clear it before the robot will stand again.
- **Trips.** The board reports one total current for all 18 servos and the
  pack voltage. Its IO thread latches the e-stop, torque on, when the total's
  mean over `safety.current_cut_s` (1 s) exceeds `current_cut_a` (10 A), when
  pulses have been still for `still_s` (1 s) and the mean exceeds `sit_cut_a`
  (1.5 A, sat / legs free) or `stand_cut_a` (5.5 A, standing still), when
  the voltage's mean over `volts_cut_s` (2 s) drops under `volts_cut` (6.0 V),
  or when telemetry stops for 2 s. That covers `hexapod serve`, `jog` and
  `neutral`. The reason shows as the status line and as `safety_trip` in
  `/api/state` until you clear the e-stop (`clear` in jog). The 10 A ceiling
  still needs three simultaneous stalls. The settled sit/stand cuts catch
  one hung servo at a held pose, which is how the femurs died. Walking
  (pulses moving) only uses the 10 A ceiling. The total cannot name the
  servo; run `poke.py --probe` afterwards. The board also refuses torque
  while it is offline, and does not replay an earlier request when the port
  comes back. See [docs/troubleshooting.md](troubleshooting.md).
- Body shift plus tilt at low ride height can push a leg past its joint limits. The UI shows that leg in yellow and `hexapod check` prints how often it happens.
- If your servo supply is above 5 V, cut the *Separate USB and Ext. Power* trace on the back of the Servo2040 first.
