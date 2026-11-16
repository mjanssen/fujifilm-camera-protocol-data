# Fujifilm X-T5: recipes, backups, photos and RAW conversion over USB, Bluetooth and Wi-Fi

A technical reference for how a **Fujifilm X-T5** exposes its custom settings
(C1–C7 "recipes"), its photos and its in-camera RAW conversion over USB-C,
Bluetooth LE and Wi-Fi, for people and coding agents implementing a client.

> **Scope.** Everything here was worked out and tested on **one X-T5 (firmware
> 4.31, IOP code `FF179502`)**, from iOS and macOS through Apple's
> ImageCaptureCore, CoreBluetooth and Network.framework. Other X-series bodies
> share much of it, but byte offsets, slot counts and codes can differ. Test on
> your own body before writing to it.
>
> Each fact is marked **(verified)** when it was confirmed on the X-T5, or
> cites its source when it comes from documentation and wasn't tested.

> **Risk.** Writing a settings file replaces *all* of the camera's settings. A
> conversion or Wi-Fi request the camera doesn't answer can leave it needing a
> restart (sometimes a battery pull). Always keep the backup you read before
> writing. Fujifilm treats unlicensed control software as outside warranty. Use
> this at your own risk.

## Contents

- [Quick reference: where each setting lives](#quick-reference-where-each-setting-lives)
  - [Film simulations](#film-simulations)
  - [Recipe settings](#recipe-settings)
- [1. The settings backup file](#1-the-settings-backup-file)
- [2. USB: PTP in "USB RAW CONV./BACKUP RESTORE" mode](#2-usb-ptp-in-usb-raw-convbackup-restore-mode)
  - [2.1 Reading and restoring the backup](#21-reading-and-restoring-the-backup)
  - [2.2 In-camera RAW conversion](#22-in-camera-raw-conversion)
- [3. Bluetooth LE](#3-bluetooth-le)
  - [3.1 Pairing and session setup](#31-pairing-and-session-setup)
  - [3.2 Reading and restoring the backup](#32-reading-and-restoring-the-backup)
  - [3.3 Opening the camera's Wi-Fi](#33-opening-the-cameras-wi-fi)
- [4. Wi-Fi: photo browsing and import](#4-wi-fi-photo-browsing-and-import)
- [5. Recipes in photo metadata (maker note)](#5-recipes-in-photo-metadata-maker-note)
- [6. Things that hang or break the camera](#6-things-that-hang-or-break-the-camera)
- [7. Sources and credits](#7-sources-and-credits)

---

## Quick reference: where each setting lives

The same recipe setting is stored differently in four places:

- **Backup:** the camera's settings backup file (§1). Offsets are relative to
  a slot's record; slot *n* (0 = C1) starts at `0x7B7C + n × 0x400`.
- **RAW conversion:** the RAW-conversion profile, PTP property `0xD185` (§2.2).
  These are byte offsets into the profile; each cell is a signed 32-bit
  little-endian integer.
- **Photo:** the maker note the camera writes into each photo (§5).

### Film simulations

| Film simulation | Backup byte at `+0x0D0` | RAW conversion cell `0x21D` | Photo: `0x1401` FilmMode / `0x1003` Saturation |
|---|---|---|---|
| Provia / Standard | `0x01` | 1 | `0x1401` = `0x000` |
| Velvia / Vivid | `0x04` | 2 | `0x1401` = `0x200` (also `0x400`)¹ |
| Astia / Soft | `0x02` | 3 | `0x1401` = `0x120`¹ |
| Classic Chrome | `0x0F` | 11 (`0x0B`) | `0x1401` = `0x600` |
| Reala Ace | `0x19` | 20 (`0x14`) | `0x1401` = `0xB00`¹ |
| PRO Neg. Hi | `0x0E` | 4 | `0x1401` = `0x501`¹ |
| PRO Neg. Std | `0x0D` | 5 | `0x1401` = `0x500`¹ |
| Classic Neg. | `0x10` | 17 (`0x11`) | `0x1401` = `0x800` |
| Nostalgic Neg. | `0x11` | 19 (`0x13`) | `0x1401` = `0xA00`¹ |
| Eterna / Cinema | `0x12` | 16 (`0x10`) | `0x1401` = `0x700`¹ |
| Eterna Bleach Bypass | `0x13` | 18 (`0x12`) | `0x1401` = `0x900`¹ |
| ACROS | `0x15` | 12 (`0x0C`) | `0x1003` = `0x500`¹ |
| ACROS + Ye | `0x17` | 13 (`0x0D`) | `0x1003` = `0x502`¹ |
| ACROS + R | `0x16` | 14 (`0x0E`) | `0x1003` = `0x501` |
| ACROS + G | `0x18` | 15 (`0x0F`) | `0x1003` = `0x503`¹ |
| Monochrome | `0x09` | 6 | `0x1003` = `0x300`¹ |
| Monochrome + Ye | `0x0B` | 7 | `0x1003` = `0x302`¹ |
| Monochrome + R | `0x0A` | 8 | `0x1003` = `0x301`¹ |
| Monochrome + G | `0x0C` | 9 | `0x1003` = `0x303`¹ |
| Sepia | `0x06` | 10 (`0x0A`) | `0x1003` = `0x310`¹ |

All backup codes were verified on the X-T5. RAW conversion codes are from the
fujifilm-ptp-reference and libfuji; Provia (1) and Classic Chrome (11) were
verified by rendering. ¹ = photo code from exiftool, not seen in X-T5 output;
the others were verified. For B&W films, photos leave `0x1401` out and record
the film in `0x1003`.

**B&W variants in the backup (verified):** besides `+0x0D0`, a slot remembers
the last Monochrome variant at `+0x0D6` and the last ACROS variant at `+0x0D3`.
When writing a Monochrome film, also write its code to `+0x0D6`. Otherwise the
camera can keep the remembered variant (writing Mono `0x09` while `+0x0D6` held
`0x0C` produced Mono + G). Do the same with `+0x0D3` for ACROS.

### Recipe settings

| Setting | Backup (record offset, encoding) | RAW conversion (offset, encoding) | Photo tag (encoding) |
|---|---|---|---|
| Film simulation | `0x0D0`, see table above | `0x21D` | `0x1401` / `0x1003` |
| Dynamic range | `0x0E0`: 0 Auto, 1 DR100, 2 DR200, 3 DR400 | `0x215`: 100 / 200 / 400 | `0x1402` (0 auto, 1 manual) + `0x1403` (100/200/400) |
| D-Range Priority | `0x0E2`: 0 Auto, 2 Strong, 3 Weak, 4 Off | `0x219` (reference advises not writing it) | `0x1443` (0 auto, 1 fixed) + `0x1445` (1 weak, 2 strong) |
| White balance mode | `0x06C`: see [WB modes](#white-balance) | `0x229` source (1 as shot, 2 use `0x22D`) + `0x22D` mode | `0x1002` |
| Kelvin | `0x010`, u16 LE | `0x239`, K | `0x1005` |
| WB shift R / B | per-mode pair table, byte = 9 − value (see [WB shift](#white-balance-shift)) | `0x231` / `0x235`, −9…+9 | `0x100A`: two int32, value × 20 |
| Highlight | `0x0E7`: byte = 2·value + 4 (−2…+4, half steps) | `0x23D`: value × 10 | `0x1041`: value × −16 |
| Shadow | `0x0E9`: byte = 2·value + 4 | `0x241`: value × 10 | `0x1040`: value × −16 |
| Color | `0x0E3`: byte = 7 − value; `0x00` = none (B&W) | `0x245`: value × 10 | `0x1003`: see [Color](#color-sharpness-and-noise-reduction-in-photos) |
| Sharpness | `0x0E5`: byte = 4 − value | `0x249`: value × 10 | `0x1001` |
| High ISO NR | `0x0A6`: byte = value + 4 | `0x24D`: lookup (see §2.2) | `0x100E` |
| Clarity | `0x0A9`: byte = value + 6 | `0x269`: value × 10 | `0x100F`: value × 1000 |
| Grain effect | `0x0ED`: 0 Strong, 1 Weak, 2 Off | `0x221`, combined: 1 off, 2 weak small, 3 strong small, 4 weak large, 5 strong large | `0x1047`: 0 off, 32 weak, 64 strong |
| Grain size | `0x0EE`: 0 Small, 1 Large | (in `0x221`) | `0x104C`: 16 small, 32 large (0 when off) |
| Color Chrome | `0x0EB`: 0 Off, 1 Weak, 2 Strong | `0x225`: 1 off, 2 weak, 3 strong | `0x1048`: 0 / 32 / 64 |
| Color Chrome FX Blue | `0x0EC`: 0 / 1 / 2 | `0x261`: 1 / 2 / 3 | `0x104E`: 0 / 32 / 64 |
| Smooth Skin | `0x0EF`: 0 / 1 / 2 | `0x25D`: 1 / 2 / 3 | none in exiftool's table (untested) |
| Mono warm/cool | `0x0DC`: byte = 0x12 − value (−18…+18) | `0x259`: value × 10 | `0x1049`: signed byte |
| Mono magenta/green | `0x0DE`: byte = 0x12 − value | `0x265`: value × 10 | `0x104B`: signed byte |
| Recipe name | `0x253`: ASCII, NUL-terminated, 25 bytes max | – | – |
| Exposure (conversion only) | – | `0x211`: EV × 1000 in exact thirds (±333, ±667, ±1000 …) | – |

Backup offsets were all verified by calibration (change one setting on the
camera, read the backup, compare). The documentation for the backup format
historically used offsets 0x84 bytes later ("block-relative"):
block offset = record offset − 0x84.

#### White balance

Backup byte `+0x06C` (verified): `0x00` Auto, `0x01` Auto White Priority,
`0x02` Auto Ambience Priority, `0x03` Daylight, `0x04` Shade, `0x05`–`0x07`
Fluorescent 1–3, `0x08` Incandescent, `0x09` Underwater, `0x0A` Kelvin,
`0x0B`–`0x0D` Custom 1–3.

RAW conversion `0x22D` (with `0x229` = 2): 2 Auto, 4 Daylight, 6 Incandescent,
8 Underwater, `0x8001`–`0x8003` Fluorescent 1–3, `0x8006` Shade, `0x8007`
Kelvin, `0x8008`–`0x800A` Custom 1–3, `0x8020` Auto White Priority, `0x8021`
Auto Ambience Priority.

Photo `0x1002` (exiftool; ✓ = verified): `0x0` Auto ✓, `0x1` White Priority,
`0x2` Ambience Priority, `0x100` Daylight ✓, `0x200` Shade, `0x300`
Daylight Fluorescent, `0x301` Day White Fl., `0x302` White Fl., `0x303` Warm
White Fl., `0x400` Incandescent, `0x600` Underwater, `0xF00`+ Custom,
`0xFF0` Kelvin ✓.

#### White balance shift

Each WB mode keeps its own shift. The backup stores them in a table of 4-byte
entries, red at `+0`, blue at `+2`, byte = 9 − value (verified). Record
offsets of the red/blue bytes:

| Mode | Red | Blue | | Mode | Red | Blue |
|---|---|---|---|---|---|---|
| Auto | `0x06E` | `0x070` | | Daylight | `0x08A` | `0x08C` |
| Auto White Priority | `0x072` | `0x074` | | Shade | `0x08E` | `0x090` |
| Auto Ambience | `0x076` | `0x078` | | Fluorescent 1 | `0x092` | `0x094` |
| Custom 1 | `0x07A` | `0x07C` | | Fluorescent 2 | `0x096` | `0x098` |
| Custom 2 | `0x07E` | `0x080` | | Fluorescent 3 | `0x09A` | `0x09C` |
| Custom 3 | `0x082` | `0x084` | | Incandescent | `0x09E` | `0x0A0` |
| Kelvin | `0x086` | `0x088` | | Underwater | `0x0A2` | `0x0A4` |

#### Color, sharpness and noise reduction in photos

- `0x1003` Color: `0x0` 0, `0x80` +1, `0x100` +2 ✓, `0xC0` +3 ✓, `0xE0` +4 ✓,
  `0x180` −1, `0x400` −2, `0x4C0` −3, `0x4E0` −4.
- `0x1001` Sharpness: `0` −4, `1` −3, `2` −2 ✓, `0x82` −1 ✓, `3` 0 ✓,
  `0x84` +1 ✓, `4` +2 ✓, `5` +3, `6` +4.
- `0x100E` High ISO NR: `0x0` 0, `0x180` +1, `0x100` +2, `0x1C0` +3, `0x1E0`
  +4, `0x280` −1, `0x200` −2, `0x2C0` −3 ✓, `0x2E0` −4 ✓.

---

## 1. The settings backup file

The camera exports all its settings as one **38,780-byte** file (verified),
over USB (§2.1) or Bluetooth (§3.2).

```
0x0000  "FUJIFILMX-BACKUP" "0100" "X-T5", camera serial, zero padding
0x0098  u32 0xA8 (start of checksummed data)
0x00A0  u32 0x96D4 (file size − 0xA8)
0x00E8  u32 checksum
...     other settings (menus, buttons, ...)
0x7B7C  C1 record (0x400 bytes); C2 at 0x7F7C ... C7 at 0x937C, ending at EOF
```

**Privacy:** the file contains the camera's **serial number** (at `0x34`), its
**Wi-Fi password** (at `0x7A98`, 20 bytes) and the lens serial (at `0x7ACE`).
Strip them before sharing a backup.

**Checksum (verified).** A u32 LE at `0x0E8`:

```
chk = Σ bytes[0xA8 .. EOF] − (the 4 checksum bytes) − bytes[0x7B24..0x7B25] − 954
```

`0x7B24..0x7B25` is a value that changes on its own (sensor-like) and is
excluded; 954 is an unexplained constant. **When editing, apply a delta:**
new = old + Σ(new byte − old byte) over the edited bytes. That works without
knowing the constant, as long as you don't touch `0x7B24..0x7B25`. A file with
a wrong checksum is refused (PTP `0x200F`).

**Writing a recipe** means: read a fresh backup, set the bytes of one slot
(see the [quick reference](#recipe-settings)), fix the checksum, restore the
file. Everything outside the edited bytes passes through unchanged.

---

## 2. USB: PTP in "USB RAW CONV./BACKUP RESTORE" mode

Set the camera's USB mode to **USB RAW CONV./BACKUP RESTORE** (PTP property
`0xD16E` reads 6). In this mode the card is hidden; the camera offers the
settings backup and RAW conversion. Any PTP stack works; on Apple platforms,
ImageCaptureCore's `ICCameraDevice.requestSendPTPCommand(_:outData:completion:)`
passes raw PTP through (verified on iOS and macOS).

Commands are standard PTP containers: u32 length, u16 type (1 command, 2 data,
3 response), u16 code, u32 transaction id, u32 parameters. ImageCaptureCore
fills in the transaction id but expects the full container.

### 2.1 Reading and restoring the backup

**Read (verified):** `OpenSession`, `GetObjectInfo(0)` (format `0x5000`),
`GetObject(0)` returns the backup.

**Restore (verified):**

1. `SendObjectInfo` (`0x100C`), parameters `(0, 0)`, data = a **1076-byte**
   ObjectInfo, all zero except format `0x5000` at offset 4 and the file size at
   offset 8. libfuji's 1088-byte variant makes the X-T5 refuse the next step
   with `0x200F`.
2. `SendObject` (`0x100D`) with the file.
3. The camera applies it **after it's switched off and on**.

### 2.2 In-camera RAW conversion

The camera renders a RAF it shot with any settings, like FUJIFILM X RAW Studio
does (verified from macOS and from an iPhone 16 Pro over USB-C).

1. **Upload the RAF:** `0x900C` (Fuji SendObjectInfo) with parameters
   `[0, 0, 0]` and an ObjectInfo of format **`0xF802`**, the file size, and
   filename `FUP_FILE.dat` (empty dates and keywords). Then `0x900D` with the
   RAF as data. 86 MB took ~2 s. A wrong format fails silently.
2. **Read the profile:** `GetDevicePropValue(0xD185)` now returns the RAF's
   as-shot conversion settings: 625 bytes, u16 cell count (29), the IOP code as
   a PTP string from offset 2, then signed 32-bit cells at the fixed offsets in
   the [quick reference](#recipe-settings). A RAF from another body fails here
   with `0x2002`.
3. **Write the profile:** `SetDevicePropValue(0xD185)` with the read, padded to
   **629 bytes**, cell count 29 at offset 0 and IOP length byte 9 at offset 2.
   Patch every version from the original read: reading after a write returns
   the write.
4. **Render:** take a snapshot of `GetObjectHandles(0xFFFFFFFF, 0, 0)`, then
   `SetDevicePropValue(0xD183)` with u16 `0` (preview, 3888×2592, ~3.5 MB,
   about 1 s) or `1` (full size, 7728×5152, ~21 MB, ~1.2 s). Poll the handles
   until a new one appears.
5. **Fetch and clean up:** `GetObject(new handle)` (a JPEG), then
   `DeleteObject`.

The RAF stays loaded, so repeat steps 3–5 for more versions. High ISO NR
lookup for cell `0x24D`: +4 `0x5000`, +3 `0x6000`, +2 `0`, +1 `0x1000`, 0
`0x2000`, −1 `0x3000`, −2 `0x4000`, −3 `0x7000`, −4 `0x8000`. Don't render
above the RAF's own dynamic range (reported to turn an X-T50 render green),
and never retry a render that timed out (§6).

---

## 3. Bluetooth LE

The camera advertises with manufacturer data starting with Fujifilm's company
ID **`0x04D8`**. Backup and restore over Bluetooth **apply immediately**, with
no restart (verified). They only work while the phone is the camera's
**SELECT PAIRING DESTINATION**; otherwise the camera ignores the start
commands.

Characteristics used (the full set is discovered with CoreBluetooth):

| Purpose | UUID |
|---|---|
| Pairing status (read; ack by writing back) | `F557D96B-8284-4667-8793-B971C1DECA2A` |
| Client name (write, NUL-terminated) | `85B9163E-62D1-49FF-A6F5-054B4630D4A1` |
| App flags (write `01`) | `CAEDB497-83BF-482C-91EF-91CF6F1216FF`, `98934B2C-756C-4632-AA2F-DCBA1BFEC824` |
| Geotag interval (write) | `C95D91AE-B247-4D6D-8661-7DD5D6A0F85B` |
| Time request (notify) / time (write) | `F9150137-5D40-4801-A8DC-F7FC5B01DA50` / `C52EDBCE-1FE2-4ECC-9483-907E6592BE9E` |
| Backup: start / ready | `F14E7A88-6F57-4872-A69B-B47EF3295C9B` / `11438C83-CFB0-4511-841B-759E0D2321C8` |
| Restore: start / ready | `F1897EB9-A9AA-40F0-8CE6-2FA7915BC459` / `4B3A413C-230F-42BC-B3ED-B1DB2EADEE82` |
| Chunk size / reset | `7F3400FE-17E7-4B80-8A0E-81B0343C1B49` / `051DD980-DF9D-4472-A2E1-35811DD24EE1` |
| File info (name[40] + u32 size) | `C922AC69-9480-4348-8F4B-9EE29BC30D1D` |
| File data (chunks) | `AC0C799A-FA6C-4DF5-BBC5-BB95CCE7E6EA` |
| Progress (notify) / finish | `2E27ED9F-5506-41CD-BA48-DAC06669AD95` / `68052E8A-FB91-404F-8847-0EB4BE24308C` |
| Wi-Fi: SSID / password | `BF6DC9CF-3606-4EC9-A4C8-D77576E93EA4` / `E809256A-915C-4967-92E8-53B7D4CAD213` |
| Wi-Fi: start / status | `600655E6-3637-42F1-8FB2-44EFC5C63B13` / `A68E3F66-0FCC-4395-8D4C-AA980B5877FA` |
| Wi-Fi keep-alive (read every 8 s) | `75823784-FBB7-4B71-ABAE-CD9A34072E3C` |

Session services to subscribe to (all notify/indicate characteristics in them):
`4C0020FE-F3B6-40DE-ACC9-77D129067B14`, `4E941240-D01D-46B9-A5EA-67636806830B`,
`AF854C2E-B214-458E-97E2-912C4ECF2CB8`, `804DAA8E-FFEB-4AB3-8E75-6EDD7303208D`.

### 3.1 Pairing and session setup

The first connection must happen while the camera shows **PAIRING
REGISTRATION**; reading the protected pairing-status characteristic makes iOS
bond. After that, every connection runs the same setup (verified):

1. Read the pairing status (4 bytes); write it back with **byte 3 = `0x20`**.
   Byte 2 varies (`00`, `01`, `21`) and doesn't say whether the camera accepts
   the device. A camera that doesn't accept it simply disconnects after this
   ack.
2. Write the client name (it shows in the camera's pairing list).
3. Write `01` to both app flags.
4. Subscribe to the session services' notifications.
5. Write `0A 00` to the geotag interval.
6. When the time-request characteristic notifies, write the time: u16 year,
   month, day, hour, minute, second, then 5 zero bytes.

### 3.2 Reading and restoring the backup

Data is a **zlib** stream (`78 9C`, raw DEFLATE, Adler-32), followed by a
**16-bit byte sum** of the stream, sent in chunks of `u16 index, u32 length,
payload`. The last chunk has index `0xFFFF`.

**Read (verified):**

1. Write `01 00` to *backup start*. Wait for *backup ready* = `01` (`00` means
   busy: ask again a few seconds later).
2. Write chunk size `120` (u32), then reset `0`.
3. On each *progress* notification `01`/`02 …`, read *file data* for the next
   chunk. `03` means done.
4. Read *file info*, write `03 00` to *finish*, then decompress. The byte sum
   must match.

**Restore (verified):** write `01 00` to *restore start*, wait for *restore
ready*, set the chunk size and reset, and write *file info* (name, for example
the date as `MMDDYYYY_HHMMSS`, padded to 40 bytes, plus the compressed size).
Then write the chunks to *file data*, each answered by a progress notification
`02` (`03` after the last one), and finish with `03 00`. The camera applies it
at once. Right after a transfer it may answer *busy* to the next start for a
few seconds.

### 3.3 Opening the camera's Wi-Fi

For photo transfer the camera opens its own access point when asked over
Bluetooth (verified). The X-T5 has no camera menu to start it for a phone.

1. **Read the status** first: `03 80` = network off, `00 80` = switching or
   not in transfer mode, `01 80` = on.
2. If it isn't on, write **`03 00`** to *Wi-Fi start* (libfuji writes `04 00`
   for older bodies). Writing it while the network is on switches the network
   **off**; a second write switches it on again.
3. Read the SSID (for example `FUJIFILM-X-T5-xx`) and the password.
4. Wait for the status indication `01 80`.
5. Read the keep-alive characteristic every 8 s while Wi-Fi is in use.

The client then joins that network (WPA2, the SSID and password above) before
connecting (§4).

---

## 4. Wi-Fi: photo browsing and import

The camera is at **192.168.0.1:55740** (TCP). Use one connection, and never
leave a connection attempt running: one that connects later without an init
occupies the camera's only session.

**Init (verified).** Send an 82-byte packet: u32 `0x52`, u32 `1`, u32
`0x8F53E4F2`, a 16-byte client GUID, and the client name as UTF-16LE padded to
54 bytes. The reply carries type 5 on refusal; otherwise the camera's name
starts at offset 28 (`X-T5`). After that, every message is a USB-style PTP
container (§2).

**Session setup (verified, order as the official FUJIFILM XApp sends it):**

1. `OpenSession(1)`
2. `GetDevicePropValue(0xD212)` (event list: u16 count, then u16 property + u32
   value each; a fresh session reports `DF00 = 6`, the photo count in `D222`)
3. `SetDevicePropValue(0xDF01, u16 0x14)` (client mode)
4. `0xDF28`: read it and write the same value back
5. `GetDevicePropValue(0xD244)` twice
6. `0x9054(0x10000001)`. If this answers `0x2009`, the camera isn't in its
   transfer mode (for example on a network left open after a closed session):
   stop and restart the network over Bluetooth.
7. `0x9055(0x10000001)`, `0x9050`, `GetDevicePropValue(0xD22B)`,
   `0x9053(0, 30000)`, `0xD212`
8. `GetDevicePropValue(0xD620)` (count) and `(0xD621)` (u32 count + u32
   handles)

**Listing:** `GetObjectInfo(handle)`, a standard 142-byte ObjectInfo with
format, size, filename and capture date. Some handles answer `0x2019`
(busy); skip them.

**Previews (verified):** always send `GetObjectInfo(handle)` **immediately
before** `GetThumb(handle)` (`0x100A`, ~30–46 KB JPEG). GetThumb on its own gets
no answer and hangs the session.

**Downloads (verified).** Two properties choose what you get:

| Purpose | Set | Result |
|---|---|---|
| Previews | `D226 = 0`, `D227 = 0` | starting state |
| Smartphone-sized copy | `D227 = 0`, then `D226 = 1` | resized JPEG (~3–7 MB) |
| Original file | `D226 = 0`, then `D227 = 1` | full JPEG (~26 MB) or RAF (~87 MB) |

`D227 = 1` also makes GetObjectInfo report the real size (otherwise JPEGs list
a 168 KB placeholder). Then: `GetObjectInfo(handle)`, read `D235` (max
transfer size), and `GetPartialObject(handle, offset, length)` in pieces of up
to 8 MB, advancing by the bytes received. Originals arrived at ~12 MB/s.

**Keep the session alive:** poll `0xD212` about once a second while idle and
after each download. A session that goes silent (client suspended, request
unanswered) is ended by the camera, which may then refuse connections until
it's restarted. A **clean** close between requests is fine: the camera refuses
new connections for ~25 s, then accepts a full new session.

---

## 5. Recipes in photo metadata (maker note)

Each photo's EXIF maker note starts with `"FUJIFILM"` and a u32 offset
(`0x0C`) to a little-endian IFD. All offsets are relative to that header, so
the same parser works on JPEG, HEIF and RAF files. Tag IDs follow exiftool's
FujiFilm table. The encodings in the [quick reference](#recipe-settings) were
checked on X-T5 renders of seven known recipes, which all read back exactly.
Edited or re-shared copies of a photo usually lose the maker note.

---

## 6. Things that hang or break the camera

- A **RAW conversion that never finishes**, or a `0x2019` afterwards, needs a
  restart (sometimes a battery pull). Never retry a timed-out render.
- **GetThumb without GetObjectInfo** before it, or **GetPartialObject on a
  file without GetObjectInfo** before it, over Wi-Fi: no answer, session hangs.
- A Wi-Fi session that **goes silent**, or a connection attempt that connects
  without sending an init: the camera keeps refusing connections, often until
  restarted.
- **Writing the Wi-Fi start command while the network is on** switches it off.
- **Restoring a backup with a wrong checksum**, or with libfuji's 1088-byte
  ObjectInfo, is refused with `0x200F`. Harmless, but the write doesn't happen.
- **Rendering above the RAF's dynamic range** has been reported to produce
  green images (X-T50).

## 7. Sources and credits

- **Own measurements** on an X-T5 (fw 4.31): settings calibration (change one
  setting, compare backups), USB/Bluetooth/Wi-Fi tests, and traffic of the
  official FUJIFILM XApp and another iOS app, captured on our own devices
  (Apple PacketLogger, `rvictl` + tcpdump). Only commands, codes, sizes and
  timings were decoded.
- [libfuji / fudge](https://github.com/petabyt/fudge) by petabyt (MIT): Wi-Fi
  init and sequence, Bluetooth Wi-Fi handoff, RAW conversion.
- [fujifilm-ptp-reference](https://github.com/zgrimz/fujifilm-ptp-reference) by
  zgrimz (docs CC BY 4.0): RAW conversion profile layout and quirks.
- [exiftool](https://exiftool.org/TagNames/FujiFilm.html) by Phil Harvey:
  maker note tag IDs and value lists.
- [FilmKit](https://github.com/eggricesoy/filmkit) by eggricesoy (MIT): cross-check
  of the recipe settings and ranges.

This document is licensed under
[CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). Corrections and
results from other bodies are welcome.
