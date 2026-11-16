# Fujifilm camera protocol data

Notes on how Fujifilm X-series cameras talk over USB-C, Bluetooth and Wi-Fi:
reading and writing the C1–C7 custom settings (film simulation recipes), the
settings backup format, in-camera RAW conversion and pulling photos off the
camera.

I worked this out on my own X-T5 and figured it might save someone else a few
weekends. Each fact says whether I tested it on the camera or where it comes
from.

## Cameras

| Camera | Firmware | Notes |
|---|---|---|
| X-T5 | 4.31 | [x-t5.md](x-t5.md) |

Other X-series bodies probably work much the same way, but offsets, slot counts
and codes can differ, so check on your own camera before writing anything to it.

## Before you write to a camera

Restoring a settings file replaces *all* of the camera's settings, and a few
requests can leave it needing a restart (sometimes a battery pull). Keep the
backup you read before you write. Fujifilm considers third-party control
software outside warranty, so this is at your own risk.

## Contributing

Tested this on another body or firmware, or found a mistake? Issues and pull
requests are welcome. Please mention which camera and firmware you used.

## License

[CC BY 4.0](LICENSE). The sources section in each document credits the work
this builds on.
