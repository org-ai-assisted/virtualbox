# virtualbox -- DevEFI coreboot-ROM loader

Our fork of Oracle VirtualBox that teaches the built-in EFI device (`DevEFI`) to
load a raw **coreboot** image (and therefore real **Dasharo** firmware) instead of
only an OVMF-shaped edk2 firmware volume.

This repo carries our delta against upstream VirtualBox, not a full source mirror:
the single modified source file plus how to build and run it. The coreboot side (the
firmware this makes loadable) lives in the companion repo `org-ai-assisted/dasharo`.

The modified file is `src/VBox/Devices/EFI/DevEFI.cpp`, at its real path in the
VirtualBox tree, based on **VirtualBox 7.2.20** (the version this was developed and
proven on). To see exactly our delta, diff it against the pristine 7.2.20 file:

```
diff -u VirtualBox-7.2.20/src/VBox/Devices/EFI/DevEFI.cpp \
        virtualbox/src/VBox/Devices/EFI/DevEFI.cpp
```

(One incidental change: a single em dash in an Oracle comment was normalized to `--`
to satisfy this repo's ASCII-only policy. No code change.)

## Why

VirtualBox ships its own edk2-derived EFI firmware. We prefer real Dasharo
(coreboot + `DasharoPayloadPkg`) for the open, auditable firmware stack it gives a
guest. VirtualBox's EFI slot, however, accepts only an OVMF-shaped combined flash
device:

- `DevEFI.cpp::efiParseFirmware` requires a varstore-first firmware volume whose
  first FV carries `gEfiSystemNvDataFvGuid`
  (`FFF12B8D-7696-4C8B-A985-2747075B4F50`, the standard edk2 GUID), then a main FV
  with a reset vector. It rejects anything else.
- A `coreboot.rom` is not an edk2 firmware volume. It is a flat image whose reset
  vector sits at the top of the 4 GiB address space (`0xFFFFFFF0`), with CBFS and an
  FMAP inside. `efiParseFirmware` therefore refuses it.

So loading real coreboot in VirtualBox needs a VirtualBox-side change: a path in
`DevEFI` that maps a raw image at the top of memory and skips OVMF parsing.

## What the fork changes

The modified `efiLoadRom` adds one new CFGM boolean, `CorebootRom` (default
`false`). When set, it skips `efiParseFirmware` and maps the raw image at the top of
4 GiB, keeping a small vestigial flash device just below so the ring-0 side is
untouched. The exact added branch:

```c
bool fCorebootRom = false;
rc = pDevIns->pHlpR3->pfnCFGMQueryBoolDef(pCfg, "CorebootRom", &fCorebootRom, false);
AssertLogRelRCReturn(rc, rc);
if (fCorebootRom)
{
    pThisCC->cbEfiRom    = RT_ALIGN_64(pThisCC->cbEfiRom, GUEST_PAGE_SIZE);
    pThisCC->offEfiRom   = 0;
    pThisCC->cbNvram     = GUEST_PAGE_SIZE; /* vestigial flash, below the ROM */
    pThisCC->GCPhysNvram = (RTGCPHYS)_4G - pThisCC->cbEfiRom - pThisCC->cbNvram;
    rc = flashR3Init(&pThis->Flash, pThisCC->pDevIns, 0xA289 /*Intel*/,
                     pThisCC->cbNvram, pThisCC->cbNvram);
    if (RT_FAILURE(rc))
        return rc;
    rc = flashR3LoadFromBuf(&pThis->Flash, pThisCC->pu8EfiRom, pThisCC->cbNvram);
    if (RT_FAILURE(rc))
        return rc;
    pThisCC->GCLoadAddress = pThisCC->GCPhysNvram + pThisCC->cbNvram;
    LogRel(("EFI: coreboot image, %#RX64 bytes mapped at %RGp..4GB\n",
            pThisCC->cbEfiRom, pThisCC->GCLoadAddress));
}
else
{
    rc = efiParseFirmware(pDevIns, pThis, pThisCC);
    ...
}
```

The existing map loop then maps the whole image into the guest as shadowed ROM at
`[4G - cbEfiRom, 4G)`, so the reset vector lands at `0xFFFFFFF0` exactly as coreboot
expects. The three edits in full: drop a now-wrong `RT_NOREF(pCfg)` in `efiLoadRom`,
add the `CorebootRom` branch above, and add `"CorebootRom|"` to the
`PDMDEV_VALIDATE_CONFIG_RETURN` allow-list.

The design is deliberately **ring-3 only**:

- It does **not** add a field to the shared `DEVEFI` struct. Adding one changes
  `cbInstanceShared`, which makes PDM reject the device at ring-0 registration with
  `VERR_INCOMPATIBLE_CONFIG` ("Ring-3 does not match ring-0 device registration").
  Keeping the struct byte-identical lets the unmodified ring-0 construct
  (`efiRZConstruct`) run as-is, so no ring-0 rebuild is needed.
- Keeping the vestigial flash fully instantiated is what keeps the ring-0 MMIO setup
  valid without any ring-0 code change.

## Build

Drop the modified `DevEFI.cpp` into an upstream VirtualBox 7.2.20 source tree (same
path), then build a non-hardened headless VirtualBox from source (Debian trixie):

```
./configure --disable-hardening --build-headless --disable-qt --disable-python \
  --disable-java --disable-docs --disable-kmods --disable-pulse --disable-alsa \
  --disable-opengl --disable-dbus --disable-devmapper --disable-dxvk --disable-sdl-ttf
source ./env.sh
kmk VBoxC VBoxSVCM VBoxXPCOMIPCC VBoxXPCOM VBoxSVC VBoxManage VBoxHeadless \
    VBoxVMM VBoxDD VBoxDDU VBoxNetDHCP VBoxNetNAT VBoxXPCOMBase-xpt-inst
```

Build deps: `xsltproc libxml2-dev libxml2-utils libxslt1-dev libssl-dev
libcurl4-openssl-dev zlib1g-dev libpng-dev uuid-dev libcap-dev libpam0g-dev
makeself`.

Notes:

- The installed stock VirtualBox is hardened and rejects a hot-swapped source
  `VBoxDD.so` (ABI mismatch), so run the `--disable-hardening` build directly out of
  `out/linux.amd64/release/bin/`.
- `VBoxXPCOMBase-xpt-inst` installs `VBoxXPCOMBase.xpt`; without it `VBoxManage`
  fails with `NS_ERROR_UNEXPECTED` ("Failed to create the VirtualBox object").
- A non-hardened build opens `/dev/vboxdrv` (0600 root:root) directly, so run the
  whole stack as root.

## Run

Point the EFI firmware at a coreboot image and set the new key:

```
VBoxManage modifyvm <vm> --firmware efi
VBoxManage setextradata <vm> "VBoxInternal/Devices/efi/0/Config/EfiRom"      "/path/to/coreboot.rom"
VBoxManage setextradata <vm> "VBoxInternal/Devices/efi/0/Config/CorebootRom" 1
```

The image map loop caps the firmware at just under 8 MiB, so build coreboot at
4 MiB (`CONFIG_COREBOOT_ROMSIZE_KB_4096`).

## Proof

Built and proven on OVH (Debian trixie), 2026-10-21, with a 4 MiB coreboot
`emulation/qemu-i440fx` image (`PAYLOAD_NONE`, serial console):

VBox.log:

```
EFI: coreboot image, 0x400000 bytes mapped at 0xffc00000..4GB
```

COM1 serial showed coreboot running bootblock -> romstage -> postcar -> ramstage
and then enumerating VirtualBox's own emulated chipset:

```
PCI: 00:00:00.0 [8086/1237] enabled     # i440fx host bridge (VBox PIIX3)
PCI: 00:00:01.0 [8086/7000] enabled     # PIIX3 ISA
PCI: 00:00:02.0 [80ee/beef] enabled     # VirtualBox VGA
PCI: 00:00:04.0 [80ee/cafe] enabled     # VirtualBox guest-service device
PCI: 00:00:05.0 [8086/2415] enabled     # AC97 audio
```

The full serial capture is in `org-ai-assisted/dasharo` under `proof/`.

A real `DasharoPayloadPkg` image was also built and integrated (coreboot
`fallback/payload`, LZMA, load `0x800000`, entry `0x800860`) and the patched
VirtualBox loads and runs it exactly the same way -- coreboot reaches ramstage and
drives VirtualBox's chipset. It does **not** yet reach the UEFI shell inside
VirtualBox: coreboot's `emulation/qemu-i440fx` board expects QEMU's PIIX4 ACPI
function at `00:01.3` (`[8086/7113]`) and the `fw_cfg` port, neither of which
VirtualBox emulates, so ramstage trips an early ACPI S5 power-down (`PCI: Static
device PCI: 00:00:01.3 not found` then `ACPI: Entering S5` in VBox.log). The coreboot
-> `DasharoPayloadPkg` hand-off itself is proven under QEMU with the identical image.
Closing that gap needs a VirtualBox-matched coreboot board (or PIIX4-ACPI/`fw_cfg`
emulation in VirtualBox); see `org-ai-assisted/dasharo` `proof/` for both logs.

## Upstreaming

The change is intentionally minimal, self-contained, and **ring-3 only**, so it can
be offered to upstream VirtualBox as the "load a raw coreboot/flat firmware image"
capability. The exact upstreamable diff against pristine VirtualBox 7.2.20 is just
three hunks, 35 insertions / 8 deletions, all in `src/VBox/Devices/EFI/DevEFI.cpp`:

- drop the now-wrong `RT_NOREF(pCfg)` in `efiLoadRom`,
- add the `CorebootRom` branch (maps the raw image, vestigial flash below it),
- add `"CorebootRom|"` to the `PDMDEV_VALIDATE_CONFIG_RETURN` allow-list.

It touches no shared struct and no ring-0 code, so it needs no ring-0 rebuild. The
vestigial 4 KiB flash is the minimal way to keep `pThis->Flash` valid for the
unchanged flash/MMIO/saved-state paths without guarding each use behind a mode check.

To regenerate that exact patch, diff this repo's file against the pristine 7.2.20
source and drop the two incidental, non-feature hunks this repo's ASCII/formatting
gates introduced (one em dash normalized to `--`, one trailing blank line removed):

```
diff -u VirtualBox-7.2.20/src/VBox/Devices/EFI/DevEFI.cpp \
        virtualbox/src/VBox/Devices/EFI/DevEFI.cpp
# keep only the three efiLoadRom / config-validation hunks above;
# the em-dash comment hunk and the trailing-blank hunk are not part of the feature.
```

## License

`src/VBox/Devices/EFI/DevEFI.cpp` is part of Oracle VirtualBox and is distributed
under its terms (GPLv3, per the VirtualBox Open Source Edition). Our modification is
offered under the same terms.
