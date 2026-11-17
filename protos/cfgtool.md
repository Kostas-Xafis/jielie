# Config tool protocol

The protocol the SDK's PC configuration tool uses to read and patch the configuration files
(`cfg_tool.bin`, `eq_cfg_hw.bin`, ...) of a running device.

In the SDKs it lives in `apps/common/config/new_cfg_tool.c` and `cfg_tool.h`
(seen in the JL701N / BR28 SDK; other recent SDKs carry the same files).

Everything below was checked against a retail Bluetooth speaker (Xiaomi Sound Outdoor,
reports SDK `JL701N_V1.4.2`) which ships with this channel still enabled on its USB port.
Things that were not tried are marked as such.

There is no authentication of any kind. Besides the config files the protocol has raw flash
read/erase/write commands, so on a device that leaves it enabled the whole flash can be dumped
(and patched) over a plain USB cable, without entering UBOOT.

## Transport

- USB: the app enumerates as a CDC ACM device ("USB Composite Device", [4C4A:4A55](../misc/usb-ids.md) on the tested device),
  the packets go over the bulk data endpoints. Line coding (baud rate etc.) does not matter.
- UART: same packets, 115200 baud by default (per the SDK, not tried).
- The SDK can also route it through SPP (not tried).

On USB each packet has to arrive as a single transfer.
Requests up to 52 bytes were used here (the bulk endpoint is 64 bytes); replies came back up to 136 bytes long.

## Packet format

```
5A AA A5 | CRC CRC | L | T | SQ | DATA...
```

- `5A AA A5`: Sync mark
- u16: CRC16 of everything that follows (L, T, SQ and DATA), little-endian
  * The usual CRC16-CCITT (poly 0x1021, init 0)
- u8: L, length of T + SQ + DATA
- u8: T, packet type
  * 0x12 = request from the host
  * 0x00 = reply
  * 0x01 = sent by the device on its own (per the SDK, not seen)
  * 0x11 = audio effect tuning (per the SDK, not explored)
- u8: SQ, sequence number. The reply carries the SQ of the request.
- DATA: for requests, a little-endian u32 command followed by its arguments. For replies, just the returned data.

Since L is a single byte, a packet carries at most ~250 bytes of data.

Example, "query basic info":

```
=> 5A AA A5 73 9C 06 12 00 23 00 00 00
<= 5A AA A5 30 01 64 00 00 01 00 39 39 35 35 2D ...
```

## Commands

All fields are little-endian u32 unless noted.

| Cmd  | Name             | Arguments              | Reply                                |
|------|------------------|------------------------|--------------------------------------|
| 0x23 | Query basic info | -                      | see below                            |
| 0x0B | Query file size  | file id                | size                                 |
| 0x0C | Read file        | file id, offset, size  | `size` bytes of the file             |
| 0x22 | Prepare write    | file id, size          | flash address, file size, erase unit |
| 0x27 | Read flash       | address, size          | `size` bytes                         |
| 0x24 | Erase flash      | address, size          | `OK` / `FA`                          |
| 0x25 | Write flash      | address, size, data... | `OK` / `FA`                          |
| 0x26 | Enter upgrade mode | -                    | (not tried)                          |

Status replies are three bytes: `"OK\0"` on success, `"FA\0"` on failure and `"ER\0"` for an unknown command.

### File IDs

| ID | File                               |
|----|------------------------------------|
| 0  | `cfg_tool.bin`                     |
| 1  | `eq_cfg_hw.bin` ("old" EQ tool)    |
| 2  | `effects_cfg.bin`                  |
| 3  | `eq_cfg_hw.bin`                    |

Asking for a file that is not there returns `FA`.

### Query basic info (0x23)

```c
struct {
  uint16_t protocol_ver;
  char     prog_crc[32];  /* "9955-939485be" */
  char     sdk_name[32];  /* "JL701N_V1.4.2-@20240603-$15bb351" */
  char     pid[16];       /* "JL701N" */
  char     vid[16];       /* "0.01" */
};
```

The pid/vid come from the [flash header](../datafmt/jlfs.md).

### Read file (0x0C) / Read flash (0x27)

Both end up reading the flash directly, "Read file" just resolves the file's flash address first.
Reads of 128 bytes per request work fine, a whole 2 MiB flash takes about a minute.

The data comes back the way the application sees it: code, resources and file headers inside the app area
were all readable as plain data on the tested device, erased space reads as FF,
and the [flash header](../datafmt/jlfs.md) comes back in its stored (scrambled) form.

### Prepare write (0x22)

Returns where the file's data sits in flash, its size, and the erase unit (4096 on the tested device).
The host is expected to read the erase-unit-aligned range around the file, patch it, erase it and write it back.

This command also has two side effects in the firmware:

- The erase unit it reports is the one that the following "Erase flash" commands will use.
- The file id is remembered, see the note about the reset below.

### Erase flash (0x24)

Erases `size / erase unit` units starting from `address`, using the erase unit from the last "Prepare write".

### Write flash (0x25)

Programs `size` bytes at `address`.

In the SDK source the data is scrambled before it is programmed: `doe()` (presumably the [ENC cipher](../misc/cipher.md))
is applied in 32-byte blocks keyed with `((address - sfc base) >> 2) ^ chip key`, starting from the address given in the command.
In practice the read and write commands round-trip: writing back exactly what "Read flash" returned leaves the flash unchanged
(this can be checked without erasing anything, by "writing" a block of data over itself).
To stay on the same 32-byte grid, write in chunks whose offset from the start of the erase unit is a multiple of 32.

### Automatic reset

After each write, if the last "Prepare write" was for file 0 (`cfg_tool.bin`), the firmware adds the write size to a counter
and resets the chip 500 ms after the counter reaches the size of the last erase.
This is how the config tool makes the device pick up a new `cfg_tool.bin`.

Two consequences:

- If nothing has been erased since boot, any write (even a 32-byte one) resets the device right away.
- Doing a "Prepare write" for another file id (e.g. 3) first keeps the device from resetting,
  which is handy when patching something else than the config, like the tone files.

## Notes

- Erasing a sector the running firmware executes from is of course a bad idea.
  The resource area (tones, config, EQ) was rewritten many times on the tested device without issues.
- The files' data CRC in the [file header](../datafmt/jlfs.md) is not updated by anything here.
  Whether the firmware checks it at all for resource files was not tested;
  the safe route is to keep the CRC the same or to patch the header as well.
