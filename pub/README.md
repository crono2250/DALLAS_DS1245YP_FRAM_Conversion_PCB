# Public TDS7104 NVRAM resources

This directory contains independently reconstructed technical information for servicing the Tektronix TDS7000-series PPC NVRAM, with emphasis on the TDS7104.

## Publication policy

Only material that can be independently recreated from analysis is published here.

Not published:

- Tektronix firmware or executable/object files
- Windows 98 system/driver files
- Original `setup.tcs`, `.sn`, `.key`, or other recovered HDD files
- Device-specific option keys
- Third-party binaries or documentation without redistribution permission

Published:

- Reconstructed NVRAM address map
- Recovery workflow and safety notes
- Machine-readable layout metadata
- Hashes used to identify the private analysis source set

## Important cautions

1. Preserve the original NVRAM contents before any write operation whenever possible.
2. Do not invent a MAC address. The recovered HDD set did not contain a confirmed backup of the original PPC Ethernet MAC address.
3. `NvramClearDb` and `NvramClearNvram` are different operations. The former removes TekScope database files for power-up/default regeneration; the latter reformats the NVRAM filesystem and is more destructive.
4. Acquisition/calibration data is separate from the PPC NVRAM configuration described here.
5. Some address-range behavior near the end of the mapped NVRAM remains unresolved; see the [open verification tasks](TDS7104_NVRAM_RECOVERY.md#open-verification-tasks).

## Files

- `TDS7104_NVRAM_RECOVERY.md` — reconstructed map, behavior, and recovery sequence
- `tds7104_nvram_layout.yaml` — machine-readable address/layout summary
- `source_manifest.sha256` — private source-set identifier only; source files are intentionally not included
