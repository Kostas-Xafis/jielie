# SDK config

## AC46xx SDK

- file: ac46btcf.bin

```
2: CRC16 of the rest

2: Size of the file (including this field and the CRC16 field before)
    => Should be 0x31 (49) bytes

32: Device name

4: PIN code

6: MAC address

1: Something1 (0x18)

1: Something2 (0x0A)
```

## AC690N SDK (BT configuration tool)

- file: bt_cfg.bin

```
2: CRC16 of the rest

2: Size of the file (including this field and the CRC16 field before)
    => Should be 0x2B0 (688) bytes

1: Count of the device names
    => Range: 1~20

32: Device name [1]
32: Device name [2]
32: Device name [3]
32: Device name [4]
32: Device name [5]
32: Device name [6]
32: Device name [7]
32: Device name [8]
32: Device name [9]
32: Device name [10]
32: Device name [11]
32: Device name [12]
32: Device name [13]
32: Device name [14]
32: Device name [15]
32: Device name [16]
32: Device name [17]
32: Device name [18]
32: Device name [19]
32: Device name [20]

4: PIN code

6: MAC address

1: RF power
   => Range: 0~9

1: AEC dac analog gain
   => Range: 0~99 (seemingly)

1: AEC mic analog gain
   => Range: 0~99 (seemingly)

16: BLE device name

6: BLE public address

1: BLE random address type
    0 = Static random
    1 = Resolvable random
    2 = Non-resolvable random

6: BLE random address

1: BLE identity address
    0 = Uses public address
    1 = Uses random address

```

## AC692N SDK (SDK configuration tool)

- file: bt_cfg.bin

Differs from the AC690N SDK's bt_cfg by the fact that it the device names have
the CRC16 prepended to them, and the BLE device name being now 32 bytes long.

```
2: CRC16 of the rest

2: Size of the file (including this field and the CRC16 field before)
    => Should be 0x2C0 (704) bytes

1: Count of the device names
    => Range: 1~20

32: Device name [1]
32: Device name [2]
32: Device name [3]
32: Device name [4]
32: Device name [5]
32: Device name [6]
32: Device name [7]
32: Device name [8]
32: Device name [9]
32: Device name [10]
32: Device name [11]
32: Device name [12]
32: Device name [13]
32: Device name [14]
32: Device name [15]
32: Device name [16]
32: Device name [17]
32: Device name [18]
32: Device name [19]
32: Device name [20]
    ** Each field has the CRC16 of the name that comes next
    e.g. 29 52 74 73 75 6B 61 73 61 00 00 ...
         |---| |------------------>
         CRC16  Name (remaining 30 bytes)

4: PIN code

6: MAC address

1: RF power
   => Range: 0~9

1: AEC dac analog gain
   => Range: 0~99 (seemingly)

1: AEC mic analog gain
   => Range: 0~99 (seemingly)

32: BLE device name
    ** Same as with the BT device name, it starts with the CRC16

6: BLE public address

1: BLE random address type
    0 = Static random
    1 = Resolvable random
    2 = Non-resolvable random

6: BLE random address

1: BLE identity address
    0 = Uses public address
    1 = Uses random address

```

## JL701N SDK (config tool)

- file: cfg_tool.bin

Not a fixed structure anymore, but a list of items, each with an ID that matches the `CFG_xxx` IDs
from the SDK's `syscfg_id.h` (the same IDs that are passed to `syscfg_read()`).

Can be read and written on a running device with the [config tool protocol](../protos/cfgtool.md).

```
4: Size of the rest of the file

then the items, until the end of the file:
  1: Low byte of the CRC16 of the item data
  3: (little-endian)
      bit0-11  = Item ID
      bit12-23 = Item data length
  N: Item data
```

Items seen in a JL701N based speaker (890 byte file):

|  ID  | Size | SDK name                | Contents                                           |
|------|------|-------------------------|----------------------------------------------------|
| 101  | 660  | CFG_BT_NAME             | 20 entries of: 1 byte flag (01 on the first entry, 00 on the rest), 32 bytes name |
| 102  | 6    | CFG_BT_MAC_ADDR         | All FF                                             |
| 103  | 32   | CFG_BLE_NAME            | Name                                               |
| 104  | 6    | CFG_BLE_MAC_ADDR        | All FF                                             |
| 524  | 4    | CFG_AUDIO_ID            | `AUDIO_CONFIG`, see below                          |
| 537  | 3    | CFG_MIC_TYPE_ID         |                                                    |
| 538  | 4    | CFG_COMBINE_SYS_VOL_ID  |                                                    |
| 539  | 4    | CFG_COMBINE_CALL_VOL_ID |                                                    |
| 601  | 1    | CFG_BT_RF_POWER_ID      |                                                    |
| 602  | 2    | CFG_TWS_PAIR_CODE_ID    | FF FF                                              |
| 603  | 1    | CFG_AUTO_OFF_TIME_ID    |                                                    |
| 604  | 72   | CFG_AEC_ID              | A few bytes, then floats                           |
| 607  | 9    | CFG_LRC_ID              |                                                    |
| 615  | 1    | CFG_BLE_RF_POWER_ID     |                                                    |
| 1022 | 3    | ?                       | FF 3F 07                                           |
| 1023 | 14   | ?                       | Version string of the config: `soundbox-1.1.0`     |

`AUDIO_CONFIG` (item 524):

```
1: Enable
1: Max system volume (31)
1: Default volume (25)
1: Tone volume (25)
```

Note that the firmware does not have to honor these: on that speaker changing the tone volume
(with the item CRC and the file's data CRC fixed up) made no audible difference.
