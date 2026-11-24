# Ubuntu Touch on the 2 GB Lenovo Tab M10 HD (TB-X306F): failed install and full recovery

[Back to README](../README.md) · [Ubuntu Touch guide for the 4 GB model](ubuntu-touch-4gb.md)

**Status (November 2026):** Ubuntu Touch does not work on the 2 GB model. My attempt left the tablet in a boot loop, but it is **fully recovered** to stock Android 10, and **LineageOS 21 now runs on it**. This report covers what broke, the exact steps that unbricked it from Windows and how I installed LineageOS afterwards.

**If you only read one thing:** check your RAM first. If your TB-X306F has 2 GB, do not run the UBports Installer on it. If you already did and it is stuck in a boot loop, skip to [How I recovered it](#how-i-recovered-it).

## Device

| Item | Value |
|---|---|
| Model | Lenovo TB-X306F (Tab M10 HD 2nd Gen, Wi-Fi), codename `amar_row_wifi` |
| RAM / storage | 2 GB / 32 GB eMMC |
| SoC | MediaTek MT6765 (Helio P22T), hardware code `0x766` |
| Stock firmware | Android 10, build `TB-X306F_S120637_230301_BMP` |
| PC | Windows 11 (build 26100) |

## Why Ubuntu Touch fails on this model

- The UBports port targets the **4 GB / 64 GB** variant and requires the **Android 11** stock firmware as its base.
- Lenovo never released Android 11 for the 2 GB variant. Lenovo's own rescue tool (LMSA / Software Fix) only offered Android 10 for my unit.
- The UBports Installer shows a notice that Android 11 is required, but it does not check for it. It installed anyway and the tablet ended up in a boot loop with a black screen.
- Other users on the UBports forum report the same boot loop on 2 GB units.

## What the installer actually changed

After recovering access, I dumped the partitions and compared them with both firmware packages:

| Partition | State after the UBports Installer |
|---|---|
| `preloader`, `lk`, `lk2`, `tee1`, `tee2`, `logo` | Untouched (still stock Android 10) |
| `boot`, `recovery` | Replaced with Ubuntu Touch images |
| `vbmeta` | Replaced (verification disabled) |
| `dtbo` | Replaced with the **Android 11** `dtbo`, which does not match the Android 10 base |
| `super`, `userdata` | Overwritten |
| `para` (misc / bootloader message) | Left with the command `boot-fastboot` |

The low-level bootloaders were intact, which is why the tablet was recoverable. The boot loop came from mixing an Android 10 base with Ubuntu Touch and Android 11 pieces.

## Why the usual tools failed

| Tool | Error | Cause |
|---|---|---|
| LMSA (Software Fix) | Does not detect the tablet | It needs a booting device. A tablet in a boot loop is never detected. |
| SP Flash Tool 5.1924 and 5.2404 | `STATUS_BROM_CMD_SEND_DA_FAIL (0xC0060003)`, or `LIB DA not match` | The SoC has **SBC and DAA enabled** (target config `0xe5`). SP Flash Tool needs an authorized Lenovo service account to send its Download Agent. The `auth_sv5.auth` file in the firmware is not enough. |
| mtkclient on Windows | `USB enumeration error: Operation not supported or unimplemented on this platform`, then `Handshake failed` | The tablet shows up in Device Manager as `MT65xx Preloader` with **no driver**. The libusb build in mtkclient cannot open a driverless device unless the **UsbDk** backend is turned on, and in my install it was turned off. |

## How I recovered it

Everything was done from Windows with [mtkclient](https://github.com/bkerler/mtkclient). No Linux and no Lenovo account were needed.

### 1. Prerequisites

- Python 3.12 and mtkclient (the [community Windows installer](https://github.com/codefl0w/mtkclient-windows-installer) sets up both).
- [UsbDk](https://github.com/daynix/UsbDk/releases) installed. Check it with `UsbDkController -n` while the tablet is connected. You should see a device with ID `0e8d:2000` (preloader) or `0e8d:0003` (BROM).
- **mtkclient must use the UsbDk backend.** In `mtkclient/Library/Connection/usblib.py`, find this line and make sure it is not commented out:

  ```python
  self.backend.lib.libusb_set_option(self.backend.ctx, 1)  # 1 = LIBUSB_OPTION_USE_USBDK
  ```

  Without this line I always got the "USB enumeration error". With it, the handshake worked.
- The **Android 10 stock firmware for your exact build.** I had downloaded it with LMSA before breaking the tablet. LMSA stores it in `C:\ProgramData\RSA\Download\RomFiles\<firmware name>_SVC`. Do not use the UBports Android 11 package on a 2 GB unit.
- A charged battery. After weeks in a boot loop mine was completely flat, so I charged it from a wall charger for a few hours first.

### 2. Enter BROM mode

1. Unplug the USB cable and hold **Power for about 15 seconds** so the tablet is fully off.
2. Hold **Volume Up + Volume Down** together.
3. While still holding them, plug in the USB cable. Keep holding for about **15 seconds**, even after Windows plays the connect sound.

The screen stays **black** the whole time, which is normal. The handshake sometimes fails on the first try: just repeat the steps while mtkclient keeps waiting. When it works, mtkclient prints `BROM mode detected`, runs the Kamakiri payload and bypasses SBC/DAA.

Start mtkclient **before** plugging in the tablet. It waits for the device. A safe first test, which only reads, is:

```
python mtk.py printgpt
```

If a previous session did not end with `reset`, mtkclient may try to reuse it and crash with `unpack requires a buffer of 12 bytes`. Delete the `.state` file in the mtkclient folder and try again.

### 3. Back up the device-specific partitions

These partitions hold the serial number, Wi-Fi MAC address and calibration data. They are **not** included in any firmware package:

```
python mtk.py multi "r nvram,nvdata,nvcfg,proinfo,persist,protect1,protect2,seccfg,frp,para,lk,boot,recovery,vbmeta,dtbo nvram.bin,nvdata.bin,nvcfg.bin,proinfo.bin,persist.bin,protect1.bin,protect2.bin,seccfg.bin,frp.bin,para.bin,lk.bin,boot.bin,recovery.bin,vbmeta.bin,dtbo.bin;r boot1 boot1_preloader.bin --parttype boot1;reset"
```

This took about two minutes (around 400 MB at about 7.5 MB/s). mtkclient splits `multi` commands on spaces, so use file paths **without spaces**.

### 4. Convert the sparse images

`super.img`, `cache.img` and `userdata.img` in the stock firmware are Android **sparse** images. mtkclient writes files byte for byte and does not expand sparse images, so convert them to raw first with any `simg2img` tool:

```
simg2img super.img super.raw        (about 4.5 GB)
simg2img cache.img cache.raw
simg2img userdata.img userdata.raw  (about 3 GB)
```

### 5. Clear the boot command in `para`

The first 32 bytes of `para` held `boot-fastboot`. I zeroed those 32 bytes in a copy of the `para` backup and kept the rest unchanged (`para_clean.bin`).

### 6. Write stock Android 10

I only wrote the partitions the installer had changed. **Do not write `preloader`, `lk` or `tee`** unless you are sure they are damaged: they were fine, and a wrong `preloader` can hard-brick the device.

```
python mtk.py multi "w boot,recovery,vbmeta,dtbo,para,cache,userdata,super boot.img,recovery.img,vbmeta.img,dtbo-verified.img,para_clean.bin,cache.raw,userdata.raw,super.raw;e metadata;reset"
```

This took about 25 minutes (around 8 GB at about 6 MB/s). Erasing `metadata` avoids encryption leftovers from the previous `userdata`.

### 7. First boot

After `reset`, the tablet stayed off. I unplugged the cable and pressed Power: the Lenovo logo appeared, followed by the Android 10 setup wizard. The bootloader is still unlocked, so an "unlocked" warning shows at every boot. That is expected.

## Recommendations

1. **Check your RAM before anything else** (Settings > About tablet, or the box). If it is 2 GB, the UBports port is not meant for it.
2. **Back up your firmware while the tablet still works.** In LMSA, open Rescue and click Download. Then copy the resulting `_SVC` folder somewhere safe. That folder is what made this recovery possible.
3. **Back up the device-specific partitions with mtkclient** (step 3) before experimenting. It takes two minutes.
4. On Windows, use **mtkclient with UsbDk**, not SP Flash Tool. SP Flash Tool cannot get past DAA on this device without a Lenovo service account.
5. If you only want a usable tablet, install LineageOS instead of Ubuntu Touch. See the next section.

## Update: LineageOS 21 works on the 2 GB Wi-Fi model

After the recovery I installed the unofficial **LineageOS 21** build for the sibling model TB-X306X (`amar_row_lte`, release `25.04-rel` from [Bakoubak/lineage_releases](https://github.com/Bakoubak/lineage_releases/releases)) on this 2 GB TB-X306F. It boots to the setup wizard on the first try. The zip only writes `system`, `vendor`, `boot`, `recovery` and `vbmeta`, on top of the stock Android 10 base. It does not touch `preloader`, `lk`, `tee` or `dtbo`.

Steps, starting from stock Android 10 with the bootloader unlocked and USB debugging on:

```
adb reboot bootloader
fastboot --disable-verity --disable-verification flash vbmeta vbmeta.img   (vbmeta.img extracted from the LineageOS zip)
fastboot flash recovery lineage-RECOVERY-21.0-20250421-UNOFFICIAL-amar_row_lte.img
fastboot reboot recovery
```

Then, in LineageOS Recovery:

1. Factory reset > Format data / factory reset.
2. Apply update > Apply from ADB.
3. On the PC: `adb sideload lineage-21.0-20250421-UNOFFICIAL-amar_row_lte.zip`. The tablet shows `Install completed with status 0`.
4. Reboot system now. The first boot took a few minutes.

**Windows fastboot driver tip.** In fastboot mode this tablet shows up as `0E8D:201C` with no driver (Code 28). I installed WinUSB for it with Zadig. After that, `fastboot devices` still found nothing, because Zadig gives the device a random interface GUID. To fix it, run this in an administrator PowerShell, with your device serial in place of `<serial>`, then replug the cable:

```
Set-ItemProperty 'HKLM:\SYSTEM\CurrentControlSet\Enum\USB\VID_0E8D&PID_201C\<serial>\Device Parameters' -Name DeviceInterfaceGUIDs -Type MultiString -Value '{F72FE0D4-CBCB-407d-8814-9ED673D0DD6B}'
```

This sets the standard Android USB interface GUID, so fastboot can find the device.


## Sources

- UBports device page: https://devices.ubuntu-touch.io/device/amar-row-wifi/
- UBports forum, TB-X306F thread: https://forums.ubports.com/topic/9673/lenovo-tab-m10-hd-wifi-x306f
- mtkclient: https://github.com/bkerler/mtkclient
- mtkclient Windows installer (community): https://github.com/codefl0w/mtkclient-windows-installer
- UsbDk: https://github.com/daynix/UsbDk/releases
- Lenovo Rescue and Smart Assistant (LMSA): https://support.lenovo.com/us/en/downloads/ds101291-rescue-and-smart-assistant-lmsa
- Unofficial LineageOS builds for `amar_row_lte`: https://github.com/Bakoubak/lineage_releases/releases
