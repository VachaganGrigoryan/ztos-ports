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
  udevadm control --reload && udevadm trigger --subsystem-match=misc
  qbootctl -m
  systemctl daemon-reload
  systemctl enable --now qbootctl tqftpserv rmtfs ztos-bt-addr bluetooth \
    hexagonrpcd-adsp-rootpd hexagonrpcd-adsp-sensorspd
  nmcli general reload conf'
```

About 10 seconds later `wlan0` appears. Connect to Wi-Fi from ztOS **Settings**. The files do the following:

| File | What it does |
|---|---|
| `qbootctl` + unit | Marks the boot slot successful at every boot, so the bootloader doesn't fall back to the other slot |
| `ztos-bt-addr` | Gives Bluetooth a stable address, since the WCN3990 has none on mainline |
| `10-stable-wifi-mac.conf` | Stable Wi-Fi MAC per network |
| `hexagonrpcd-*` + udev rules | Serves the sensor DSP |
| UCM files | Speaker and microphone routing |

## 13. Check it

```sh
ssh root@172.16.42.1 'qbootctl | head -5; for r in /sys/class/remoteproc/*; do echo "$(cat $r/name) $(cat $r/state)"; done; ip -br link | grep wlan0; bluetoothctl show | grep Powered'
```

You should see the slot `Successful : 1`, three `running` remote processors, `wlan0`, and `Powered: yes`. Reboot once to confirm everything comes back on its own.

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
