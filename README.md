# v4l2py

[![V4L2py][pypi-version]](https://pypi.python.org/pypi/v4l2py)
[![Python Versions][pypi-python-versions]](https://pypi.python.org/pypi/v4l2py)
![License][license]
[![CI][CI]](https://github.com/tiagocoutinho/v4l2py/actions/workflows/ci.yml)

Video for linux 2 (V4L2) python library

A two purpose API:

* high level Device API for humans to play with :-)
* raw python binding for the v4l2 (video4linux2) userspace API, using ctypes (don't even
  bother wasting your time here. You probably won't use it)

Only works on python >= 3.7.


## Installation

From within your favorite python environment:

```bash
$ pip install v4l2py
```

## Usage

Without further ado:

```python
>>> from v4l2py import Device
>>> with Device.from_id(0) as cam:
>>>     for i, frame in enumerate(cam):
...         print(f"frame #{i}: {len(frame)} bytes")
...         if i > 9:
...             break
...
frame #0: 54630 bytes
frame #1: 50184 bytes
frame #2: 44054 bytes
frame #3: 42822 bytes
frame #4: 42116 bytes
frame #5: 41868 bytes
frame #6: 41322 bytes
frame #7: 40896 bytes
frame #8: 40844 bytes
frame #9: 40714 bytes
frame #10: 40662 bytes
```

Getting information about the device:

```python
>>> from v4l2py.device import Device, BufferType

>>> cam = Device.from_id(0)
>>> cam.open()
>>> cam.info.card
'Integrated_Webcam_HD: Integrate'

>>> cam.info.capabilities
<Capability.STREAMING|EXT_PIX_FORMAT|VIDEO_CAPTURE: 69206017>

>>> cam.info.formats
[ImageFormat(type=<BufferType.VIDEO_CAPTURE: 1>, description=b'Motion-JPEG',
             flags=<ImageFormatFlag.COMPRESSED: 1>, pixelformat=<PixelFormat.MJPEG: 1196444237>),
 ImageFormat(type=<BufferType.VIDEO_CAPTURE: 1>, description=b'YUYV 4:2:2',
             flags=<ImageFormatFlag.0: 0>, pixelformat=<PixelFormat.YUYV: 1448695129>)]

>>> cam.get_format(BufferType.VIDEO_CAPTURE)
Format(width=640, height=480, pixelformat=<PixelFormat.MJPEG: 1196444237>}

>>> for ctrl in cam.controls.values(): print(ctrl)
<IntegerControl brightness min=0 max=255 step=1 default=128 value=128>
<IntegerControl contrast min=0 max=255 step=1 default=32 value=32>
<IntegerControl saturation min=0 max=100 step=1 default=64 value=64>
<IntegerControl hue min=-180 max=180 step=1 default=0 value=0>
<BooleanControl white_balance_automatic default=True value=True>
<IntegerControl gamma min=90 max=150 step=1 default=120 value=120>
<MenuControl power_line_frequency default=1 value=1>
<IntegerControl white_balance_temperature min=2800 max=6500 step=1 default=4000 value=4000 flags=inactive>
<IntegerControl sharpness min=0 max=7 step=1 default=2 value=2>
<IntegerControl backlight_compensation min=0 max=2 step=1 default=1 value=1>
<MenuControl auto_exposure default=3 value=3>
<IntegerControl exposure_time_absolute min=4 max=1250 step=1 default=156 value=156 flags=inactive>
<BooleanControl exposure_dynamic_framerate default=False value=False>

>>> cam.controls["saturation"]
<IntegerControl saturation min=0 max=100 step=1 default=64 value=64>

>>> cam.controls["saturation"].id
9963778
>>> cam.controls[9963778]
<IntegerControl saturation min=0 max=100 step=1 default=64 value=64>

>>> cam.controls.brightness
<IntegerControl brightness min=0 max=255 step=1 default=128 value=128>
>>> cam.controls.brightness.value = 64
>>> cam.controls.brightness
<IntegerControl brightness min=0 max=255 step=1 default=128 value=64>
```

(see also examples/v4l2py-ctl.py)

### asyncio

v4l2py is asyncio friendly:

```bash
$ python -m asyncio

>>> from v4l2py import Device
>>> with Device.from_id(0) as camera:
...     async for frame in camera:
...         print(f"frame {len(frame)}")
frame 10224
frame 10304
frame 10224
frame 10136
...
```

(check examples/basic_async.py)

### gevent

v4l2py is also gevent friendly:

```
$ python

>>> from v4l2py import Device, GeventIO
>>> with Device.from_id(0, io=GeventIO) as camera:
...     for frame in camera:
...         print(f"frame {len(frame)}")
frame 10224
frame 10304
frame 10224
frame 10136
...
```

(check examples/basic_gevent.py and examples/web/app.py)

## Bonus track

You've been patient enough to read until here so, just for you,
a 20 line gem: a flask web server displaying your device on the web:

```bash
$ pip install flask
```

```python
# web.py

import flask
from v4l2py import Device

app = flask.Flask('basic-web-cam')

def gen_frames():
    with Device.from_id(0) as cam:
        for frame in cam:
            yield b"--frame\r\nContent-Type: image/jpeg\r\n\r\n" + frame.data + b"\r\n"

@app.route("/")
def index():
    return '<html><img src="/stream" /></html>'

@app.route("/stream")
def stream():
    return flask.Response(
        gen_frames(), mimetype='multipart/x-mixed-replace; boundary=frame')
```

run with:

```bash
$ FLASK_APP=web flask run -h 0.0.0.0
```

Point your browser to [127.0.0.1:5000](http://127.0.0.1:5000) and you should see
your camera rolling!

## Migrating from 1.x to 2

A frame changed from a simple bytes object to a Frame which contains
the data plus all frame metadata.

As a consequence, when migrating from 1.x to 2, you will need to cast
frame object with `bytes` or access the `frame.data` item:

Before:

```python
with Device.from_id(0) as cam:
    for frame in cam:
        buff = io.BytesIO(frame)
```

Now:

```python
with Device.from_id(0) as cam:
    for frame in cam:
        frame = bytes(frame)  # or frame = frame.data
        buff = io.BytesIO(frame)
```

## Improved device controls

Device controls have been improved to provide a more pythonic interface. The
new interface is the default now; however, the legacy interface can be
requested: `Device.from_id(x, legacy_controls=True)`.

Before:
```python
>>> from v4l2py import Device
>>> cam = Device.from_id(0)
>>> cam.open()
>>> for ctrl in cam.controls.values():
...     print(ctrl)
...     for item in ctrl.menu.values():
...             print(f" - {item.index}: {item.name}")
<Control brightness type=integer min=0 max=255 step=1 default=128 value=255>
<Control contrast type=integer min=0 max=255 step=1 default=32 value=255>
<Control saturation type=integer min=0 max=100 step=1 default=64 value=100>
<Control hue type=integer min=-180 max=180 step=1 default=0 value=0>
<Control white_balance_automatic type=boolean min=0 max=1 step=1 default=1 value=1>
<Control gamma type=integer min=90 max=150 step=1 default=120 value=150>
<Control gain type=integer min=1 max=7 step=1 default=1 value=1>
<Control power_line_frequency type=menu min=0 max=2 step=1 default=2 value=2>
 - 0: Disabled
 - 1: 50 Hz
 - 2: 60 Hz
<Control white_balance_temperature type=integer min=2800 max=6500 step=1 default=4000 value=4000 flags=inactive>
<Control sharpness type=integer min=0 max=7 step=1 default=2 value=7>
<Control backlight_compensation type=integer min=0 max=1 step=1 default=0 value=1>
<Control auto_exposure type=menu min=0 max=3 step=1 default=3 value=3>
 - 1: Manual Mode
 - 3: Aperture Priority Mode
<Control exposure_time_absolute type=integer min=10 max=333 step=1 default=156 value=156 flags=inactive>
<Control exposure_dynamic_framerate type=boolean min=0 max=1 step=1 default=0 value=1>

>>> type(cam.controls.exposure_dynamic_framerate.value)
<class 'int'>
```

Now:
```python
>>> from v4l2py.device import Device, MenuControl
>>> cam = Device.from_id(0)
>>> cam.open()
>>> for ctrl in cam.controls.values():
...     print(ctrl)
...     if isinstance(ctrl, MenuControl):
...             for (index, name) in ctrl.items():
...                     print(f" - {index}: {name}")
<IntegerControl brightness min=0 max=255 step=1 default=128 value=255>
<IntegerControl contrast min=0 max=255 step=1 default=32 value=255>
<IntegerControl saturation min=0 max=100 step=1 default=64 value=100>
<IntegerControl hue min=-180 max=180 step=1 default=0 value=0>
<BooleanControl white_balance_automatic default=True value=True>
<IntegerControl gamma min=90 max=150 step=1 default=120 value=150>
<IntegerControl gain min=1 max=7 step=1 default=1 value=1>
<MenuControl power_line_frequency default=2 value=2>
 - 0: Disabled
 - 1: 50 Hz
 - 2: 60 Hz
<IntegerControl white_balance_temperature min=2800 max=6500 step=1 default=4000 value=4000 flags=inactive>
<IntegerControl sharpness min=0 max=7 step=1 default=2 value=7>
<IntegerControl backlight_compensation min=0 max=1 step=1 default=0 value=1>
<MenuControl auto_exposure default=3 value=3>
 - 1: Manual Mode
 - 3: Aperture Priority Mode
<IntegerControl exposure_time_absolute min=10 max=333 step=1 default=156 value=156 flags=inactive>
<BooleanControl exposure_dynamic_framerate default=False value=True>

>>> type(cam.controls.white_balance_automatic.value)
<class 'bool'>
>>> cam.controls.white_balance_automatic.value
<BooleanControl white_balance_automatic default=True value=True>
>>> cam.controls.white_balance_automatic.value = False
<BooleanControl white_balance_automatic default=True value=False>

>>> wba = cam.controls.white_balance_automatic
>>> wba.value = "enable"    # or "on", "1", "true", "yes"
>>> wba
<BooleanControl white_balance_automatic default=True value=True>
>>> wba.value = "off"       # or "disable", "0", "false", "no"
>>> wba
<BooleanControl white_balance_automatic default=True value=False>
```

The initial upgrade path for existing code is to request the legacy interface
by passing `legacy_controls=True` when instantiating the `Device` object, use
`LegacyControl` instead of `Control` for instantiations, and `BaseControl`
for isinstance() checks. And in the unlikely case your code does isinstance()
checks for `MenuItem`, these should be changed to `LegacyMenuItem`.

## References

See the ``linux/videodev2.h`` header file for details.


* `Video for Linux Two Specification <http://linuxtv.org/downloads/v4l-dvb-apis/ch07s02.html>`

[pypi-python-versions]: https://img.shields.io/pypi/pyversions/v4l2py.svg
[pypi-version]: https://img.shields.io/pypi/v/v4l2py.svg
[pypi-status]: https://img.shields.io/pypi/status/v4l2py.svg
[license]: https://img.shields.io/pypi/l/v4l2py.svg
[CI]: https://github.com/tiagocoutinho/v4l2py/actions/workflows/ci.yml/badge.svg
