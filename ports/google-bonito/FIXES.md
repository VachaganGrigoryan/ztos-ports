# Everything that had to be fixed, and why

Setup: Pixel 3a XL (`bonito`, Samsung SOFEF00 panel), bootloader `b4s4-0.4-8048689`. Earlier the phone
ran Droidian. ztOS GSI 0.2.1 + ztConflo `39a9d2d` (GSI pipeline, postmarketOS fallback),
pmaports `b1bc0428`, kernel `linux-postmarketos-qcom-sdm670` 7.2.3. Date 2026-11-04.

## Build time (ztConflo / ztOS)

| # | Symptom | Cause | Fix |
|---|---|---|---|
| 1 | `ztconflo build … --gsi 0.2.1`: HTTP 404 | `ztos-build` v0.2.1 has no release files | Build the GSI: `installer/ztos build --target gsi` |
| 2 | `pmbootstrap install`: `Selected kernel () is not valid for device google-bonito` | bonito has two kernel subpackages (`kernel-sdc`, `kernel-tianma`), and `ztconflo/build.py` writes `kernel = ` (empty) into pmbootstrap.cfg | [patches/ztconflo-kernel-variant.diff](patches/ztconflo-kernel-variant.diff) + `ZT_KERNEL=sdc` |
| 3 | `fastboot boot`: `Error verifying the received boot.img: Invalid Parameter` | deviceinfo has only `deviceinfo_dtb_sdc`/`_tianma`, so the draft entry got `dtb = ""` and the image had no appended DTB | `dtb = "qcom/sdm670-google-bonito-sdc"` in the draft `device.toml` |
| 4 | GSI or pmbootstrap build stops on a missing `uic`, `kpartx`, `img2simg`; pmbootstrap not on PyPI for Python 3.14 | Undocumented host dependencies | `qt6-base pyside6 multipath-tools android-tools`, pmbootstrap from git |

## On the phone

| # | Symptom | Cause | Fix |
|---|---|---|---|
| 5 | initramfs: `ERROR: failed to mount subpartitions!` | An old **backup GPT** at the end of `userdata` survived the sparse flash. The ext4 image zeroes sector 1, and the primary GPT was then rebuilt from the backup over the ext4 superblock (blocks 0–4 rewritten, everything after intact) | Zero the first 34 and last 64 sectors of `userdata`, reflash |
| 6 | initramfs `fail_halt_boot` right after `Mount root partition` | ztOS `/etc/fstab` has only comments. pmOS `init_2nd.sh` then calls `wait_boot_partition` for a `pmOS_boot` that doesn't exist | Add `LABEL=pmOS_root / ext4 rw,relatime 0 0` |
| 7 | Black screen with one white pixel; `greetd` start-limit-hit | `qcom/a630_sqe.fw` and `a630_gmu.bin` missing, so freedreno `get_param` returns `-ENXIO`, EGL fails, and ztMETRO has no renderer | Copy `qcom/ ath10k/ qca/ a300_*` from the pmbootstrap rootfs (the kernel reads `.zst`) |
| 8 | Slot `a` reaches `retry_count=0` and is never `successful`; it would fall back to slot `b` | ztOS has no `qbootctl` | qbootctl 0.2.2 built on the phone, `qbootctl -m` at every boot |
| 9 | No Wi-Fi (`WIFI-HW missing`, no `wlan0`) although `ath10k_snoc` is loaded | The WCN3990 firmware runs on the **modem DSP**, which stays offline because `rmtfs` and `tqftpserv` aren't installed. ztConflo enables them but ships no binaries. (`pd-mapper` and the QRTR name service are in-kernel.) | `rmtfs-git qrtr-git` (ALARM `[aur]`), tqftpserv v1.3 built on the phone |
| 10 | Random Wi-Fi MAC every boot | The WCN3990 has no stored MAC | NetworkManager `wifi.cloned-mac-address=stable` |
| 11 | Audio: `no backend DAIs enabled … missing UCM profile` | `alsa-ucm-conf-qcom-sdm670` (pmOS) not installed | `ucm2/Google/sargo/*` + `ucm2/conf.d/sdm660/` from pmOS. PipeWire profile `VoiceCall (Microphone, Speakers)` |
| 12 | Bluetooth: `hci0` set up but no service, no address | bluez not installed; mainline WCN3990 has no public BD address | `bluez bluez-utils`, plus `ztos-bt-addr` (stable `02:…` address hashed from the serial) |
| 13 | Sensor DSP not served; `/dev/fastrpc-adsp` missing | No `hexagonrpcd`. The device's udev rule was copied to `/vendor/ztos/udev/`, which udev never reads | hexagonrpc v0.4.0 built on the phone, `fastrpc` user, the udev rules in `/etc/udev/rules.d`, `/usr/share/qcom` from pmOS |

## What still doesn't work (ztOS, not drivers)

- **Sensors in the ztOS UI:** `ztoscsf` drives the Galaxy S9's Samsung sensorhub
  (`/sys/class/sec/sensorhub`). The Qualcomm SSC path (hexagonrpcd) runs, but ztOS needs a
  ztOSIAL sensor backend to use it.
- **Cellular:** ztOS's modem service is S9-specific. The modem DSP and `rmnet_ipa0` are up;
  ModemManager over QRTR wasn't tried.
- **Camera:** only the CAMSS nodes exist; there is no camera-sensor driver on mainline sdm670.
