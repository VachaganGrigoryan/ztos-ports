# ztOS on the Google Pixel 3a XL (bonito)

[ztOS](https://ztos.online/) is a phone OS with a Windows Phone 8.1–style shell (ztMETRO) on
Arch Linux ARM. This folder shows how to build it for the **Pixel 3a XL** (Snapdragon 670) with
[ztConflo](https://github.com/Zerone-Laboratories/ztconflo), and records **everything that had to
be fixed** to get a working phone. ztConflo's output does not boot to a usable system on its own yet.

<img src="screenshot.png" alt="ztMETRO lock screen on the Pixel 3a XL, showing Wi-Fi, Bluetooth and charging icons" width="300">

**Status:** experimental cross port, tested on one Pixel 3a XL (Samsung panel) on 2026-11-04.
ztOS GSI 0.2.1, kernel `linux-postmarketos-qcom-sdm670` 7.2.3 (mainline).

| Works | Partly | Doesn't work in ztOS yet | Not tried |
|---|---|---|---|
| Boot from internal storage, panel, GPU (Adreno 615, GLES 3.2), touch, ztMETRO, Wi-Fi (2.4 and 5 GHz), Bluetooth (scan), USB networking, speaker, battery and charging readout, CPU frequency scaling, A/B slot marking | Sensors: the DSP side runs, but ztOS's sensor service is S9-only | Mobile data, SIM, SMS, calls (ztOS modem service is S9-only); cameras (no sensor driver on mainline) | Microphone, headphones, brightness control, vibration, suspend, GPS, Waydroid |

## Install

**→ [INSTALL.md](INSTALL.md)**: the full step-by-step guide. It erases the phone.

## What was fixed

**→ [FIXES.md](FIXES.md)**: 13 problems, each with its symptom, cause and fix. In short:

- **ztConflo:**
  - doesn't pick a kernel variant or device tree for this two-panel phone;
  - leaves out the Qualcomm firmware and UCM;
  - enables `rmtfs`/`tqftpserv` without shipping them.
- **ztOS:** the GSI's `/etc/fstab` makes the postmarketOS initramfs halt.
- **Phone:**
  - an old GPT on `userdata` overwrote the new root filesystem;
  - nothing marked the A/B boot slot successful.

## Files in this folder

| Path (in this folder) | What it is |
|---|---|
| `INSTALL.md` | The installation guide |
| `FIXES.md` | Symptom, cause and fix for every problem |
| `files/` | Files installed on the phone, laid out from `/`: systemd units, the Bluetooth address helper, udev rules, NetworkManager and ld.so config |
| `patches/ztconflo-kernel-variant.diff` | Lets ztConflo take the kernel variant from `$ZT_KERNEL` |

## Upstream reports

Reported to the ztOS developers:

- [Multi-kernel devices get an empty `kernel =`](https://github.com/Zerone-Laboratories/ztconflo/issues/1)
- [`dtb_<variant>` ignored, so boot.img has no DTB](https://github.com/Zerone-Laboratories/ztconflo/issues/2)
- [Firmware dependencies and UCM not installed](https://github.com/Zerone-Laboratories/ztconflo/issues/3)
- [Daemons enabled but not shipped; `/vendor/ztos/udev` never read](https://github.com/Zerone-Laboratories/ztconflo/issues/4)
- [Comment-only fstab halts the pmOS initramfs](https://github.com/Zerone-Laboratories/ztconflo/issues/5)
- [Stale backup GPT on userdata restored over the root](https://github.com/Zerone-Laboratories/ztconflo/issues/6)
- [No qbootctl, so the A/B slot falls back](https://github.com/Zerone-Laboratories/ztconflo/issues/7)
- [GSI 0.2.1 release has no files (404)](https://github.com/Zerone-Laboratories/ztos-build/issues/1)

## References

- ztOS: [ztos.online](https://ztos.online/) · [Supported devices](https://devices.ztos.online/) ·
  [ztOS](https://github.com/Zerone-Laboratories/ztOS) · [ztConflo](https://github.com/Zerone-Laboratories/ztconflo) ·
  [ztos-build](https://github.com/Zerone-Laboratories/ztos-build)
- postmarketOS: [Pixel 3a XL wiki](https://wiki.postmarketos.org/wiki/Google_Pixel_3a_XL_(google-bonito)) ·
  [Pixel 3a wiki](https://wiki.postmarketos.org/wiki/Google_Pixel_3a_(google-sargo)) ·
  [Snapdragon 670](https://wiki.postmarketos.org/wiki/Qualcomm_Snapdragon_670_(SDM670)) ·
  [device-google-bonito](https://gitlab.postmarketos.org/postmarketOS/pmaports/-/tree/master/device/community/device-google-bonito) ·
  [pmbootstrap](https://gitlab.postmarketos.org/postmarketOS/pmbootstrap) ·
  [debug shell](https://wiki.postmarketos.org/wiki/Debug_shell)
- Qualcomm userspace (linux-msm): [rmtfs](https://github.com/linux-msm/rmtfs) · [tqftpserv](https://github.com/linux-msm/tqftpserv) ·
  [qrtr](https://github.com/linux-msm/qrtr) · [qbootctl](https://github.com/linux-msm/qbootctl) · [hexagonrpc](https://github.com/linux-msm/hexagonrpc)
- [Arch Linux ARM](https://archlinuxarm.org/)
- Google: [factory images](https://developers.google.com/android/images) · [flash.android.com](https://flash.android.com/) ·
  [bootloader unlocking](https://source.android.com/docs/core/architecture/bootloader/locking_unlocking)

## Credits

ztOS and ztConflo by [Zerone Laboratories](https://github.com/Zerone-Laboratories). The kernel, the initramfs, the firmware
packaging and the UCM come from [postmarketOS](https://postmarketos.org/). The Qualcomm daemons come from [linux-msm](https://github.com/linux-msm).
Port and write-up by Vachagan Grigoryan.

## License

MIT for the scripts, units and docs here (see [LICENSE](../../LICENSE)). The upstream projects keep their own licenses.
