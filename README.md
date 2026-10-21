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

## Second fix: ACPI S5 on `SLP_TYP` without `SLP_EN` (`DevACPI.cpp`)

Loading coreboot was not enough to boot it: with the guest's ACPI device enabled,
coreboot powered the VM off partway through ramstage. Root cause, traced on the wire:
coreboot's PIIX4 southbridge (`i82371eb`) writes `SUS_TYP_S0 | SCI_EN` to the PM1a
control register -- value `0x1401`, i.e. `SLP_TYP = 5` with `SLP_EN` (bit 13)
**clear**. On real PIIX4 that is a harmless "arm the S0 sleep type and enable SCI"
write; nothing sleeps, because the ACPI spec (1.0, section 4.7.2.2.1) makes `SLP_EN`
the trigger -- `SLP_TYP` alone only selects which state a *subsequent* `SLP_EN=1`
would enter.

VirtualBox's `acpiR3PM1aCtlWrite` decided the sleep state from `SLP_TYP` **alone**,
without checking `SLP_EN`, and its `SLP_TYP` encoding maps 5 to S5 -- so it read
coreboot's arm-S0 write as "enter S5" and powered off. The one-line, spec-correct
fix gates the transition on `SLP_EN`:

```c
/* ACPI 1.0+ 4.7.2.2.1: the sleep transition is triggered by SLP_EN, not by
   the SLP_TYP value alone.  Firmware legitimately pre-arms SLP_TYP (e.g.
   coreboot writes SUS_TYP|SCI_EN with SLP_EN clear); acting on SLP_TYP
   without SLP_EN spuriously powers the VM off. */
if ((u32 & SLP_EN) && uSleepState != pThis->uSleepState)
```

With this, coreboot runs to completion with the ACPI device **enabled** -- it writes
its own ACPI tables and hands off to the payload -- and ACPI stays **functional**: a
normal guest shutdown sets `SLP_TYP=5` together with `SLP_EN`, which still powers the
VM off. (VirtualBox relocates its PM I/O decode from the `PMBA` config write coreboot
makes, so coreboot's FADT and the emulated ports line up.) The change is a strict
superset of correctness -- it only ignores a class of writes the spec already says do
nothing -- so it is safe for every existing guest and is offered upstream too.

Without this fix, `--acpi off` lets coreboot boot but leaves the guest with no working
ACPI power management; the fix is what makes ACPI both present and functional.

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

With both fixes in place, a real coreboot `UefiPayloadPkg` boots all the way to an
**interactive UEFI shell** inside the patched VirtualBox, with ACPI functional:

```
UEFI Interactive Shell v2.2
EDK II
UEFI v2.70 (EDK II, 0x00010000)
Shell>
```

and `reset -s` at that prompt (an ACPI S5 shutdown written *with* `SLP_EN`) powers the
VM off cleanly -- the other half of the `SLP_EN` fix. The payload-side build/VM tweaks
needed to get there (a 36-bit `--cpuid-set` to tame the 48-bit identity map, `--hpet
on`, the LAPIC timer, and a coreboot skip-empty-MCFG fix) and the full reproduce recipe
live in `org-ai-assisted/dasharo` (`proof/uefi-shell-in-vbox.log`). Those are platform
details on the coreboot/payload side; the VirtualBox change is just the two fixes
above.

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
