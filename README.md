zybo 
=========

This repo contains the software and instructions necessary to run a RISC-V Rocket Core on the Digilent Zybo board. 
Setup 
--------------

### Method 1 \(Quick\):

Step 1: Copy everything from the `sd_image` directory to the root of your 
microsd card.

Jump to the "Booting Up and interacting with the RISC-V Rocket core" section 
to continue.

### Method 2 \(Build From Scratch\): 

You may also build the system from scratch. These instructions have been tested with Vivado 2013.4 and Xilinx SDK 2013.4.

As a quick summary, here's what the Zybo will do when booting:

1) Impact the Rocket Core onto the zynq's programmable logic and configure the
on-board ARM core. These come from the `system_wrapper.bit` file, which will be 
part of `boot.bin` on the sd card.

2) Run the First Stage BootLoader (FSBL), which is part of BOOT.bin which we will eventually create.

3) The FSBL will start u-Boot, which sets up the environment for linux on the ARM core.

4) Finally, u-Boot will start our linux image (uImage) on the ARM core. 
This requires a custom devicetree.dtb file, which we will compile shortly.

We'll now go ahead and build each of these components.

####Step 0:

Make sure you've sourced the settings for Vivado as necessary on your system.

####Step 1: Building the Vivado Project

```sh
$ git clone git@github.com:ucb-bar/zybo.git
cd zybo/hw
vivado -mode tcl -source zybo_refchip.tcl
cd zybo_bsd
vivado ./zybo_bsd.xpr
```

Once in Vivado, first hit "Generate Block Design" -> system.bd, then hit "Open Block Design" -> system.bd.

Then, run Generate Bitstream (it should step through Synthesis and Implementation automatically).
While this is running, you can go ahead and follow steps 3 and 4 to build u-boot and Linux (but not the FSBL).

Once bitstream generation in vivado finishes, make sure that "Open Implemented Design" is selected and hit "OK".
After the implemented design is open, we can export the project to the Xilinx SDK. 
To do so, hit File -> Export -> "Export Hardware for SDK...".

In the dialog that pops up, make sure that Source is selected as "system.bd". 
You can leave "Export to" and "Workspace" as their defaults, which should be 
`<Local to Project>`. The rest of the tutorial assumes that this is the case. 
Finally, make sure that "Export Hardware", "Include bitstream", and "Launch SDK" 
are all checked. Then click OK. Once the SDK launches, feel free to exit Vivado.

In case the SDK fails to launch, the command to do so is:

```sh
xsdk -bit [ZYBO REPO LOCATION]/hw/zybo_bsd/zybo_bsd.sdk/SDK/SDK_Export/hw/system_wrapper.bit -workspace [ZYBO REPO LOCATION]/hw/zybo_bsd/zybo_bsd.sdk/SDK/SDK_Export -hwspec [ZYBO_REPO_LOCATION]/hw/zybo_bsd/zybo_bsd.sdk/SDK/SDK_Export/hw/system.xml
```

####Step 2: Building the First-Stage BootLoader (FSBL) in Xilinx SDK

Now, we'll use the SDK to build the FSBL:

Inside the SDK, hit File -> New -> Project. In the window that pops up, expand 
the Xilinx node in the tree of options and click "Application Project".

In the following window, enter "FSBL" as the Project name and ensure that 
Hardware Platform under Target Hardware is set to `hw_platform_0` and that the 
Processor is set to `ps7_cortexa9_0`. Under Target Software, ensure that OS 
Platform is set to standalone, Language is set to C, and Board Support Package 
is set to Create New, with "FSBL_bsp" as the name.

After you hit next, you'll be presented with a menu with available templates. 
Select "Zynq FSBL" and click finish. Once you do so, you should see compilation 
in progress in the console. Wait until "Build Finished" appears. Do not exit 
Xilinx SDK yet, we'll come back to it in a couple of steps to package up 
`BOOT.bin`.

####Step 3: Build u-boot for the zybo:


First, you'll want to grab the u-boot source for the Zybo. Since things tend to 
shift around in Digilent's repos, we've made a fork to ensure a stable base for 
this guide. This copy of the repo also contains a modified copy of the default 
zybo u-boot configuration to support the RISC-V Rocket core. The repo is already 
present as a submodule, so you'll need to initialize it (this will also download 
the linux source for you, which you'll need in Step 5):

```sh
[From inside the zybo repo]
git submodule update --init
```

Next, we'll go ahead and build u-boot. To do so, enter the following:

```sh
cd u-boot-Digilent-Dev
make CROSS_COMPILE=arm-xilinx-linux-gnueabi- zynq_zybo_config
make CROSS_COMPILE=arm-xilinx-linux-gnueabi-
```

Once the build completes, the file we need is called "u-boot", but we need to 
rename it. Run the following to do so:

```sh
mv u-boot u-boot.elf
```

#### Step 4: Create BOOT.bin

At this point, we'll package up the 3 main files we've built so far: 
`system_wrapper.bit`, the FSBL binary, and the `u-boot.elf` binary. To do so, 
return to the Xilinx SDK and select "Create Zynq Boot Image" from the 
"Xilinx Tools" menu. 

In the window that appears, choose an output location for the `boot.bif` file 
we'll create. Do not check "Use Authentication" or "Use Encryption". Also set 
an output path for `boot.bin` at the bottom (it will likely be called 
`output.bin` by default). Now, in the Boot Image Partitions area, hit Add. 

In the window that pops up, ensure that Partition type is set as bootloader and 
click browse. The file we're looking for is called `FSBL.elf`. If you've followed
these instructions exactly, it will be located in:

```sh
[ZYBO REPO LOCATION]/hw/zybo_bsd/zybo_bsd.sdk/SDK/SDK_Export/FSBL/Debug/FSBL.elf
```

Once you've located the file, click OK.

Next we'll add the bit file containing our design (with the Rocket core). Click 
add once again. This time, Partition type should be set as datafile. Navigate 
to the following directory to choose your bit file:

```sh
[ZYBO REPO LOCATION]/hw/zybo_bsd/zybo_bsd.sdk/SDK/SDK_Export/hw_platform_0/system_wrapper.bit
```

Once you've located the file, click OK.

Finally, we'll add in u-boot. Again, click add and ensure that Partition type 
is set to datafile in the window that appears. Again. click browse and select 
`u-boot.elf` located in:

```sh
[ZYBO REPO LOCATION]/u-boot-Digilent-Dev/u-boot.elf
```

Once you've located the file, click OK and hit Create Image in the 
Create Zynq Boot Image window. Copy the generated boot.bin file to your sd card.

#### Step 5: Build Linux Kernel uImage

Next, we'll move on to building the Linux Kernel uImage. Again, to keep our 
dependencies in-order, we've created a fork of the Digilent linux repo and 
added a modified dts file to support Rocket's Host-Target Interface (HTIF). 
When you ran `git submodule update --init` back in step 3, the source for 
linux was pulled into `[ZYBO REPO LOCATION]/Linux-Digilent-Dev`. Enter that 
directory and do the following to build linux:

```sh
make ARCH=arm CROSS_COMPILE=arm-xilinx-linux-gnueabi- xilinx_zynq_defconfig
make ARCH=arm CROSS_COMPILE=arm-xilinx-linux-gnueabi-
```

This will produce a zImage file, however we need a uImage. In order to build a 
uImage, the Makefile needs to be able to run the `mkimage` program found in 
`u-boot-Digilent-Dev/tools`. Thus, we'll have to add it to our path:

```sh
export PATH=$PATH:[ZYBO REPO LOCATION]/u-boot-Digilent-Dev/tools
make ARCH=arm CROSS_COMPILE=arm-xilinx-linux-gnueabi- UIMAGE_LOADADDR=0x8000 uImage
```

This will produce the file
`[ZYBO REPO LOCATION]/Linux-Digilent-Dev/arch/arm/boot/uImage`. Copy this file 
to your sdcard.

Next, we'll compile our device tree, so that the kernel knows where our devices 
are. The copy of linux in this repo already contains a modified device tree to 
support the HTIF infrastructure that Rocket needs. To compile it, run the 
following from inside `[ZYBO REPO LOCATION]/Linux-Digilent-Dev/`.

```sh
./scripts/dtc/dtc -I dts -O dtb -o devicetree.dtb arch/arm/boot/dts/zynq-zybo.dts
```

This will produce a `devicetree.dtb` file, which you should copy to your sdcard.

Finally, copy the uramdisk.image.gz file from `[ZYBO REPO LOCATION]/sd_image/` 
to your sd card. This is the root filesystem for ARM linux, which contains a 
copy of riscv-fesvr (frontend server) that will interact with our Rocket core 
on the programmable logic. If you'd like to modify this ramdisk, see Appendix A
at the bottom of this document.

At this point, there should be 4 files on your sd card. Continue to the 
"Booting Up and interacting with the RISC-V Rocket core" section.

Booting Up and interacting with the RISC-V Rocket core
--------------

TODO: how to boot, run linux on rocket, telnet IP address etc.

Appendices
--------------

### Appendix A: Modifying the rootfs:

The RAMDisk that holds linux (uramdisk.image.gz) is a gzipped cpio archive 
with a u-boot header for the zybo. To open it up (will need sudo):

```sh
dd if=sd_image/uramdisk.image.gz  bs=64 skip=1 of=uramdisk.cpio.gz
mkdir ramdisk
gunzip -c uramdisk.cpio.gz | sudo sh -c 'cd ramdisk/ && cpio -i'
```

When changing or adding files, be sure to keep track of owners, groups, and 
permissions. When you are done, to package it back up:

```sh
sh -c 'cd ramdisk/ && sudo find . | sudo cpio -H newc -o' | gzip -9 > uramdisk.cpio.gz
mkimage -A arm -O linux -T ramdisk -d uramdisk.cpio.gz uramdisk.image.gz
```

