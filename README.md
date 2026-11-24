# Lenovo Tab M10 HD 2nd Gen (TB-X306F): Ubuntu Touch, boot-loop recovery and LineageOS

These are my notes from trying to install a lighter system on a Lenovo Tab M10 HD 2nd Gen (TB-X306F, Wi-Fi). I tried Ubuntu Touch, ended up in a boot loop, recovered the tablet from Windows, and now it runs LineageOS 21.

## Which page do you need?

| Your situation | Read this |
|---|---|
| You have the **4 GB / 64 GB** model and want Ubuntu Touch | [Ubuntu Touch guide (4 GB model)](docs/ubuntu-touch-4gb.md) |
| You have the **2 GB / 32 GB** model | Do **not** install Ubuntu Touch. See [LineageOS 21 on the 2 GB model](docs/test-report-2gb.md#update-lineageos-21-works-on-the-2-gb-wi-fi-model). |
| Your tablet is stuck in a **boot loop or black screen** after flashing | [How I recovered it](docs/test-report-2gb.md#how-i-recovered-it) (Windows, mtkclient, no Lenovo account) |
| You want the full story and the error messages | [Test report (2 GB model)](docs/test-report-2gb.md) |

## Check your RAM first

Go to **Settings > About tablet**, or look at the box. The TB-X306F was sold with **2 GB** or **4 GB** of RAM, and the model number is the same for both.

- The UBports port of Ubuntu Touch targets the **4 GB / 64 GB** model running the **Android 11** stock firmware.
- Lenovo never released Android 11 for the 2 GB model. The UBports Installer does not check this. On a 2 GB unit it installs anyway, and the tablet ends up in a boot loop.

## Results on my tablet (2 GB / 32 GB, Android 10)

| What | Result |
|---|---|
| Ubuntu Touch with the UBports Installer | ❌ Boot loop with a black screen |
| Recovery to stock Android 10 with SP Flash Tool | ❌ `STATUS_BROM_CMD_SEND_DA_FAIL`. The SoC has SBC and DAA enabled. |
| Recovery to stock Android 10 with mtkclient + UsbDk on Windows | ✅ Fully recovered |
| LineageOS 21 (unofficial `amar_row_lte` build) | ✅ Boots and runs |

## What's in this repository

```
README.md                    This page
docs/ubuntu-touch-4gb.md     Ubuntu Touch install guide for the 4 GB model
docs/test-report-2gb.md      Test report: boot loop, recovery and LineageOS 21 on the 2 GB model
```

There are no firmware or tool files here. Each guide links to the official downloads.

## Tools mentioned

- [Android Platform Tools](https://developer.android.com/tools/releases/platform-tools) (ADB and Fastboot)
- [UBports Installer](https://devices.ubuntu-touch.io/installer/)
- [mtkclient](https://github.com/bkerler/mtkclient) and [UsbDk](https://github.com/daynix/UsbDk/releases), for recovering MediaTek devices
- [Zadig](https://zadig.akeo.ie/), for the Windows fastboot driver
- [Lenovo Rescue and Smart Assistant (LMSA)](https://support.lenovo.com/us/en/downloads/ds101291-rescue-and-smart-assistant-lmsa), for backing up and restoring stock firmware

## Credits

- Ubuntu Touch port for this tablet: [Rúben Carneiro and the UBports community](https://devices.ubuntu-touch.io/device/amar-row-wifi/)
- Unofficial LineageOS builds for `amar_row_lte`: [Bakoubak/lineage_releases](https://github.com/Bakoubak/lineage_releases/releases)
- The LMSA tip for flashing stock firmware: osnard, in [issue #1](https://github.com/joan-bl/lenovo-m10hd-ubuntu-touch-installer/issues/1)

---

**Disclaimer:** flashing firmware can make your device unusable, and it erases all your data. Back up first. Follow these notes at your own risk.
