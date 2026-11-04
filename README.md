# ztos-ports

Ports of [ztOS](https://ztos.online/) to phones that aren't officially supported yet. ztOS is a phone OS with a Windows
Phone 8.1–style shell (ztMETRO) on Arch Linux ARM. Each port has an install guide, a list of every fix it needed (with
the cause), and the exact files that go on the phone.

| Phone | Codename | SoC | Status | ztOS | Folder |
|---|---|---|---|---|---|
| Google Pixel 3a XL | `google-bonito` | Snapdragon 670 | Boots to ztMETRO; Wi-Fi, Bluetooth, touch, speaker work | GSI 0.2.1 | [ports/google-bonito](ports/google-bonito/) |

All ports are **experimental**: tested on the units listed and built with
[ztConflo](https://github.com/Zerone-Laboratories/ztconflo) from postmarketOS's mainline ports. Installing erases the
phone. The official list of phones is at [devices.ztos.online](https://devices.ztos.online/).

## Layout

```
ports/<pmos-codename>/
  README.md     status, what works, references
  INSTALL.md    step-by-step guide
  FIXES.md      symptom → cause → fix, for every problem hit
  files/        files installed on the phone, laid out from /
  patches/      patches to ztConflo or other tools, if any
```

## License

MIT (see [LICENSE](LICENSE)). The upstream projects keep their own licenses.
