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
| 14 | No vibration in ztMETRO, though `drv2624:haptics` (`event3`) supports FF_RUMBLE | ztMETRO opens `/dev/input/eventN` itself (`ztmetro.c` `haptics_init`). `event3` is `root:input 0660`, and `alarm` is in `wheel` only: ztConflo's `id alarm \|\| useradd -G wheel,video,input,audio` skips `useradd` because Arch Linux ARM already has `alarm` | `usermod -aG video,input,audio alarm`, plus `70-bonito-haptics.rules` (`uaccess`) |
| 15 | Flashlight tile does nothing | ztMETRO hard-codes the S9's `/sys/class/leds/white:torch`. Mainline names this LED `white:flash`, and its `brightness` is root-only | `90-bonito-flash.rules` (group `video`), and the shell patch in 33 |
| 16 | `ztos-update-check.service` fails; `pacman -Sy`: `Landlock is not supported by the kernel` → `switching to sandbox user 'alpm' failed` | pacman 7.1's download sandbox needs Landlock; `linux-postmarketos-qcom-sdm670` has `# CONFIG_SECURITY_LANDLOCK is not set` | `DisableSandboxFilesystem` in `/etc/pacman.conf` (better: Landlock in the kernel) |
| 17 | Every boot: "Vibration motor / Notification light / Core power: did not start", a "Sensor hub failed" dialog, `ztoscorenetworking-modem` restarting forever (binary missing) | The GSI's `/etc/ztoscoresensorframework/components` and enabled units are the Galaxy S9's | A bonito `components` list; `systemctl disable ztoscorenetworking-modem ztoscorenetworking`; the sensor-hub check is hard-coded in ztoscsf, so `tmpfiles.d/ztos-no-sensorhub.conf` creates its "stopped" mark `/run/ztoscsf-sensorhub.stopped` |
| 18 | ztOSIAL `Display1` shows `Node="auto"`, `MaxLevel=0`; `Haptics1` shows `Device=""` | ztConflo writes `node = "auto"` / `device = "auto"`, but the GSI's `generic.py` takes the value literally (a path / an input name) | Real values in `/vendor/ztos/manifest.toml`. Brightness itself worked already: ztMETRO falls back to `brightnessctl` |
| 19 | Sometimes after boot: no Bluetooth, `ztos-bt-addr`: `Set Public Address for hci0 failed with status 0x11 (Invalid Index)` | `hci0` appears before the WCN3990 firmware setup finishes, and mgmt has no index until then. The unit runs about 1 s too early on a fast boot | `ztos-bt-addr` retries for up to 20 s |
| 20 | Sensor DSP up, but libssc sees no accelerometer, gyro or proximity | `hexagonrpcd -s` served files from its default root `/usr/share/qcom/`, so the DSP's `/mnt/vendor/persist/sensors/registry` lookups found nothing | `-R /usr/share/qcom/sdm670/Google/sargo` in `hexagonrpcd-adsp-sensorspd.service`, as pmOS does |
| 21 | `monitor-sensor`: light only, "No accelerometer", "No proximity sensor" | Arch's `80-iio-sensor-proxy.rules` tags `fastrpc-adsp*` with `ssc-light ssc-compass` only (Alpine's, used by pmOS, tags all four). The node is `fastrpc-adsp-secure`; `fastrpc-adsp` is only a symlink | `81-bonito-ssc-sensors.rules` adds `ssc-accel ssc-proximity` on `fastrpc-adsp*` |
| 22 | No auto brightness; ztSettings shows it off | ztMETRO gets auto brightness from `ztoscsf-sensord`, which talks only to the S9's sensor hub | `ztos-ssc-sensord`: the same control pipe, status file and lux→brightness curve, fed by iio-sensor-proxy. Checked: 91 lux → 621 ‰ → shell level 62 → backlight 634/1023 |
| 23 | Speaker too quiet | Both CS35L36 amps' `BOOST Enable Switch` default to off (`SOC_DAPM_SINGLE_VIRT`), and pmOS's UCM never sets it, so the amps run off the battery (`BST_DIS_VP`) without boost or class H | Boost on, analog gain 19, saved with `alsactl store`; plus a PipeWire smart filter (+8 dB into swh's look-ahead limiter) in front of the speaker only. The mic clipped: PipeWire uses `ADC1 Capture Volume` as its hardware volume, and 100% is 63 (max). `wpctl set-volume @DEFAULT_AUDIO_SOURCE@ 0.60` gives the UCM's 45, and WirePlumber restores it |
| 24 | Android never sleeps or freezes; `ztmetro.log`: `ztos-waydroid-power: No such file`, "Waydroid did not confirm screen-off freeze" | ztMETRO calls `ztos-waydroid-power`, but only `android/waydroid/install.sh` installs it, and nothing runs that script on a GSI (it also rebuilds Android's image) | Install the helpers, user units and sudoers rule from `/usr/share/ztos-waydroid` (step 13) |
| 25 | With those helpers, Android is frozen half-way through booting and never reports ready | ztMETRO asks for "idle" (→ freeze) as soon as it shows its own screens; `ztos-waydroid-boot` didn't hold the launcher's mark that `power.sh` respects. Its boot mark goes to `/run`, which the user can't write | Hold the mark while Android boots; `ztos-waydroid-recover ready` (root) writes the boot mark: [patches/ztos-waydroid-boot.diff](patches/ztos-waydroid-boot.diff), [patches/ztos-waydroid-recover.diff](patches/ztos-waydroid-recover.diff) |
| 26 | `WARNING: mm/shrinker.c:212 set_shrinker_bit` (from Waydroid's CameraService, WPEWebProcess) | Kernel 7.2.3, THP `always`: a huge page faulted in inside a memory cgroup | THP `madvise` (`tmpfiles.d/ztos-thp-madvise.conf`) |
| 27 | Camera shows only its test scene | ztCamera opens `/dev/video0`, a raw CAMSS node, and takes only YUYV. Both cameras need libcamera's software ISP, which gives RGB | [patches/ztcamera-libcamera.diff](patches/ztcamera-libcamera.diff): a libcamera source that reads each camera from `gst-launch-1.0 libcamerasrc` in a helper process (step 14) |
| 28 | Through `libcamerify`: zoomed in about 6×, very noisy, one frame then frozen, no front camera | imx363 has one mode, 4032×3024. libcamera's CPU debayer makes 640×480 by **cropping** the centre. Its GPU (EGL) debayer scales the whole sensor, but stalls after about 30 frames inside an app that also renders with EGL (the eventfd count stays 0). Both cameras list the same CAMSS nodes, so `libcamerify` only ever reaches the first | The helper process: GPU debayer, whole field of view, 1280×960 at 29 fps; `camera-name` picks back or front |
| 29 | Blurry: never in focus | The lens motor (`lc898219xi`, `focus_absolute` 0–4095) is never moved; the simple pipeline has no AF | Contrast autofocus in ztCamera (coarse and fine sweep). Measured: 0 = infinity, 4095 = nearest (a 40 cm target is sharpest at 4095, forwards and backwards) |
| 30 | `dma-heap allocation failure` from the ISP's buffers | libcamera uses the first DMA heap it can open; `reserved` here is small | `71-bonito-dma-heap.rules`: `reserved` root-only, so the system heap is used |
| 31 | After running once, Camera stays black or on its test scene until a reboot | (a) A second libcamera client starting while one streams breaks both ("Failed to setup link … busy"), and CAMSS stays dead until a reboot (rebinding CAMSS or the sensors doesn't help; VFE0 gets 2 interrupts per start). (b) A helper left running after the app was killed kept the camera | A lock (`/tmp/.ztcamera-libcamera.lock`) held by the helper for its lifetime, `PR_SET_PDEATHSIG` so it dies with ztCamera, and a "restart the phone" note after 8 s without frames |
| 32 | Devices and Services: touch screen, graphics, backlight, battery, charger, vibration, modem and flashlight "missing"; five services "not installed" | `ztdevices` checks the S9's driver names (`s6sy761`, panfrost, `s9p-panel`, `max170xx_battery`, `zrn_*`, `white:torch`) and always lists the S9's own units (`zrn-charge`, `zrn-cpuidle`...) | [patches/ztdevices-any-phone.diff](patches/ztdevices-any-phone.diff): device paths may be patterns, with mainline names beside the S9's; an S9 service that isn't installed is left out. 15 of 17 rows now read "working" (step 15) |
| 33 | Flashlight tile drawn faint, does nothing (see 15) | `ztmetro.c` writes only `/sys/class/leds/white:torch/brightness`, at Samsung's level 9 | [patches/ztmetro-torch-led.diff](patches/ztmetro-torch-led.diff): the first `*:torch`, then `*:flash` LED, at its `max_brightness` (255: `leds-qcom-flash`'s torch current) (step 15). Checked: the tile switches `white:flash` between 255 and 0 |
| 34 | Settings: "Samsung Galaxy S9 (SM-G960N starlte)", Exynos 9810, Mali-G72, 3000 mAh, UFS; the big cluster's clock blank and its governor never set | `ztsettings.c` hard-codes the S9, and `cpufreq/policy4` (here the big cluster is `policy6`) | [patches/ztsettings-device-info.diff](patches/ztsettings-device-info.diff): the model from ztOSIAL `Device1.Name`, the rest from `soc0`, the cpufreq policies, the GPU's device tree name, the DSI mode, `charge_full_design` and the root's block device (step 15) |
| 35 | Status bar, Phone and Messages: no modem at all, though ModemManager has it | ztMETRO, ztPhone and ztMessages talk only to ztoscorenetworkingd (`/run/ztoscorenetworking/status`, its control pipe, its SMS and call logs), which drives only the S9's Samsung modem | `ztos-mm-modemd`: the same protocol, from ModemManager (signal, operator, registration, technology, airplane, data, SMS, calls). Checked without a SIM: `sim=absent`, airplane mode on and off, a refused SMS logged as failed |
| 36 | Speaker still too quiet; the volume "stops at 30" | 30/30 is ztMETRO's 100% (Windows Phone's scale). Everything after it is at maximum too: the amps' gain 19, 0 dB digital, 10 V boost (Android's own value), no fault or brownout bits. Loud music already reaches the limiter's −1 dB ceiling at +8 dB, so more gain alone adds ~1 dB | [patches/ztmetro-speaker-boost.diff](patches/ztmetro-speaker-boost.diff): the volume keys go on past 30 to speaker boost 1…10 when `ztos-speaker-boost` is installed. Each step raises a high-pass (60 Hz per step, up to 600 Hz: bass the speaker can't play) and the gain into two limiters, live with `pw-cli set-param` and saved in the filter's config, now in `~/.config/pipewire/pipewire.conf.d/` so no root is needed. Measured on a ringtone, in the speaker's band (above 500 Hz): +1.5 dB at 5, +2.4 dB at 10 (steps 13, 15) |

## What still doesn't work

Checked on the phone on 2026-11-04 (cellular and flashlight on 2026-11-05). The hardware works under
Linux; what's missing is ztOS userspace.

- **Cellular (calls, SMS, data):** `ztos-mm-modemd` (fix 35) connects ModemManager to ztMETRO, Phone and
  Messages, but there was no SIM to test with (ModemManager: `failed: sim-missing`). Signal, SMS, data
  and calls are untested. **Call audio** isn't routed: nothing sets up the voice path on the modem DSP
  (pmOS's UCM has a `VoiceCall` verb, but nothing switches to it), so a call may connect without sound.
  Speaker and mute in the call screen do nothing yet.
- **Camera:** works (fixes 27–31). Colour and low-light noise are the uncalibrated software ISP's (no
  tuning file for imx363/imx355). Running `cam` or another libcamera program while Camera is open can
  stop the camera until a reboot (fix 31); this is upstream (libcamera simple pipeline / CAMSS).
- **Screen rotation:** ztMETRO doesn't rotate at all. The accelerometer works in
  iio-sensor-proxy, but its mount matrix isn't calibrated yet (flat face-up reads "face-down").
- **GPS:** no `/dev/gnss*`. The modem's QMI LOC service is up; it needs ModemManager with a
  working modem state and a ztOSIAL `Location1` backend. ztMETRO has no location users yet.
- **45 s boot splash:** ztMETRO waits up to 45 s for Android to be ready, and Android's cold
  boot on this phone takes longer, so the splash always times out.
- **Not root-caused:** `Atomic commit failed: Device or resource busy` on DSI-1, a few
  seconds after unlocking.
- **Speaker loudness:** with boost alone (fix 23) the speaker stayed quiet, although the amps
  were at their design level during playback: boost engaged (`PWR_CTRL2` = 0x21, 10 V target),
  0 dB digital, no error or brownout flags (`INT1..4`, `VPBR_STATUS` = 0). Android adds a
  loudness stage on top, so the port now does too (+8 dB and a limiter, fix 23; up to +2.4 dB more
  with speaker boost, fix 36). That is about all software can do: the signal is already near full
  scale. A higher boost voltage doesn't help either: these are the 10 V CS35L36s (`CS35L36_10V_L36`),
  and the driver clamps the boost to 10 V whatever the device tree asks (tried 12 V: the register
  stays at `0x96`). There is still no speaker protection (Android's thermal model), so avoid long stretches
  at full volume, above all with boost on.
- **Audio quirk:** pmOS's UCM has only the `VoiceCall` verb. The sink is "Earpiece (L) and
  Speaker (R)", so the left channel of music goes to the earpiece.
- **Not supported:** NFC (`nfc0` is up; ztOS has no NFC stack) and fingerprint.
