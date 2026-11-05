# Contributing

## Run it

```sh
python3 -m venv .venv && .venv/bin/pip install -e '.[dev]'
.venv/bin/pytest
.venv/bin/hexapod --dry-run serve
```

The tests and the dry run need no hardware. CI runs the tests on Python 3.9,
3.11 and 3.13.

## What a change needs

- Tests for new behaviour. `FakeBoard`, `FakeImu` and `FakeCamera` stand in
  for the hardware.
- Anything that moves a servo goes through `limits.pulse_us` and the
  controller. Nothing writes pulses around them.
- Safety thresholds in [config/hexapod.yaml](config/hexapod.yaml) stay
  conservative. Raise one only with a measurement behind it.
- A hardware problem you solved goes in
  [docs/troubleshooting.md](docs/troubleshooting.md): symptom, cause, fix. If
  an earlier entry turns out wrong, keep it and say what replaced it.
- The client is one HTML file with no build step and no CDN. Keep it that way.

## Hardware changes

Different servos, a different screen or a different body are welcome. Put the
differences in config, not in code, where you can. Open a build report issue
with photos so others can find it.
