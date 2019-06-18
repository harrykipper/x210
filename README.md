# x210

Collection of patches and mods for the 51nb x210. 
 * **ec.bin** Embedded controller modified by Thinkpad forum user Vladislav to increase brightness levels and voltage detection. Source: https://forum.thinkpads.com/viewtopic.php?p=833699#p833699
 * **bios-ec-mod.bin** Full bios dump including the modified EC. 
  * **layout** Layout file required to flash the EC portion of the BIOS provided by Thinkpad forum user L29Ah https://forum.thinkpads.com/viewtopic.php?p=834229#p834229
 * **x210-battery-fix** Linux kernel patch to detect correct battery capacity.
 
 ## How to patch the X210 BIOS
 
 You can patch the Embedded Controller with Vladislav's version: 

```flashrom -p internal -r x210-current-internal-flashrom.bin
dd if=x210-current-internal-flashrom.bin of=fw.bin bs=1024 count=2048
dd if=ec.bin of=fw.bin bs=1024 count=64 seek=2048
dd if=x210-current-internal-flashrom.bin of=fw.bin bs=1024 count=6080 seek=2112
flashrom -V -p internal -l layout -i ec -w fw.bin
```
Credit for this goes to Thinkpad forum user L29Ah https://forum.thinkpads.com/viewtopic.php?p=834033#p834033

Alternatively you might want to flash the already patched bios:

```flashrom -p internal -w bios-ec-mod.bin```

Be advised that bios-ec-mod.bin contains my customizations and might not boot on a different machine. In particular, the SATA controller is disabled and only UEFI boot is enabled.

##  Battery capacity

Once the BIOS is flashed, in order for the system to report the correct battery capacity and power draw you'll need to patch the kernel with x210-battery-fix.patch. This is because most of the X210's EC is probably directly taken from the X201 EC, including the bugs.  In particular, the Linux kernel sources hint at what has been causing the capacity detection issues:

> On Lenovo Thinkpad models from 2010 and 2011, the power unit switches between mWh and mAh depending on whether the system is running on battery or not. When mAh is the unit, most reported values are incorrect and need to be adjusted by 10000/design_voltage. Verified on x201, t410, 410s, and x220. Pre-2010 and 2012 models appear to always report in mWh and are thus unaffected (tested with t42, t61, t500, x200, x300, and x230). Also, in mid-2012 Lenovo issued a BIOS update for the 2011 models that fixes the issue (tested on x220 with a post-1.29 BIOS), but as of Nov. 2012, no such update is available for the 2010 models.
