# Flashing and rooting the Magcubic HY300 Pro (Allwinner H726, Android 14)

This guide covers the whole process: flashing the `update` folder via USB, enabling ADB, and
rooting the projector with Magisk by patching `init_boot_a`. Credit for the firmware and the
original USB-flash steps goes to the person who posted the ROM; the rooting part (steps 3 onward)
was put together and tested on a real HY300 Pro H726 unit.

## Applies to

- Magcubic HY300 Pro / HY300 Ultra with the **Allwinner H726** chip, Android 14, 32-bit kernel
  (`armeabi-v7a`). Check with:
  ```
  adb shell getprop ro.board.platform      # hermes
  adb shell getprop ro.hardware            # sun50iw15p1
  adb shell getprop ro.product.cpu.abi     # armeabi-v7a
  ```
- The bootloader must already be unlocked: `ro.boot.verifiedbootstate=orange` and
  `ro.boot.flash.locked=0`. If yours reports anything else, **stop here** — this guide is not for
  your unit.
- The **HY300 Pro+ with H713** (Android 11) is a different chip and uses a different method
  (fastboot). Look for the H713-specific guide instead.

## Part 1 — Flash the firmware via USB

1. Rename the firmware file to `update.img`.
2. Create a new folder called `update`.
3. Copy `update.img` and `auto_update.txt` into that `update` folder.
4. Format a USB drive as **FAT32**.
5. Copy the whole `update` folder onto the root of the USB drive.
6. Turn off the projector.
7. Insert the USB drive into the projector, then connect the power cable.
8. Press and hold the Power button until the indicator light changes from red to blue.
9. A progress bar will appear on screen showing the firmware update status.
10. Wait for the installation to complete.
11. Once the progress bar reaches 100%, the projector will restart automatically.
12. After it restarts, eject the USB drive.

## Part 2 — Enable Developer Options and USB debugging

1. On the projector, go to **Settings > About / System info**.
2. Tap on the device/projector name several times until it says Developer Options have been
   enabled.
3. Go back to the main Settings screen; a new **Developer Options** entry should now be visible.
4. Inside Developer Options, turn on **USB debugging** (and **ADB** if it's listed separately).
5. Connect the projector to your PC with a **USB-A to USB-A data cable**, plugged directly into
   the PC (no hub).
6. On the PC, with [Android platform tools](https://developer.android.com/tools/releases/platform-tools)
   installed, run:
   ```
   adb devices
   ```
   Accept the "Allow USB debugging?" prompt that appears on the projector's screen. The device
   should now show up as `device`.

## Part 3 — Root with Magisk (patch init_boot_a)

Some of these ROMs already have the Magisk app installed, but root is **not** actually active (on
the projector, `su` reports "not found" or "inaccessible", and Magisk shows `N/A`). This part
patches your own `init_boot_a` with Magisk and writes it back, with a guard that refuses to write
unless your partition still matches your original backup byte for byte. No need to go back into
FEL mode for this — on the H726, `/dev/block/mmcblk0` is both readable and writable over plain
ADB.

### You will need

- The same PC with `adb.exe` set up from Part 2.
- About 8–10 GB of free disk space for the backup.
- The Magisk APK, downloaded **only** from
  https://github.com/topjohnwu/Magisk/releases (latest non-beta/non-canary release).

### Files in this guide

- `root_hy300.bat` — runs the whole process step by step, with checks at every stage.
- `device_write.sh` — the script that runs inside the projector and does the guarded write.
- `restore_stock.bat` — if anything goes wrong, puts your original `init_boot_a` back.

Put all three files in the same folder.

### Steps

1. With the projector connected and `adb devices` showing it as `device`, double-click
   `root_hy300.bat`.
2. The script will:
   - Check that your unit is compatible (H726, slot A, unlocked, 32-bit). If anything doesn't
     check out, it stops right there without touching anything.
   - Automatically locate your `init_boot_a` partition (its position isn't the same on every
     unit, so the script works it out live instead of using a fixed number).
   - Offer to make a **full storage backup** (recommended — takes 10–20 minutes). This is your
     most complete safety net.
   - Extract and save **your own copy** of `init_boot_a` before touching anything, and compute
     its SHA256.
   - Ask for the path to your Magisk APK and show you its SHA256. **You compare it yourself**
     against the one shown on the GitHub release page, under "Assets". The script will not
     continue without your explicit confirmation.
   - Patch that copy using the Magisk APK's own tools, run inside the projector.
   - Before writing anything, ask you to type `WRITE` literally. Nothing on the disk is touched
     before that.
   - Write the result only if the partition is still, byte for byte, your original backup, and
     verify afterwards that the final content is correct.
3. If everything goes well, reboot the projector when the script asks. Wait 30–40 seconds and run:
   ```
   adb install -r Magisk-vXX.X.apk
   adb shell su -c id
   ```
   Accept the Magisk superuser prompt on screen. If it worked, `id` will show `uid=0(root)`.

### If something goes wrong

- If the script says `READBACK MISMATCH`, **do not reboot the projector**. Run
  `restore_stock.bat` in the same folder: it uses your saved `init_boot_a` copy to put it back.
- If the projector fails to boot, the H726 has no backup B slot, so the only way back is **FEL
  mode + PhoenixSuit** with your exact original firmware. That's why the full-storage backup in
  step 3 matters — keep it in two places.
- Keep `init_boot_a_stock.img` and its hash file around separately; `restore_stock.bat` needs them.

## After getting root

- Don't connect the projector to your main Wi-Fi network: these ROMs often ship with factory
  malware (silent installer, telemetry, a hidden `su` daemon). Use an isolated guest network, at
  least until you've cleaned it up.
- Don't accept system (OTA) updates: they would overwrite `init_boot` and you'd lose root.
- Only install Magisk modules from sources you trust.

## Disclaimer

This writes directly to the device's internal storage. The scripts include checks meant to avoid
writing without a verified backup, but you are responsible for following the steps in order and
not skipping the backup. Neither this guide nor whoever wrote it is responsible for a bricked
device.
