# TDS7104 PPC NVRAM reconstruction notes

## Scope

This document records independently reconstructed behavior of the PPC-board NVRAM used by the Tektronix TDS7104/TDS7000 family. Original firmware, recovered HDD files, option keys, and other third-party binaries are intentionally not distributed here.

## Confirmed base values

- TekScope-side NVRAM start symbol: `0xFD0E0000`
- TekScope-side NVRAM size symbol: `0x00020000` bytes
- `sysNvRamGet()` / `sysNvRamSet()` logical base: `0xFD0E0100`
- BSP reserved logical region: `0x1000` bytes
- Boot-line logical offset: `0x000`
- Ethernet MAC logical offset: `0x100`, length 6 bytes
- TCS/configuration shared logical block: `0x200..0xFFF` (`0xE00` bytes)
- Serial/option configuration structure: logical `0x800..0x857` (`0x58` bytes)
- DBMS power-up status: physical `0xFD0E1100..0xFD0E1103`
- TekScope database RAM-disk base: physical `0xFD0E1104`

Physical address for `sysNvRam` offsets is:

```text
physical = 0xFD0E0100 + logical_offset
```

## Reconstructed logical layout

```text
logical offset     physical address     use
----------------   ------------------   ---------------------------------------
0x000              0xFD0E0100           VxWorks boot line (up to 255 bytes)
0x100              0xFD0E0200           Ethernet MAC storage, 6 bytes
0x106..0x1FF       0xFD0E0206..02FF     not classified by current analysis
0x200              0xFD0E0300           TCS/configuration checksum, 2 bytes
0x202..0x7FF       0xFD0E0302..08FF     serialized TCS data / padding
0x800..0x857       0xFD0E0900..0957     serial + option configuration
0x858..0xFFF       0xFD0E0958..10FF     remaining shared TCS/config region
0x1000             0xFD0E1100           DBMS power-up status (4 bytes)
0x1004             0xFD0E1104           TekScope DB RAM-disk base
```

The boot-line call observed in the BSP uses a length of `0xFF` bytes, so the final byte actually read/written by that operation is logical offset `0x0FE`; the nominal `0x100` slot boundary is retained in this map.

## TCS/configuration checksum

The shared block beginning at logical `0x200` contains a two-byte checksum followed by `0xDFE` bytes of protected data:

```text
checksum storage : logical 0x200..0x201
checksum input   : physical 0xFD0E0302, length 0x0DFE
```

Serial/option updates recalculate the same checksum; therefore those values are part of the shared protected block rather than an independent NVRAM area.

## Serial / option structure

At logical offset `0x800`:

```text
+0x00  uint32_be serial_length
+0x04  char      serial[16]
+0x14  uint32_be option_key_length
+0x18  char      option_key[64]
--------------------------------
        total 0x58 bytes
```

The original device-specific values are intentionally not published.

## HDD backup file format behavior

Analysis shows the private backup files used by TekScope are length-prefixed binary records rather than plain text:

```text
uint32_be length
uint8_t   payload[length]
```

The actual recovered `.sn` and `.key` files are intentionally not included in this repository.

## TCS reload interaction with serial/option data

The TCS serializer writes the entire `0xE00`-byte block and fills unused space with `0xAA`. The serial/option structure resides inside this block. Therefore a TCS reload can temporarily replace the serial/option fields with `0xAA`.

An invalid length such as `0xAAAAAAAA` is then rejected by the configuration loader, which can restore the private serial/option values from their HDD backup files and recalculate the shared checksum.

This explains why TCS reload and serial/option recovery can appear sequentially during a legitimate recovery flow.

## Factory-default database behavior

`NvramClearDb` and `NvramClearNvram` must not be treated as synonyms.

### `NvramClearDb`

The factory-default path removes the TekScope database files and allows them to be regenerated. The files targeted by the analyzed implementation include:

```text
/nvram/TDS7000.CVN
/nvram/TDS7000.CVO
/nvram/TDS0.DBS
/nvram/TDS1.DBS
...
/nvram/TDS10.DBS
```

This is the behavior associated with restoring the power-up scope state to factory defaults.

### `NvramClearNvram`

This invokes filesystem creation/format behavior and is more destructive. It should not be used merely as a substitute for the factory-default database reset.

## VxWorks `nvfs=0x1000`

The boot-line `other` parameter `nvfs=0x1000` is parsed as an NVRAM filesystem size of 4096 bytes. The BSP computes the initial `/nvram` base as:

```text
0xFD100100 - 0x1000 = 0xFD0FF100
```

During later TekScope initialization, the initial device can be removed and replaced by the larger TekScope NVRAM RAM-disk beginning at `0xFD0E1104`.

## MAC address handling

The Ethernet address is stored at logical offset `0x100` for six bytes. The analyzed display/set routines use reversed byte ordering relative to conventional textual MAC display.

The recovered HDD set did not contain a confirmed copy of the original PPC MAC address. Therefore the original MAC must not be guessed. If surviving NVRAM is available, read and record it before destructive operations. On the analyzed BootROM, `n netif` is the read-only command used to print the network interface device address.

## Recommended recovery order

Before any write:

1. Preserve/dump the existing NVRAM if possible.
2. Record current BootROM parameters.
3. Record the Ethernet MAC with the BootROM read command if it remains valid.
4. Preserve acquisition/calibration information separately.

For reconstruction using legitimately retained private service data:

1. Restore a valid boot line appropriate to the instrument/software installation.
2. Restore the thermal-control table using the instrument's legitimate private TCS source.
3. Allow the instrument software to restore its private serial/option values using its normal recovery path.
4. Verify that the shared configuration checksum is regenerated successfully.
5. Use the database-only factory-default path (`NvramClearDb`) when the goal is to regenerate TekScope power-up/user database state.
6. Disable the clear flag after the one intended factory-default boot.
7. Do not touch acquisition calibration merely to repair PPC NVRAM state.

## Unresolved point

The observed TekScope RAM-disk construction uses a base of `0xFD0E1104` and parameters that, when interpreted as a simple `254 * 512` byte device, extend beyond the straightforward `0x20000`-byte range implied by the NVRAM size symbol. Current evidence is insufficient to determine whether this results from address mirroring, board-level decode behavior, or another implementation detail. Do not assume an answer without board-level confirmation.
