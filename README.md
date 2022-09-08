# TP1803 Windows Guide

## Warning

**⚠️ FIRST, READ THE WHOLE GUIDE BEFORE PROCEEDING ⚠️**

**THIS WILL MODIFY PARTITIONING SCHEME AND ERASE YOUR DATA AND THE ORIGINAL FIRMWARE.**

**BACKUP YOUR IMPORTANT DATA AND ANY DEVICE SPECIFIC PARTITIONS (NOTABLY THE PARTITIONS IN LUN5) BEFORE PROCEEDING.**

This software is highly experimental and we are not responsible for any eventual damage to your device. Please proceed with caution.

Original firmware dump link: https://drive.google.com/drive/folders/1PZcb8x5lUJpL8zSRouZqt-PnngTKiHNy

If you want to go back to the original firmware, you can use the dump provided above. Simply reboot to EDL and flash every partition.

**If you decide to reflash the firmware dump, avoid flashing LUN5 or you will lose the IMEI and other device specific data.**

## Requirements

- A Windows PC running Windows 10 2004 or newer
- TWRP: https://t.me/TP1803_repo
- UEFI firmware from [releases](https://github.com/alula/TP1803-Windows-Guide/releases) or compiled from source: https://github.com/edk2-porting/MU-sm8150Pkg
- TP1803 specific driver package: https://github.com/alula/TP1803-Drivers/releases
- DriverUpdater, to install the driver set: https://github.com/WOA-Project/DriverUpdater/releases/
- Files for this guide from the repo: https://github.com/alula/TP1803-Windows-Guide/archive/master.zip
- ADB/Fastboot tools: https://developer.android.com/studio/releases/platform-tools
- An ARM64 Windows 10/11 ISO of your choice

  You can create one using https://uupdump.net/ or [UUPMediaCreator](https://github.com/gus33000/UUPMediaCreator) ([here's a guide](https://github.com/WOA-Project/SurfaceDuo-Guides/blob/main/CreateWindowsISO.md))

## Partitioning

- Flash TWRP and reboot into it
- Repartition your device:

  - Option 1 - Completely remove Android and use entire LUN0 for Windows

    Push the necessary files to your device:

    ```bash
    # Push the GPT dump to the device
    adb push <path to guide files>/windows64G.gpt /tmp/

    # Shell into the device
    adb shell
    ```

    Apply the new partitioning scheme to LUN0:

    ```bash
    # THIS NUKES THE PARTITION TABLE AND POSSIBLY YOUR DATA
    # MAKE SURE YOU HAVE A BACKUP OF YOUR DATA BEFORE PROCEEDING
    sgdisk --restore /tmp/windows64G.gpt /dev/block/sda

    # Reboot so the kernel can pick up the new partition table
    reboot recovery
    ```

    Shell into the device again:

    ```bash
    adb shell
    ```

    Verify that the partitioning scheme is correct:

    ```bash
    sgdisk --print /dev/block/sda

    # The output of command above should look like this:
    #
    # Disk /dev/block/sda: 14501888 sectors, 55.3 GiB
    # Logical sector size: 4096 bytes
    # Disk identifier (GUID): xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
    # Partition table holds up to 128 entries
    # First usable sector is 6, last usable sector is 14501882
    # Partitions will be aligned on 256-sector boundaries
    # Total free space is 250 sectors (1000.0 KiB)
    #
    # Number  Start (sector)    End (sector)  Size       Code  Name
    #    1             256          131327   512.0 MiB   EF00  EFIESP
    #    2          131328          135423   16.0 MiB    0C01  MSReserved
    #    3          135424        14501882   54.8 GiB    0700  MainOS
    ```

    Format the partitions:

    ```bash
    mkfs.fat /dev/block/by-name/EFIESP
    mkfs.ntfs -f /dev/block/by-name/MSReserved
    mkfs.ntfs -f /dev/block/by-name/MainOS
    ```

  - Option 2 - Shrink userdata and use the rest for Windows

    **TODO**

## Install Windows

Exit the ADB shell if you're still in it.

Put the device in mass storage mode:

```bash
adb push <path to guide files>/msc.sh /tmp/
adb shell "sh /tmp/msc.sh"
```

Mount the MainOS and EFIESP partitions on your PC:

- Make sure you are in Mass Storage Mode, that your TP1803 is plugged into your PC
- Mount the partitions you have created using diskpart and assign them some letters:

```
⚠️ THESE ARE NOT ALL COMMANDS. DISKPART COMMANDS VARY A LOT, SO THESE ARE SOME ROUGH INSTRUCTIONS. 
ACTUAL COMMANDS START WITH AN HASHTAG (which you'll need to remove)

# list disk
Find the TP1803 Disk, and take note of the number.
# select disk <number>
# list partition
You'll be able to recognize the partitions we made earlier by their size. take note of the ESP and WIN partition numbers.
# select partition <esp-partition-number>
# assign letter=Y:
# select partition <win-partition-number>
# assign letter=X:
```

You'll have two partitions loaded, one is the ESP partition, and the other is the Windows partition. Take note of the letters you've used.

**⚠️ WARNING: From now on we'll assume X: is the Win partition and that Y: is the ESP partition for all the commands. Replace them correctly or you'll lose data on your PC.**

We'll need our install.wim file now. If you haven't it already, you can [use this guide](https://github.com/WOA-Project/SurfaceDuo-Guides/blob/main/CreateWindowsISO.md). When you're ready, run these commands:

```
dism /apply-image /ImageFile:"<path to install.wim>" /index:1 /ApplyDir:X:\
```

This will take a bit of time. Go make some coffee ☕ or some tea 🍵.

Once that's done:

```
bcdboot X:\Windows /s Y: /f UEFI
```

Windows is now installed but has no drivers.

## Installing the drivers

Extract the drivers and run the following command to install them:

```
DriverUpdater.exe -d "<path to extracted drivers>\definitions\Desktop\ARM64\Internal\tp1803.txt" -r "<path to extracted drivers>" -p X:\
```

Now we want to disable driver signature checks (otherwise Windows will throw a BSOD at boot):

```
bcdedit /store Y:\EFI\Microsoft\BOOT\BCD /set {default} testsigning on
```

## Installing the UEFI firmware

At this point Windows is installed and has the drivers, but in order to boot it we need to flash the UEFI firmware.

Put the device in fastboot mode:

```bash
adb reboot bootloader
```

Flash the firmware:

```bash
fastboot flash boot <path to downloaded uefi.img>
```

Reboot the device. 

```bash
fastboot reboot
```

Congratulations! If you've done everything correctly, you should be able to boot and use Windows now.

## Credits

This guide has been partially based on https://github.com/WOA-Project/SurfaceDuo-Guides

Technically you can follow it for most part as well.
