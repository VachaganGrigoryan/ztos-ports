> **This erases the phone.** The current slot's `boot` partition and all of `userdata` are overwritten, so every app, photo and file is lost. The bootloader, the modem, the radio partitions and the other A/B slot are not touched. To go back to Android, flash a [Google factory image](https://developers.google.com/android/images) or use [flash.android.com](https://flash.android.com/).

This is an **experimental cross port** built with [ztConflo](https://github.com/Zerone-Laboratories/ztconflo) from postmarketOS's mainline port. It was tested on a Pixel 3a XL (`bonito`) with the Samsung panel. The Pixel 3a (`sargo`) uses the same SoC and kernel but was **not** tried. Every fix in this guide is explained in [FIXES.md](https://github.com/VachaganGrigoryan/ztos-ports/blob/main/ports/google-bonito/FIXES.md). The files it installs are in [`files/`](https://github.com/VachaganGrigoryan/ztos-ports/tree/main/ports/google-bonito/files).

## What you need

- A Pixel 3a XL whose bootloader can be unlocked (carrier-locked units, such as Verizon, cannot). Tested on the final Android 12 bootloader `b4s4-0.4-8048689`.
- An **x86-64 Arch-based Linux** computer, or a privileged `archlinux` container, with about 30 GB free and Python 3.12 or newer. ztConflo requires Arch.
- `android-tools` (`adb`, `fastboot`) on the computer the phone is plugged into. A USB data cable. The phone charged above 50%.
- About 1.5 hours. The two builds take about 25 minutes and 5 minutes.

## 1. Unlock the bootloader

1. On Android: **Settings → About phone →** tap **Build number** 7 times. Then in **Developer options**, turn on **OEM unlocking** and **USB debugging**.
2. Run the commands below, then confirm on the phone with the volume and power keys. **This wipes the phone.**

```sh
adb reboot bootloader
fastboot flashing unlock
```

Google's docs: [Locking/unlocking the bootloader](https://source.android.com/docs/core/architecture/bootloader/locking_unlocking).

## 2. Prepare the build computer

```sh
sudo pacman -S --needed git python python-pip base-devel util-linux libarchive dosfstools mtools \
  e2fsprogs zstd rsync arch-install-scripts qemu-user-static qemu-user-static-binfmt \
  qt6-base pyside6 multipath-tools android-tools
# pmbootstrap from source: PyPI had no installable build for Python 3.14
pip install --user --break-system-packages git+https://gitlab.postmarketos.org/postmarketOS/pmbootstrap.git
export PATH=$HOME/.local/bin:$PATH
```

- **Why the extra packages:**
  - `qt6-base`/`pyside6` provide `uic` for the SDK apps.
  - `multipath-tools` provides `kpartx` for pmbootstrap.
  - `android-tools` provides `img2simg`.
- **In a container:**
  - Build as a normal user with a `/etc/subuid` and `/etc/subgid` range.
  - Make sure `/proc/sys/fs/binfmt_misc/qemu-aarch64` exists. If it doesn't, mount `binfmt_misc` and register `/usr/bin/qemu-aarch64-static`.
  - If a later `pacman` run reinstalls `shadow`, restore the caps: `setcap cap_setuid=ep /usr/bin/newuidmap; setcap cap_setgid=ep /usr/bin/newgidmap`.

## 3. Build the ztOS system image (GSI)

`ztconflo --gsi 0.2.1` cannot download the GSI: the release has no files (HTTP 404). Build it yourself. It takes about 25 minutes.

```sh
git clone https://github.com/Zerone-Laboratories/ztOS.git
cd ztOS
installer/ztos doctor
installer/ztos build --target gsi --password x
cd ..
# result: ztOS/build/ztos/gsi/ztos-gsi-0.2.1-aarch64.tar.zst
```

## 4. Build the phone image with ztConflo

The Pixel 3a XL shipped with one of **two panels**, and each needs its own kernel package and device tree:
- `sdc`: Samsung SOFEF00
- `tianma`

Start with `sdc`. Step 5 tells you whether it is right. ztConflo picks neither, so two workarounds are needed: a one-line patch, and one line in the device entry.

```sh
git clone https://github.com/Zerone-Laboratories/ztconflo.git
git clone https://github.com/VachaganGrigoryan/ztos-ports.git
cd ztconflo
git apply ../ztos-ports/ports/google-bonito/patches/ztconflo-kernel-variant.diff   # reads the kernel variant from $ZT_KERNEL
python -m ztconflo lookup bonito                                     # expect: eligible: yes
export ZT_KERNEL=sdc
python -m ztconflo build google-bonito --gsi ../ztOS/build/ztos/gsi/ztos-gsi-0.2.1-aarch64.tar.zst --out ./out
```

The build prints `draft registry entry: …/devices/google-bonito/device.toml`, usually under `~/Documents/Projects/ztos-devices/`. In that file, set the device tree, then **run the same build command again**:

```toml
dtb = "qcom/sdm670-google-bonito-sdc"     # or qcom/sdm670-google-bonito-tianma
```

Check that `out/google-bonito/build.json` now shows that `dtb`. Without it the bootloader rejects the image with `Error verifying the received boot.img: Invalid Parameter`.

## 5. Test the kernel without writing anything

Put the phone in fastboot mode: hold **Power + Volume Down** until it shows FASTBOOT MODE.

```sh
fastboot boot out/google-bonito/boot.img
```

Within about 15 seconds the screen should show white text from the postmarketOS initramfs, ending in a **debug shell**. The error `failed to mount subpartitions` is expected at this point. That means the panel choice is right.

If the screen stays black, rebuild with `ZT_KERNEL=tianma` and the `tianma` dtb.

## 6. Clear old partition tables from `userdata`

If the phone ever ran Droidian, postmarketOS or another Linux, `userdata` can hold an old GPT. Its backup copy at the end survives a sparse flash and gets restored over the new root filesystem. Clearing it is harmless, so do it anyway.

The debug shell from step 5 is reachable over USB networking, with the phone at `172.16.42.1`. Connect with `telnet 172.16.42.1`, or `nc 172.16.42.1 23`, and run:

```sh
blkid /dev/mmcblk0p72          # must say PARTLABEL="userdata"
N=$(blockdev --getsz /dev/mmcblk0p72)
dd if=/dev/zero of=/dev/mmcblk0p72 bs=512 count=34 conv=fsync
dd if=/dev/zero of=/dev/mmcblk0p72 bs=512 seek=$((N-64)) count=64 conv=fsync
```

Then hold **Power + Volume Down** to go back to fastboot.

## 7. Flash

```sh
cd out/google-bonito
fastboot getvar current-slot        # e.g. "a"
fastboot flash boot boot.img
fastboot flash userdata rootfs.img  # about 2 minutes
fastboot --set-active=a             # the slot printed above; resets its boot-retry counter
fastboot reboot
```

## 8. First boot: fix fstab and add your SSH key

The first boot stops in the debug shell again. ztOS's `/etc/fstab` contains only comments, so the pmOS initramfs goes looking for a `pmOS_boot` partition that doesn't exist.

The root filesystem is already mounted at `/sysroot`. Over `telnet 172.16.42.1`, run:

```sh
printf 'LABEL=pmOS_root\t/\text4\trw,relatime\t0 0\n' >> /sysroot/etc/fstab
mkdir -p /sysroot/root/.ssh && chmod 700 /sysroot/root/.ssh
echo "ssh-ed25519 AAAA… you@pc" >> /sysroot/root/.ssh/authorized_keys   # your public key
chmod 600 /sysroot/root/.ssh/authorized_keys
sync; umount /sysroot; reboot -f
```

ztOS now boots, but the screen stays **black with one white pixel**. The GPU firmware is missing, which you fix next. SSH works anyway: `ssh root@172.16.42.1`.

## 9. GPU, Wi-Fi and Bluetooth firmware

ztConflo leaves out `linux-firmware-qcom` and friends. Copy them from the pmbootstrap rootfs that the build left on your computer:

```sh
P=~/.cache/ztconflo/pmbootstrap/chroot_rootfs_google-bonito
sudo tar -C $P/lib/firmware -cf - qcom ath10k qca a300_pfp.fw.zst a300_pm4.fw.zst \
  | ssh root@172.16.42.1 'tar -C /usr/lib/firmware --skip-old-files -xf -'
ssh root@172.16.42.1 'systemctl reset-failed greetd; systemctl restart greetd'
```

The ztMETRO lock screen appears. Swipe up, and touch works.

## 10. Packages for the modem, Wi-Fi and Bluetooth

On the Pixel, the Wi-Fi firmware runs on the **modem DSP**. `rmtfs` starts the modem, and `tqftpserv` serves it the Wi-Fi firmware. ztConflo enables both but ships neither.

The phone has no internet yet, so let its `pacman` list the packages, and download them on the computer:

```sh
ssh root@172.16.42.1 'pacman -Sp --needed rmtfs-git qrtr-git bluez bluez-utils meson ninja' > urls.txt
mkdir pkgs && cd pkgs
for u in $(grep ^http ../urls.txt); do curl -fLO "$u" && curl -fLO "$u.sig"; done
scp * root@172.16.42.1:/var/cache/pacman/pkg/
ssh root@172.16.42.1 'pacman -S --needed --noconfirm rmtfs-git qrtr-git bluez bluez-utils meson ninja'
cd ..
```

These are signed [Arch Linux ARM](https://archlinuxarm.org/) packages. `rmtfs-git` and `qrtr-git` come from its `[aur]` repository.

## 11. Build tqftpserv, qbootctl and hexagonrpc on the phone

None of these is packaged for Arch Linux ARM. Build the pinned upstream releases with the GSI's own `gcc`:

```sh
git clone --depth 1 -b v1.3   https://github.com/linux-msm/tqftpserv.git
git clone --depth 1 -b 0.2.2  https://github.com/linux-msm/qbootctl.git
git clone --depth 1 -b v0.4.0 https://github.com/linux-msm/hexagonrpc.git
tar --exclude .git -cf - tqftpserv qbootctl hexagonrpc | ssh root@172.16.42.1 '
  mkdir -p /usr/local/src && tar -C /usr/local/src -xf - && cd /usr/local/src &&
  for p in tqftpserv qbootctl hexagonrpc; do
    (cd $p && meson setup build --prefix=/usr/local --buildtype=release && ninja -C build install) || exit 1
  done'
```

## 12. Install this repo's files and turn everything on

```sh
cd ztos-ports/ports/google-bonito
COPYFILE_DISABLE=1 tar -C files -cf - . | ssh root@172.16.42.1 'tar -C / --no-same-owner -xf -'
P=~/.cache/ztconflo/pmbootstrap/chroot_rootfs_google-bonito
sudo tar -C $P/usr/share -cf - qcom alsa/ucm2/Google/sargo alsa/ucm2/conf.d/sdm660 \
  | ssh root@172.16.42.1 'tar -C /usr/share --skip-old-files -xf -'
ssh root@172.16.42.1 '
  ldconfig
  getent passwd fastrpc >/dev/null || useradd -r -U -M -d / -s /usr/bin/nologin fastrpc
  usermod -aG video,input,audio alarm
  sed -i "s/^#DisableSandboxFilesystem/DisableSandboxFilesystem/" /etc/pacman.conf
  M=/vendor/ztos/manifest.toml
  sed -i -e "s|^node = \"auto\"|node = \"/sys/class/backlight/ae94000.dsi.0\"|" \
         -e "s|^device = \"auto\"|device = \"drv2624:haptics\"|" $M
  grep -q "^torch = " $M || sed -i "/^\[ial.Lights1\]/a torch = \"white:flash\"" $M
  udevadm control --reload
  udevadm trigger --subsystem-match=misc --subsystem-match=input --subsystem-match=leds
  qbootctl -m
  systemctl daemon-reload
  systemctl disable --now ztoscorenetworking-modem ztoscorenetworking
  systemctl enable --now qbootctl tqftpserv rmtfs ztos-bt-addr bluetooth \
    hexagonrpcd-adsp-rootpd hexagonrpcd-adsp-sensorspd
  nmcli general reload conf'
```

Reboot afterwards. The new groups only reach ztMETRO when `alarm` logs in again.

About 10 seconds later `wlan0` appears. Connect to Wi-Fi from ztOS **Settings**. The files do the following:

| File | What it does |
|---|---|
| `qbootctl` + unit | Marks the boot slot successful at every boot, so the bootloader doesn't fall back to the other slot |
| `ztos-bt-addr` | Gives Bluetooth a stable address, since the WCN3990 has none on mainline |
| `10-stable-wifi-mac.conf` | Stable Wi-Fi MAC per network |
| `hexagonrpcd-*` + udev rules | Serves the sensor DSP |
| UCM files | Speaker and microphone routing |
| `usermod -aG video,input,audio alarm` | ztConflo never adds these groups, so ztMETRO can't open the vibration motor or switch the torch |
| `70-bonito-haptics.rules` | Gives the logged-in user the vibration motor (`drv2624`) |
| `90-bonito-flash.rules` | Lets group `video` switch the flash LED as a torch |
| `DisableSandboxFilesystem` in `pacman.conf` | pacman 7's download sandbox needs Landlock, which this kernel lacks, so updates fail without it |
| `manifest.toml` edits | Names the real backlight, vibration motor and torch for ztOSIAL instead of `"auto"` |
| `ztoscoresensorframework/components` | Watches this phone's services instead of the Galaxy S9's, so no false "did not start" banners. Devices and Services lists the same ones |
| `tmpfiles.d/ztos-no-sensorhub.conf` | Marks the (absent) Samsung sensor hub as stopped at boot, so ztoscsf doesn't show the "Sensor hub failed" dialog |
| `disable ztoscorenetworking*` | The Galaxy S9 modem daemons. Their binary isn't in the GSI, so they restart forever |
| `81-bonito-ssc-sensors.rules` | Lets iio-sensor-proxy use the accelerometer and proximity sensor behind the sensor DSP, not just the light sensor |
| `ztos-ssc-sensord` + unit | Auto brightness: does ztoscsf-sensord's job (which only knows the S9's sensor hub) from iio-sensor-proxy, with the same protocol and curve |
| `71-bonito-dma-heap.rules` | Makes the small `reserved` DMA heap root-only, so libcamera uses the system heap for camera buffers (step 14) |
| `tmpfiles.d/ztos-thp-madvise.conf` | Transparent huge pages only on request: with `always`, kernel 7.2.3 warns in `set_shrinker_bit` when apps fault them in |

## 13. Sensors, modem, cameras, Android and the speaker

This step needs the Wi-Fi from step 12. All the packages are in Arch Linux ARM's `[extra]`.

```sh
ssh root@172.16.42.1 '
  pacman -S --needed --noconfirm modemmanager libqmi iio-sensor-proxy libssc \
    libcamera libcamera-ipa libcamera-tools swh-plugins
  udevadm trigger --action=change --subsystem-match=misc --sysname-match="fastrpc-adsp*"
  systemctl enable --now ModemManager ztos-ssc-sensord ztos-mm-modemd
  systemd-tmpfiles --create
  W=/usr/share/ztos-waydroid
  install -Dm755 $W/power.sh /usr/local/bin/ztos-waydroid-power
  install -Dm755 $W/launch.sh /usr/local/bin/ztos-waydroid-launch
  install -Dm755 $W/recover.sh /usr/local/libexec/ztos-waydroid-recover
  install -Dm755 $W/audio-recover.py /usr/local/libexec/ztos-waydroid-audio
  install -Dm755 $W/ztos-waydroid-boot /usr/local/libexec/ztos-waydroid-boot
  install -Dm644 $W/ztos-waydroid-boot.service /etc/systemd/user/ztos-waydroid-boot.service
  install -Dm644 $W/ztos-waydroid-session.service /etc/systemd/user/ztos-waydroid-session.service
  sed -n "/^\$ZTOS_USER ALL=(root) NOPASSWD: \/usr\/local\/libexec\/ztos-waydroid-recover audio/p" $W/install.sh \
    | sed "s/^\$ZTOS_USER/alarm/" > /tmp/zw && visudo -cf /tmp/zw && install -m440 /tmp/zw /etc/sudoers.d/ztos-waydroid
  systemctl --global enable ztos-waydroid-boot.service
  for a in Top Bottom; do
    amixer -q -D hw:0 cset "iface=MIXER,name=$a BOOST Enable Switch" on
    amixer -q -D hw:0 cset "iface=MIXER,name=$a Analog PCM Volume" 19
  done
  alsactl store 0
  U="sudo -u alarm XDG_RUNTIME_DIR=/run/user/1000"
  L=60-bonito-speaker-loudness.conf; rm -f /etc/pipewire/pipewire.conf.d/$L
  $U install -Dm644 /usr/local/share/ztos-bonito/$L /home/alarm/.config/pipewire/pipewire.conf.d/$L
  $U systemctl --user restart pipewire pipewire-pulse wireplumber
  sleep 3; $U wpctl set-volume @DEFAULT_AUDIO_SOURCE@ 0.60'
ssh root@172.16.42.1 'patch /usr/local/libexec/ztos-waydroid-boot' < patches/ztos-waydroid-boot.diff
ssh root@172.16.42.1 'patch /usr/local/libexec/ztos-waydroid-recover' < patches/ztos-waydroid-recover.diff
```

| What | Why |
|---|---|
| ModemManager + libqmi | Finds the modem over QRTR (`mmcli -L`) |
| `ztos-mm-modemd` + unit | The modem for ztMETRO, Phone and Messages: does ztoscorenetworkingd's job (which only knows the S9's Samsung modem) through ModemManager, with the same status file, control pipe and SMS/call logs. Without a SIM it reports `sim=absent`; calls, SMS and data are untested, and call audio isn't routed |
| iio-sensor-proxy + libssc | Reads the light, accelerometer and proximity sensors from the sensor DSP (`monitor-sensor`) |
| libcamera | Both cameras through the software ISP: `cam -l`, `cam -c1 -C30 -F/tmp/f#.ppm -s pixelformat=RGB888,width=1280,height=720` |
| The Waydroid helpers | ztMETRO runs `ztos-waydroid-power` at every screen-off and app switch, but nothing in the GSI installs it (only `install.sh`, which also rebuilds Android's image). Without it Android never sleeps or freezes. This installs only the helpers, units and sudoers rule |
| `ztos-waydroid-boot.diff`, `ztos-waydroid-recover.diff` | Holds the launcher's mark while Android boots, so ztMETRO's "idle" doesn't freeze it half-way. The boot mark goes to `/run`, which is root's, so `ztos-waydroid-recover ready` (root, through the sudoers rule) writes it |
| `60-bonito-speaker-loudness.conf` + swh-plugins | Louder speaker the way Android does it: +8 dB into a look-ahead limiter (peaks stay under −1 dBFS). A WirePlumber smart filter in front of the speaker sink only, so headphones bypass it. It's in the user's PipeWire config, so `ztos-speaker-boost` (the volume keys past 30, step 15) can add a high-pass (up to 600 Hz) and more gain into it and a second limiter, without root or a restart. Delete it and restart pipewire to switch the filter off |
| Speaker boost | The CS35L36 amps start with their boost converter off, so the speaker runs straight off the battery and is quiet. The device tree sets the boost (10 V, 2.6 A) as Android does. Analog gain is at its maximum, 19. The built-in mic: PipeWire drives `ADC1 Capture Volume` as the mic's hardware volume, and at 100% it is 63 (max) and clips; 0.60 gives the UCM's 45, and WirePlumber keeps it |

## 14. Camera

ztCamera only knows the Galaxy S9's camera driver and plain YUYV V4L2 devices. Both cameras work on mainline,
but only through libcamera's software ISP. [patches/ztcamera-libcamera.diff](patches/ztcamera-libcamera.diff)
adds a libcamera source to ztCamera:

- it reads each camera from `gst-launch-1.0 libcamerasrc` in a helper process. libcamera's GPU debayer runs
  there and scales the whole sensor; inside ztCamera it would share EGL with the viewfinder and stall;
- the facing button switches between back and front (mirrored);
- autofocus: a contrast sweep of the lens motor when Camera opens or you tap the picture. The focus dial's
  macro/normal/infinity set fixed positions;
- digital zoom 1×/2×/4× on the button next to the camera roll;
- one camera user at a time, with a lock, and a helper that dies with the app. A second libcamera
  program starting while one streams stops both, and only a reboot brings the camera back.

Build it on the phone from ztOS `7f42ddc`, the commit the GSI 0.2.1 apps match:

```sh
ssh root@172.16.42.1 'pacman -S --needed --noconfirm gst-plugin-libcamera'
git clone https://github.com/Zerone-Laboratories/ztOS.git && git -C ztOS checkout 7f42ddc
tar -C ztOS -cf - ztmetro | ssh root@172.16.42.1 'mkdir -p /usr/local/src/ztOS && tar -C /usr/local/src/ztOS -xf -'
ssh root@172.16.42.1 'cd /usr/local/src/ztOS && patch -p1' < patches/ztcamera-libcamera.diff
ssh root@172.16.42.1 '
  cd /usr/local/src/ztOS/ztmetro && make camera/ztcamera
  [ -e /usr/local/bin/ztcamera.gsi ] || cp -a /usr/local/bin/ztcamera /usr/local/bin/ztcamera.gsi
  install -Dm755 camera/ztcamera /usr/local/libexec/ztcamera
  cat > /usr/local/bin/ztcamera <<EOF
#!/bin/sh
# ztCamera on the Pixel 3a XL: libcamera in a GStreamer helper (GPU debayer, whole sensor),
# back camera imx363 (mounted at 90: 270 clockwise makes it upright), front imx355 (90, mirrored).
# The voice-coil lens: 0 is infinity, 4095 its nearest.
B=/base/soc@0/cci@ac4a000/i2c-bus@0/camera@1a
F=/base/soc@0/cci@ac4a000/i2c-bus@1/camera@1a
exec /usr/local/libexec/ztcamera --libcamera "\$B:270" --libcamera "\$F:90:mirror" --focus 0,1500,4095 "\$@"
EOF
  chmod 755 /usr/local/bin/ztcamera
  udevadm trigger --subsystem-match=dma_heap'
```

`71-bonito-dma-heap.rules` (from step 12) makes `/dev/dma_heap/reserved` root-only. libcamera takes the first
heap it can open, and that small region fails the ISP's buffers (`dma-heap allocation failure`). The status
line then reads `camera · 1280x960 · 29 fps`. To go back: `mv /usr/local/bin/ztcamera.gsi /usr/local/bin/ztcamera`.

- **Picture quality:** there's no libcamera tuning for imx363/imx355 (`uncalibrated.yaml`), so colour is
  plain and dim rooms are noisy (the sensor runs at full gain).
- **If the picture goes black** and the note says "no picture from the camera", restart the phone (see above).

## 15. Devices and Services, Settings and the flashlight tile

Three ztOS apps describe the Galaxy S9 instead of the phone they run on:

- **Devices and Services** (`ztdevices`) reports a working touch screen, GPU, battery, charger, vibration
  motor, modem, flashlight and backlight as missing, because it looks for the S9's driver names. It also
  lists five S9-only services as "not installed";
- **Settings** says "Samsung Galaxy S9", Exynos 9810, Mali-G72 and 3000 mAh;
- ztMETRO's **flashlight tile** looks only for the S9's LED (`white:torch`).

[patches/ztmetro-speaker-boost.diff](patches/ztmetro-speaker-boost.diff) lets the volume go on past
30/30 to **speaker boost** 1…10, when `/usr/local/bin/ztos-speaker-boost` is installed (step 13 does that).
30/30 is already 100%. Each boost step cuts more bass and pushes harder into the speaker's limiters; at 10
the speaker's band is about 2.4 dB louder, with audible squashing. Stepping back to 30 or below turns it off.
The speaker has no thermal protection, so avoid long stretches at full boost.

[patches/ztdevices-any-phone.diff](patches/ztdevices-any-phone.diff) lets ztdevices take patterns
(`/sys/class/backlight/*`, `*battery*`, `*:flash`) and hides an S9 service that isn't installed.
[patches/ztsettings-device-info.diff](patches/ztsettings-device-info.diff) reads the model from ztOSIAL
(`Device1.Name`) and the SoC, CPU clusters, GPU, panel, battery and storage from sysfs.
[patches/ztmetro-torch-led.diff](patches/ztmetro-torch-led.diff) makes the tile use the first `*:torch`
or `*:flash` LED, at its `max_brightness`. On the S9 all three fall back to what they showed before.

Build them on the phone from ztOS `7f42ddc`, in the tree from step 14:

```sh
for p in ztdevices-any-phone ztsettings-device-info ztmetro-torch-led ztmetro-speaker-boost; do
  ssh root@172.16.42.1 'cd /usr/local/src/ztOS && patch -p1' < patches/$p.diff
done
ssh root@172.16.42.1 '
  cd /usr/local/src/ztOS/ztmetro && make devices/ztdevices settings/ztsettings ztmetro
  for a in ztdevices:devices/ztdevices ztsettings:settings/ztsettings ztmetro:ztmetro; do
    n=${a%%:*}
    [ -e /usr/local/bin/$n.gsi ] || cp -a /usr/local/bin/$n /usr/local/bin/$n.gsi
    install -m755 ${a#*:} /usr/local/bin/$n
  done'
```

The new shell takes over at the next login (or reboot). Devices and Services should then show every row
working except **Notification light** (this phone has none) and **Fingerprint** (not supported).
To go back, move each `.gsi` file over its program.

## 16. Check it

```sh
ssh root@172.16.42.1 'qbootctl | head -5; for r in /sys/class/remoteproc/*; do echo "$(cat $r/name) $(cat $r/state)"; done; ip -br link | grep wlan0; bluetoothctl show | grep Powered
  id alarm; ls -l /proc/$(pidof ztmetro)/fd | grep -c event3; systemctl --failed --no-legend
  grep -E "^(sim|registered)=" /run/ztoscorenetworking/status'
```

You should see:
- the slot `Successful : 1`;
- three `running` remote processors;
- `wlan0`;
- `Powered: yes`;
- `alarm` in `video`, `input` and `audio`;
- `1` (ztMETRO holds the vibration motor);
- no failed units;
- `sim=absent` without a SIM (`ready` and `registered=home` with one).

Reboot once to confirm everything comes back on its own.

## If something goes wrong

| What you see | What to do |
|---|---|
| `fastboot boot`: `Error verifying the received boot.img: Invalid Parameter` | The boot image has no device tree. Set `dtb =` in the draft `device.toml` and rebuild (step 4). |
| `pmbootstrap … Selected kernel () is not valid for device google-bonito` | Apply the patch and `export ZT_KERNEL=sdc` (or `tianma`) (step 4). |
| `fastboot boot` works but the screen stays black | Wrong panel. Rebuild with `tianma`. |
| Debug shell, `ERROR: failed to mount subpartitions!` | An old GPT is on `userdata`. Do step 6, then flash `userdata` again. |
| Debug shell right after `Mount root partition` | The fstab fix (step 8). |
| Black screen with one white pixel, SSH works | GPU firmware missing (step 9). `~alarm/.local/state/ztmetro.log` says `failed to create the renderer`. |
| No `wlan0`, Settings shows no Wi-Fi | `rmtfs`/`tqftpserv` not running. `cat /sys/class/remoteproc/*/state` must show `running` for `4080000.remoteproc` (steps 10–12). |
| The phone boots the old system, or the bootloader says "no valid slot" | In fastboot: `fastboot --set-active=a`. Then enable `qbootctl` (step 12). |
| `pmbootstrap` not found, or `kpartx`/`img2simg`/`uic` missing | Install the packages from step 2. |
