# OrangeFox device tree - Infinix X6885 (MT6789)

Recovery-in-`vendor_boot` tree generated from the stock `vendor_boot.img`
(Android 16, build `BP2A.250605.031.A3`, `X6885-16.3.0.145`).

| | |
|---|---|
| Device | Infinix X6885 (`Infinix-X6885`, board `x6885_h8923`) |
| SoC | MediaTek MT6789 (Helio G99 family), arm64 only |
| Kernel | GKI 6.12.38-android16, 4K pages (lives in `boot`, not touched) |
| Layout | A/B + Virtual A/B, dynamic partitions (`super`), no `recovery` partition |
| vendor_boot | header v4, 64 MiB, platform + "recovery" ramdisk fragments, DT table, bootconfig |
| Display | 1080x2400, density 420, `BGRA_8888` |
| Encryption | FBE `aes-256-xts:aes-256-cts:v2+inlinecrypt_optimized` + metadata encryption |

## Build with carlodandan/OrangeFox-Action-Builder

Push the **contents of this folder** to the root of your own GitHub repo, then run
the *OrangeFox - Build* workflow with:

| Input | Value |
|---|---|
| MANIFEST_BRANCH | `12.1` (**not** 11.0: boot header v4 needs 12.1) |
| DEVICE_TREE | URL of your repo |
| DEVICE_TREE_BRANCH | your branch (e.g. `main`) |
| DEVICE_PATH | `device/infinix/X6885` (must match `DEVICE_PATH` in BoardConfig.mk) |
| DEVICE_NAME | `X6885` |
| BUILD_TARGET | `vendorboot` |

The builder runs `lunch twrp_X6885-eng && mka adbd vendorbootimage`.
Output: `out/target/product/X6885/OrangeFox*.img`.

Before the first build set `OF_MAINTAINER` in `BoardConfig.mk`.

## Flash (read all of it first)

1. Bootloader must be unlocked. Keep the **stock** `vendor_boot.img` and `boot.img` safe.
2. Disable AVB checks with your stock vbmeta:
   `fastboot flash vbmeta vbmeta.img --disable-verity --disable-verification`
3. `fastboot flash vendor_boot OrangeFox-...img`
4. `fastboot reboot recovery`

Because this build is a single ramdisk (all of Fox, plus the stock modules and
first-stage fstab), **normal Android boot also goes through it**. If Android does not
boot, reflash the stock `vendor_boot.img` from fastboot.

## What is verified vs. not

Verified from the stock image (see comments `[stock]` in BoardConfig.mk): boot header
addresses/offsets, page size, vendor cmdline, bootconfig, partition size, DTB (byte-exact),
all 234 kernel modules + `modules.load/.load.recovery/.dep/.alias/.softdep`, both stock
first-stage fstabs, pixel format, screen size/density, encryption flags, USB init.

**Not verified:** this tree has *not* been compiled and has *not* been booted on a device.
Expect to fix a variable or two on the first CI run. Open items:

* **Touch** - the DTB has a Goodix touch node but the driver is not in `vendor_boot`
  (it is in `vendor_dlkm` or built in). Find it in Android with
  `ls /vendor_dlkm/lib/modules | grep -i -E "goodix|gt9|touch"`, copy the `.ko` (and its
  dependencies) into `recovery/root/lib/modules/` and append them to `modules.load.recovery`.
  Until then use volume keys / adb.
* **Decryption of /data** - flags are set, but keymint/gatekeeper services and their
  libraries from `/vendor` must be added; the stock Android 16 stack (AIDL keymint 7.0) is
  much newer than the Android 12.1 recovery base, so this is the hardest item.
* **A/B slot control** - stock uses an AIDL boot HAL; the 12.1 recovery talks HIDL. Slot
  switching inside the recovery may not work.
* `TW_MAX_BRIGHTNESS`, `BOARD_SUPER_PARTITION_SIZE` (placeholder, build-time only),
  SD card node and OTG storage: please verify on the device.
* Flashlight / vibrator / thermal paths are intentionally not set (not derivable from the image).

## Refreshing from new firmware

```
python3 tools/unpack_vendor_boot.py vendor_boot.img out/
cp out/dtb.img prebuilt/dtb.img
cp out/ramdisk_0_platform/lib/modules/* recovery/root/lib/modules/
cp out/ramdisk_0_platform/first_stage_ramdisk/fstab.mt6789 recovery/root/first_stage_ramdisk/
cp out/ramdisk_1_recovery/first_stage_ramdisk/fstab.emmc  recovery/root/first_stage_ramdisk/
```
Kernel modules must match the kernel in `boot` exactly (vermagic
`6.12.38-android16-5-gcc51d883045d-4k`) - always refresh them together with `boot`.
