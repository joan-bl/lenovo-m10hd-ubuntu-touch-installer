# Ubuntu Touch on the Lenovo Tab M10 HD 2nd Gen (TB-X306F), 4 GB model

> **Only for the 4 GB / 64 GB model.** On the 2 GB model this procedure ends in a boot loop. See the [test report](test-report-2gb.md).
>
> I could only test this on a 2 GB unit. The steps follow the [official UBports device page](https://devices.ubuntu-touch.io/device/amar-row-wifi/), plus what I learned about this tablet's tools on Windows.

The UBports page lists three requirements, in this order:

1. The device is a Lenovo Tab M10 HD Wi-Fi (TB-X306F).
2. It runs the **Android 11** stock firmware.
3. The bootloader is unlocked.

After that, the UBports Installer does the rest.

## What you need

- A TB-X306F with **4 GB RAM / 64 GB storage**. Check in **Settings > About tablet**.
- A Windows, Linux or macOS PC with a USB data cable. These notes use Windows.
- [Android Platform Tools](https://developer.android.com/tools/releases/platform-tools). Extract the zip and run `adb` and `fastboot` from that folder. To open a command window there on Windows, type `cmd` in the Explorer address bar. You don't need to change `PATH`.
- [UBports Installer](https://devices.ubuntu-touch.io/installer/).
- The battery charged above 50%.

## 1. Back up your stock firmware

Do this **before** you change anything. It is the only easy way back if something goes wrong.

1. Install [Lenovo Rescue and Smart Assistant (LMSA)](https://support.lenovo.com/us/en/downloads/ds101291-rescue-and-smart-assistant-lmsa). It is also called Software Fix.
2. Connect the tablet while it still boots normally. Open **Rescue**, select your tablet and click **Download**. Do not flash anything.
3. LMSA saves the firmware in `C:\ProgramData\RSA\Download\RomFiles\<firmware name>_SVC`. Copy that folder somewhere safe.

Optional, but recommended: back up the device-specific partitions (serial number, Wi-Fi MAC, calibration data) with mtkclient. These are not part of any firmware package. See [step 3 of the recovery guide](test-report-2gb.md#3-back-up-the-device-specific-partitions).

## 2. Install the Android 11 stock firmware

If **Settings > About tablet** already shows Android 11, skip this step.

The UBports page links this firmware package: [Firmware.tar.xz](https://github.com/rubencarneiro/amar_row_wifi/releases/download/1.0/Firmware.tar.xz) (build `TB_X306F_S230626_230530_BMP`).

There are three ways to flash it:

- **LMSA (recommended).** In issue #1, osnard reports flashing Android 11 with LMSA by manually selecting the X306F firmware. If LMSA does not offer Android 11 for your unit, you can replace the files it downloaded to `RomFiles\<firmware name>_SVC` with the files from `Firmware.tar.xz` (same file names), then run Rescue. Other users report this works, but I have not verified it.
- **SP Flash Tool** (what the UBports page mentions). Use the `MT6765_Android_scatter.txt` file from the package (not `MT6762`) and choose **Download Only**. On my unit it always failed with `STATUS_BROM_CMD_SEND_DA_FAIL (0xC0060003)`. This tablet has SBC and DAA enabled, so SP Flash Tool needs an authorized Lenovo service account. Expect the same on yours.
- **mtkclient.** It works without a Lenovo account. See the [recovery guide](test-report-2gb.md#how-i-recovered-it) for the method. Write the Android 11 images instead of the Android 10 ones.

In every case, **do not flash `preloader`** unless you know it matches your hardware exactly.

## 3. Unlock the bootloader

This **erases all data** on the tablet.

1. Go to **Settings > About tablet** and tap **Build number** 7 times.
2. In **Settings > System > Developer options**, enable **OEM unlocking** and **USB debugging**.
3. Connect the tablet and run `adb devices`. Accept the prompt on the tablet. Then run:

   ```
   adb reboot bootloader
   fastboot flashing unlock
   ```

4. Confirm on the tablet with the volume keys. Then check:

   ```
   fastboot getvar unlocked     (should print "unlocked: yes")
   fastboot reboot
   ```

The tablet shows an "unlocked" warning at every boot from now on. That is normal.

**If `fastboot devices` shows nothing on Windows:** in fastboot mode the tablet appears in Device Manager as `Android` (USB ID `0E8D:201C`) with no driver. Install WinUSB for it with [Zadig](https://zadig.akeo.ie/), then set the Android interface GUID so fastboot can find it. The details are in the [test report](test-report-2gb.md#update-lineageos-21-works-on-the-2-gb-wi-fi-model) under "Windows fastboot driver tip".

## 4. Install Ubuntu Touch

1. After the unlock, finish the Android setup and enable **USB debugging** again.
2. Open the UBports Installer and select **Lenovo Tab M10 HD Wifi**.
3. Pick the **stable** channel (Ubuntu Touch 24.04 at the time of writing). Leave **wipe userdata** enabled.
4. Follow the on-screen steps and do not disconnect the tablet until the installer finishes.

According to the UBports page, Wi-Fi, Bluetooth, the camera, audio, rotation, charging and Waydroid (Android apps) work. GPS works but is slow to get a fix. The flashlight, fingerprint reader, wired external monitor and PIN unlock are listed as unsupported.

## If something goes wrong

- **Boot loop or black screen after the installer:** see [How I recovered it](test-report-2gb.md#how-i-recovered-it). It works from Windows, even when LMSA no longer detects the tablet.
- **The installer stays on "rebooting" for a long time:** check that you really have the 4 GB model and that Android 11 was installed first.
- Ask in the [UBports forum category for this tablet](https://forums.ubports.com/category/121/lenovo-tab-m10-hd-2nd-gen).

## Sources

- UBports device page: https://devices.ubuntu-touch.io/device/amar-row-wifi/
- Device source: https://gitlab.com/ubports/porting/community-ports/android11/lenovo-tab-m10-hd/lenovo-m10-hd
- UBports forum: https://forums.ubports.com/category/121/lenovo-tab-m10-hd-2nd-gen
- Issue #1 (LMSA tip): https://github.com/joan-bl/lenovo-m10hd-ubuntu-touch-installer/issues/1
