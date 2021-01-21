# x210

Collection of patches and mods for the 51nb x210. 

## Table of Contents

* [Repo contents](#repo-contents)
* [How to patch the X210 BIOS](#how-to-patch-the-x210-bios)
  + [Option 1: use a patched embedded controller](#option-1-use-a-patched-embedded-controller)
  + [Option 2: flash an already patched BIOS image](#option-2-flash-an-already-patched-bios-image)
  + [Fix battery capacity detection](#fix-battery-capacity-detection)
* [Coreboot](#coreboot)
  + [Coreboot build HOWTO](#coreboot-build-howto)


## Repo contents
* blobs
   + **ec.bin** Embedded controller modified by Thinkpad forum user vladisslav2011 to increase brightness levels and improve battery voltage detection. Source: https://forum.thinkpads.com/viewtopic.php?p=833699#p833699
   + **me.bin** INTEL Management Engine blob, updated to the latest version with all known vulnerabilities patched (check with this tool: https://downloadcenter.intel.com/download/28632/Intel-Converged-Security-and-Management-Engine-Version-Detection-Tool-Intel-CSMEVDT-)
   + descriptor.bin, vbt.bin, vgabios.bin: other blobs needed for coreboot
* coreboot_images - precompiled, flashable coreboot images containing the patched EC, CPU microcode updates, updated ME, battery capacity detection fixes and power management improvements
   + **coreboot.rom** Coreboot 4.13 w/Intel ME and libgfxinit 
* colour_profiles
   + **X210-122.icm** ICM colour profile for the 12.2" 1920x1200 screen. 
   + **LP130QP1_SPA1.icm** ICM colour profile for the 13" 3000x2000 screen.
* kernel_patches
   + **x210-battery-fix.patch** Detect correct battery capacity and discharge rate (only needed on stock BIOS)
   + **r8169-enable-aspm.patch** Enable L1 ASPM and substates on the realtek ethernet NIC. As of Linux 5.10 there seem to be no issues enabling ASPM for the chip contained in the x210. Alternatively the [r8168](https://github.com/simonbcn/r8168-dkms) module from Realtek can be used. r8168 enables ASPM natively, however ASPM is often lost after resuming from sleep with this module.
   + **hda_intel-enable-power-gating.patch** /sys/kernel/debug/pmc_core/pch_ip_power_gating_status shows PCH IP: 9  - HDA-PGD0 State: Off with this patch. This is a precondition for s0ix (which does not work yet)
* **.config**  Coreboot .config file
* **bios-ec-mod.bin** Full bios dump including the modified EC. 
* **layout** Layout file required to flash the EC portion of the BIOS contributed by Thinkpad forum user L29Ah https://forum.thinkpads.com/viewtopic.php?p=834229#p834229

## How to patch the X210 BIOS

### Option 1: use a patched embedded controller

 You can replace the Embedded Controller with Vladislav's version: 

```
flashrom -p internal -r x210-current-internal-flashrom.bin
dd if=x210-current-internal-flashrom.bin of=fw.bin bs=1024 count=2048
dd if=ec.bin of=fw.bin bs=1024 count=64 seek=2048
dd if=x210-current-internal-flashrom.bin of=fw.bin bs=1024 count=6080 seek=2112
flashrom -V -p internal -l layout -i ec -w fw.bin
```
Credit for this goes to Thinkpad forum user L29Ah https://forum.thinkpads.com/viewtopic.php?p=834033#p834033

#### Option 1a: cherry-pick EC patches

L29Ah built a handy tool to select which patches to apply to the EC: https://github.com/l29ah/x210-ec

### Option 2: flash an already patched BIOS image

Alternatively you might want to flash the already patched bios:

```flashrom -p internal -w bios-ec-mod.bin```

Be advised that bios-ec-mod.bin contains my customizations and might not boot on a different machine. In particular, the SATA controller is disabled, only UEFI boot is enabled and the processor is undervolted by 100mV. You can modify these settings upon first boot from the BIOS setup panel.
My configuration also enables several power saving features disabled by defatult.


###  Fix battery capacity detection

Once the BIOS is flashed, in order for a system running the stock UEFI bios to report the correct battery capacity and discharge rate, you'll need to patch the kernel with **x210-battery-fix.patch**. Most of the X210's EC is likely directly taken from the X201 EC, including the bugs. In particular, the Linux kernel sources hint at what has been causing the capacity detection issues:

> On Lenovo Thinkpad models from 2010 and 2011, the power unit switches between mWh and mAh depending on whether the system is running on battery or not. When mAh is the unit, most reported values are incorrect and need to be adjusted by 10000/design_voltage. Verified on x201, t410, 410s, and x220. Pre-2010 and 2012 models appear to always report in mWh and are thus unaffected (tested with t42, t61, t500, x200, x300, and x230).

The only difference is that the power unit never changes on the X210, it's always mAh, and the values are always wrong. The patch fixes that by simply removing the checks on machine type and model and enabling the workaround on the X210

## Coreboot

Matthew Garrett has been porting coreboot to the X210: https://forum.thinkpads.com/viewtopic.php?f=80&t=126731 As of 20 March 2020 the X210 is part of the official coreboot tree. Pre-compiled coreboot images for the x210 (3rd batch) are provided here.

Flash coreboot with the following command

```flashrom -p internal -w coreboot.rom```

The build provided includes the patched EC, fixes battery capacity detection problems so that no kernel patching is required, and enables several power saving features such as SATA Aggressive PM, Devslp, ASPM L1 substates for all PCIe devices including NVMe. It also includes CPU microcode updates from Intel. 

The payload is tianocore, therefore it will boot a UEFI operating system, not a BIOS one. It'll also boot the installer or an existing installation of Windows 10.

This build includes the full 3rd gen Management Engine. It may or may not work on other generations of the X210. Coreboot gives the ability to strip down the ME at build time using me_cleaner. If you want to neuter the ME completely you should build your own image following the steps below.

### Coreboot build HOWTO

If you want a bleeding edge coreboot for your X210 do the following.

Download the latest coreboot tree:
```
git clone https://review.coreboot.org/coreboot
cd coreboot
git submodule update --init --checkout
```

Copy the .config file from the repo into the coreboot dir.

Extract the the stock bios: ```flashrom -p internal -r x210-stock-bios.rom```

Open the stock bios in UEFITool https://github.com/LongSoft/UEFITool and extract Descriptor region and ME Region to descriptor.bin and me.bin ("Extract as is"). Then search for ```VGA Compatible BIOS``` (unselect UNICODE), double click on the string in "Messages" which will take you to the relevant section, then Action -> Section -> Extract Body. Save it as vgabios.bin

Copy the VBT: ```cp /sys/kernel/debug/dri/0/i915_vbt vbt.bin```

Put all the .bin files (including ec.bin from the repo) in coreboot/3rdparty/blobs/mainboard/51nb/x210. If you plan to use libgfxinit you can safely omit vgabios.bin and vbt.bin (although linux will complain about not finding the VBT). Then build the crosstools. ```make crosstools-i386 CPUS=8```

Run ```make menuconfig``` in the main coreboot directory. The .config provided here should work well for all users. You have the option of neutering the ME in the "Chipset" menu. In System tables you can add the serial number of your machine to the SMBIOS tables. The SN is usually found on a handwritten sticker attached to one of the RAM slots.

Compile coreboot: ```make -j8```

Pay attention to any error/issue that might arise. If all goes well the image will be generated under build/ and it won't brick your machine.
```flashrom -p internal -w coreboot.rom```

You can keep your coreboot tree up to date by issuing ```git pull``` in the root directory, if you want to recompile later.
