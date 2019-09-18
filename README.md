# x210

Collection of patches and mods for the 51nb x210. 

## Table of Contents

* [Repo contents](#repo-contents)
* [How to patch the X210 BIOS](#how-to-patch-the-x210-bios)
  + [Option 1: patch the embedded controller](#option-1-patch-the-embedded-controller)
  + [Option 2: flash an already patched BIOS image](#option-2-flash-an-already-patched-bios-image)
  + [Fix battery capacity detection](#fix-battery-capacity-detection)
* [Coreboot](#coreboot)
  + [Coreboot build HOWTO](#coreboot-build-howto)


## Repo contents
 * **ec.bin** Embedded controller modified by Thinkpad forum user vladisslav2011 to increase brightness levels and improve battery voltage detection. Source: https://forum.thinkpads.com/viewtopic.php?p=833699#p833699
 * **bios-ec-mod.bin** Full bios dump including the modified EC. 
  * **layout** Layout file required to flash the EC portion of the BIOS contributed by Thinkpad forum user L29Ah https://forum.thinkpads.com/viewtopic.php?p=834229#p834229
 * **x210-battery-fix.patch** Linux kernel patch to detect correct battery capacity and discharge rate on the stock BIOS.
 * **coreboot.rom** Coreboot image containing the patched EC, CPU microcode updates, battery capacity detection fixes and power management improvements.
 * **coreboot-gfxinit.rom** Same as above implementing libgfxinit instead of GOP driver
 * **.config**  Coreboot .config file
 * **X210-122.icm** ICM colour profile for the 12.2" 1920x1200 screen. 

## How to patch the X210 BIOS
 
 ### Option 1: patch the embedded controller
 
 You can patch the Embedded Controller with Vladislav's version: 

```
flashrom -p internal -r x210-current-internal-flashrom.bin
dd if=x210-current-internal-flashrom.bin of=fw.bin bs=1024 count=2048
dd if=ec.bin of=fw.bin bs=1024 count=64 seek=2048
dd if=x210-current-internal-flashrom.bin of=fw.bin bs=1024 count=6080 seek=2112
flashrom -V -p internal -l layout -i ec -w fw.bin
```
Credit for this goes to Thinkpad forum user L29Ah https://forum.thinkpads.com/viewtopic.php?p=834033#p834033

### Option 2: flash an already patched BIOS image

Alternatively you might want to flash the already patched bios:

```flashrom -p internal -w bios-ec-mod.bin```

Be advised that bios-ec-mod.bin contains my customizations and might not boot on a different machine. In particular, the SATA controller is disabled, only UEFI boot is enabled and the processor is undervolted by 100mV. You can modify these settings upon first boot from the BIOS setup panel.
My configuration also enables several power saving features disabled by defatult.


###  Fix battery capacity detection

Once the BIOS is flashed, in order for a system running the stock UEFI bios to report the correct battery capacity and discharge rate, you'll need to patch the kernel with **x210-battery-fix.patch**. Most of the X210's EC is likely directly taken from the X201 EC, including the bugs. In particular, the Linux kernel sources hint at what has been causing the capacity detection issues:

> On Lenovo Thinkpad models from 2010 and 2011, the power unit switches between mWh and mAh depending on whether the system is running on battery or not. When mAh is the unit, most reported values are incorrect and need to be adjusted by 10000/design_voltage. Verified on x201, t410, 410s, and x220. Pre-2010 and 2012 models appear to always report in mWh and are thus unaffected (tested with t42, t61, t500, x200, x300, and x230).

The patch fixes that by simply removing the checks on machine type and model and enabling the workaround on the X210

## Coreboot

Matthew Garrett has been porting coreboot to the X210: https://forum.thinkpads.com/viewtopic.php?f=80&t=126731 It is still not part of the official coreboot tree, but it may be at some point. So far everything seems to work very well, except for the SD card reader. A compiled coreboot image for the x210 (3rd batch) is provided here.

Flash coreboot with the following command

```flashrom -p internal -w coreboot.rom``` or, if you are already running coreboot, ```flashrom -p internal:laptop=force_I_want_a_brick -w coreboot.bin```

The build provided includes the patched EC, fixes battery capacity detection problems so that no kernel patching is required, and enables several power saving features such as SATA Aggressive PM, Devslp, ASPM L1 substates for all PCIe devices including NVMe. It also includes CPU microcode updates from Intel. 

The payload is tianocore, therefore it will boot a UEFI operating system, not a BIOS one. It'll also boot Windows 10, however the installer doesn't seem to detect the NVME drive, I am unsure about SATA drives.

This build includes the full 3rd gen Management Engine. It may or may not work on other generations of the X210. Coreboot gives the ability to strip down the ME at build time using me_cleaner. If you want to neuter the ME completely you can build your own image with the up-to-date tree here https://github.com/harrykipper/coreboot

### Coreboot build HOWTO

If you want a bleeding edge coreboot for your X210 do the following.

Download the latest coreboot tree:
```
git clone https://review.coreboot.org/coreboot
cd coreboot
git submodule update --init --checkout
```

*Optional:* cherrypick smmstore patches to enable saving and loading Tianocore settings
```
git fetch "https://review.coreboot.org/coreboot" refs/changes/12/30012/16 && git cherry-pick FETCH_HEAD
git fetch "https://review.coreboot.org/coreboot" refs/changes/32/30432/9 && git cherry-pick FETCH_HEAD
```

 Download the x210_test branch from my repo: 
 ```
 cd ..
 wget https://github.com/harrykipper/coreboot/archive/x210_test.zip
 unzip x210_test.zip
 ```
 
 Copy the X210 stuff over from x210_test into the main coreboot dir
 ``` 
 cp -r coreboot-x210_test/src/mainboard/51nb coreboot/src/mainboard/
 cp -r coreboot-x210_test/src/ec/51nb coreboot/src/ec/
 ```
Copy the .config file from the repo into the coreboot dir.

Extract the the stock bios: ```flashrom -p internal -r x210-stock-bios.rom```

Open the stock bios in UEFITool https://github.com/LongSoft/UEFITool and extract Descriptor region and ME Region to descriptor.bin and me.bin ("Extract as is"). Then search for ```VGA Compatible BIOS``` (unselect UNICODE), double click on the string in "Messages" which will take you to the relevant section, then Action -> Section -> Extract Body. Save it as vgabios.bin

Copy the VBT: ```cp /sys/kernel/debug/dri/0/i915_vbt vbt.bin```

Put all the .bin files (including ec.bin from the repo) in coreboot/3rdparty/blobs/mainboard/51nb/. If you plan to use libgfxinit you can safely omit vgabios.bin and vbt.bin (although linux will complain that it can't find a vbt). Then build the crosstools. ```make crosstools-i386 CPUS=8```

Run ```make menuconfig``` in the main coreboot directory. The default configuration should work well for all users. You have the option of neutering the ME in the "Chipset" menu. In System tables you can add the serial number of your machine to the SMBIOS tables. The SN is usually found on a handwritten sticker attached to one of the RAM slots.

Compile coreboot: ```make -j8```

Pay attention to any error/issue that might arise. If all goes well the image will be generated under build/ and it won't brick your machine.
```flashrom -p internal -w coreboot.rom```

You can keep your coreboot tree up to date by issuing ```git pull``` in the root directory, if you want to recompile later.
