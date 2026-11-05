# Third-party notices

This repo includes or derives from the work below. Each keeps its own license.

## MakeYourPet hexapod

https://github.com/MakeYourPet/hexapod, MIT.

- `assets/print-me/legs/` holds unmodified copies of `left-tibia.stl`,
  `right-tibia.stl`, `limiter.stl`, `micro-switch-tip.stl`,
  `servo-back-hole2.stl`, `shield.stl` and `tip.stl` from its `STL/` folder.
- `assets/print-me/body/chica_body.py` is drawn around its `frame.stl` and
  `top-cover4.stl`.
- `config/hexapod.yaml` ports servo channels, calibration pulses and frame
  geometry from its `chica-config-2040.txt`.

```
MIT License

Copyright (c) 2022 MakeYourPet.com

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

## chica-servo2040-simpleDriver

https://github.com/EddieCarrera/chica-servo2040-simpleDriver, MIT.

The Servo2040 runs this firmware unchanged. `hexapod/protocol.py` implements
its serial wire format, read off `chica-servo2040/chica-servo2040.cpp`.

```
MIT License

Copyright (c) 2023 Eddie Carrera

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

## MediaPipe Tasks Vision

https://github.com/google-ai-edge/mediapipe, Apache-2.0.

`hexapod/static/vendor/mediapipe/` vendors the `@mediapipe/tasks-vision`
0.10.32 JS bundle unmodified. `tools/fetch-hand-assets.sh` downloads its wasm
and the `hand_landmarker.task` model, which are not committed. The license text
is in [hexapod/static/vendor/mediapipe/LICENSE](hexapod/static/vendor/mediapipe/LICENSE).
