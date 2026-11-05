# API and protocol

## API

| Method | Path | What |
|---|---|---|
| GET | `/` | Client page |
| GET | `/status` | Panel page |
| GET | `/api/state` | The state snapshot, once. Includes `angles` per leg and the 18 `pulses` last sent |
| GET | `/api/config` | Static: leg names, coxa positions, servo channels and directions, pulse clamp |
| WS | `/telemetry` | The state snapshot, pushed at 30 Hz, nothing accepted |
| WS | `/ws` | Commands in, state out. What the client page uses |
| GET | `/stream` | MJPEG live view |
| POST | `/capture` | Full-resolution still. Returns `{path, bytes}` |
| POST | `/move` | `{"direction": "forward", "speed": 0.5}` or `{"vx", "vy", "yaw"}` in -1..1 |
| POST | `/stop` | Latches e-stop. `/api/estop` is the same thing |

A `/move` command lives for 0.5 s. Keep posting or the robot halts; that is the
watchdog, not a bug. Directions are `forward`, `back`, `left`, `right`,
`turn_left`, `turn_right`, `stop`. `/move` only produces motion once the robot
is standing with torque on, so on a robot that has not been through bring-up
it logs the intent and nothing turns.

## Adding sensors

`Controller.command(source, velocity, priority)` takes velocity from any named source. Highest live priority wins, and a command expires after 0.5 s. The web UI is one source at priority 10. A vision behaviour is another call to the same method, no changes to the control loop.

## Protocol

Read off the firmware source in [EddieCarrera/chica-servo2040-simpleDriver](https://github.com/EddieCarrera/chica-servo2040-simpleDriver).

```
host -> board  SET   0xD3  start_idx  count  (lo hi) * count
host -> board  GET   0xC7  start_idx  count
board -> host  reply 0xC7  start_idx  count  (lo hi) * count
```

Values are 14 bit, split into two 7-bit bytes, low first. The command byte is the only byte with bit 7 set, which is how the parser resynchronises.

Channels: `0..17` servos, `18..23` touch sensors, `24` current, `25` voltage, `26` relay and torque enable, `27..28` spare GPIO.

Setting channel 26 to zero closes the relay and drops PWM on every servo. Servo positions are remembered while torque is off, so enabling torque snaps to the last commanded pose rather than to 1500 us.
