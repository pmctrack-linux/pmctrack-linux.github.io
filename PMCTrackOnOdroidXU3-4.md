---
layout: page
title: PMCTrack on Odroid-XU3/4
permalink: /install/pmctrack-on-odroid-xu3-4/
toc: true
---


<!--
**Permalink:**
http://pmctrack.dacya.ucm.es/install/pmctrack-on-odroid-xu3-4/
-->


# PMCTrack on Odroid-XU3/4

This article explains how to install PMCTrack on an Odroid-XU3 (or XU4) running Ubuntu. Note that the PMCTrack tool requires a patched Linux kernel to function. The kernel may be compiled on the board itself or on a Linux development host (e.g., x86 machine). In this article, we illustrate how to build the kernel on the Odroid-XU3 board. Nevertheless, a tutorial on how to build and install the kernel on a different machine with a cross-compiler can be found [here][1]. In addition, you can find the instructions on how to install Ubuntu on an Odroid-XU3 (or XU4) [here][2].

The complete process to install PMCTrack on the Odroid-XU3 board consists of the following steps:

1.  [Obtain the necessary files to build both PMCTrack and the Linux kernel on the board][3]
2.  [Patch, configure, and build the kernel][4] 
3.  [Install the kernel on the board][5] 
4.  [Boot PMCTrack's kernel][6] 
5.  [Build PMCTrack from source][7] 
6.  [Load PMCTrack's kernel module][8] 

<a name="get-info"></a>

### Obtain the necessary files to build PMCTrack and the Linux kernel

Get PMCtrack source code from github:

    $ git clone https://github.com/jcsaezal/pmctrack.git


Get the Linux kernel code (v3.10.y) for Odroid-XU3 (also valid for Odroid-XU4):

    $ git clone --depth 1 https://github.com/hardkernel/linux.git -b odroidxu3-3.10.y odroidxu3-3.10.y


<a name="patch-build"></a>

### Patch, configure, and build the kernel

First of all, switch to the root directory of the kernel source tree:

    $ cd odroidxu3-3.10.y


Assuming that PMCtrack source code can be found under ~/pmctrack, PMCTrack kernel patch can be applied as follows:

    $ patch -p1 < ~/pmctrack/src/kernel-patches/pmctrack_linux-odroidxu3-3.10.y_arm.patch
    patching file arch/arm/boot/dts/exynos5422_evt0.dtsi
    patching file arch/arm/kernel/perf_event_cpu.c
    patching file include/linux/pmctrack.h 
    patching file include/linux/sched.h
    ...


Configure the kernel and make sure that CONFIG_PMCTRACK is enabled:

    $ make odroidxu3_defconfig
      HOSTCC  scripts/basic/fixdep
      HOSTCC  scripts/kconfig/conf.o
      SHIPPED scripts/kconfig/zconf.tab.c
      SHIPPED scripts/kconfig/zconf.lex.c
      SHIPPED scripts/kconfig/zconf.hash.c
      HOSTCC  scripts/kconfig/zconf.tab.o
      HOSTLD  scripts/kconfig/conf
    #
    # configuration written to .config
    #
    
    $ grep PMCTRACK .config
    CONFIG_PMCTRACK=y


Build the kernel: (This will take some time. Be patient!!)

    $ make prepare modules_prepare
    $ make -j9 bzImage modules dtbs


<a name="install-kernel"></a>

### Install the kernel on the board

All the commands shown in this section must be executed from the root directory of the kernel source tree (where we ran the aforementioned commands for patching, configuring and building the kernel).

Make sure that the boot partition is mounted on /media/boot. If is not mounted, run the following command:

    $ sudo mount /media/boot 


Copy dtb file to boot partition (as board-pmc.dtb):

    $ sudo cp arch/arm/boot/dts/exynos5422-odroidxu3.dtb /media/boot/board-pmc.dtb


Copy kernel image to boot partition (as zImage-pmc):

    $ sudo cp arch/arm/boot/zImage  /media/boot/zImage-pmc


Install kernel modules under /lib/modules/<kernel_release>:

    $ sudo make modules_install
    
    # Make sure the modules were installed correctly (directory was created successfuly)
    $ file /lib/modules/`make kernelrelease`
    /lib/modules/3.10.104+: directory


Create initramfs:

    $ sudo update-initramfs -c -k `make kernelrelease`


Create uInitrd:

    $ sudo mkimage -A arm -O linux -T ramdisk -C none -a 0 -e 0 -n uInitrd -d /boot/initrd.img-`make kernelrelease` /boot/uInitrd-`make kernelrelease`


Install new uInitrd in boot partition (as uInitrd-pmc):

    $ sudo cp /boot/uInitrd-`make kernelrelease` /media/boot/uInitrd-pmc


<a name="boot-kernel"></a>

### Boot PMCTrack's kernel

Two alternatives exist to boot PMCtrack's kernel on the board. The first approach entails disabling autoboot mode in uboot and typing a set of uboot commands manually upon reboot. This alternative constitutes a very flexible option since the kernel that comes by default with Ubuntu's installation can be easily selected to boot anytime later without changing the uboot configuration. The second approach comes down to setting the PMCTrack kernel as the default one by changing uboot's configuration.

#### (1st approach) Disable autoboot and boot PMCTrack's kernel manually

Edit the /media/boot/boot.ini file (with sudo or as root) and comment out the very last line of the file, as follows:

    ...
    # Boot the board
    #boot


Reboot the board:

    $ sudo reboot


When the uboot's prompt is displayed on the serial port, type the following commands:

    > setenv bootcmd "fatload mmc 0:1 0x40008000 zImage-pmc; fatload mmc 0:1 0x42000000 uInitrd-pmc; fatload mmc 0:1 0x44000000 board-pmc.dtb; bootz 0x40008000 0x42000000 0x44000000"
    > boot


Note that by typing `boot` only, the default kernel will boot.

#### (2nd approach) Set the PMCTrack's kernel as the default kernel

Edit the /media/boot/boot.ini file (with sudo or as root) and comment out the following line:

    #setenv bootcmd "fatload mmc 0:1 0x40008000 zImage; fatload mmc 0:1 0x42000000 uInitrd; fatload mmc 0:1 0x44000000 exynos5422-odroidxu3.dtb; bootz 0x40008000 0x42000000 0x44000000"


Add the following bootcmd line to point to new zImage, uInitrd, and dtb files after the line you just commented out:

    setenv bootcmd "fatload mmc 0:1 0x40008000 zImage-pmc; fatload mmc 0:1 0x42000000 uInitrd-pmc; fatload mmc 0:1 0x44000000 board-pmc.dtb; bootz 0x40008000 0x42000000 0x44000000"


Save the changes and reboot the board. The PMCTrack kernel will boot by default.

    $ sudo reboot


<a name="build"></a>

### Build PMCTrack from source for the Odroid-XU3 board

Once the patched kernel is up and running, PMCTrack can be built from source by following the instructions shown [here][9]. Assuming that PMCtrack source code is found under ~/pmctrack, this comes down to running the following three commands:

    $ cd ~/pmctrack
    $ . shrc
    $ pmctrack-manager build
    **** System information ***
    Processor_vendor=ARM
    Kernel_HZ=200
    Processor_bitwidth=32
    ***********************************************
    Press ENTER to start the build process...
    
    *************************************************
    *** Building supported PMCTrack kernel modules **
    *************************************************
    Building kernel module arm....
    ============================================
    make -C /lib/modules/3.10.104+/build M=/home/odroid/pmctrack/src/modules/pmcs/arm modules
    make[1]: Entering directory '/scratch/odroid/Kernel/Build/odroidxu3-3.10.y'
      CC [M]  /home/odroid/pmctrack/src/modules/pmcs/arm/mchw_core.o
      CC [M]  /home/odroid/pmctrack/src/modules/pmcs/arm/mc_experiments.o
      CC [M]  /home/odroid/pmctrack/src/modules/pmcs/arm/pmu_config_arm.o
      CC [M]  /home/odroid/pmctrack/src/modules/pmcs/arm/cbuffer.o
      CC [M]  /home/odroid/pmctrack/src/modules/pmcs/arm/monitoring_mod.o
      CC [M]  /home/odroid/pmctrack/src/modules/pmcs/arm/syswide.o
      CC [M]  /home/odroid/pmctrack/src/modules/pmcs/arm/ipc_sampling_sf_mm.o
      CC [M]  /home/odroid/pmctrack/src/modules/pmcs/arm/vexpress_sensors_core.o
    /home/odroid/pmctrack/src/modules/pmcs/arm/vexpress_sensors_core.c: In function 'initialize_energy_sensors':
    /home/odroid/pmctrack/src/modules/pmcs/arm/vexpress_sensors_core.c:468:6: warning: unused variable 'retval' [-Wunused-variable]
      int retval=0;
          ^
      CC [M]  /home/odroid/pmctrack/src/modules/pmcs/arm/vexpress_sensors_mm.o
      LD [M]  /home/odroid/pmctrack/src/modules/pmcs/arm/mchw_arm.o
      Building modules, stage 2.
      MODPOST 1 modules
      CC      /home/odroid/pmctrack/src/modules/pmcs/arm/mchw_arm.mod.o
      LD [M]  /home/odroid/pmctrack/src/modules/pmcs/arm/mchw_arm.ko
    make[1]: Leaving directory '/scratch/odroid/Kernel/Build/odroidxu3-3.10.y'
    Done!!
    ============================================
    Building kernel module odroid-xu....
    ============================================
    make -C /lib/modules/3.10.104+/build M=/home/odroid/pmctrack/src/modules/pmcs/odroid-xu modules
    make[1]: Entering directory '/scratch/odroid/Kernel/Build/odroidxu3-3.10.y'
      CC [M]  /home/odroid/pmctrack/src/modules/pmcs/odroid-xu/mchw_core.o
      CC [M]  /home/odroid/pmctrack/src/modules/pmcs/odroid-xu/mc_experiments.o
      CC [M]  /home/odroid/pmctrack/src/modules/pmcs/odroid-xu/pmu_config_arm.o
      CC [M]  /home/odroid/pmctrack/src/modules/pmcs/odroid-xu/cbuffer.o
      CC [M]  /home/odroid/pmctrack/src/modules/pmcs/odroid-xu/monitoring_mod.o
      CC [M]  /home/odroid/pmctrack/src/modules/pmcs/odroid-xu/syswide.o
      CC [M]  /home/odroid/pmctrack/src/modules/pmcs/odroid-xu/ipc_sampling_sf_mm.o
      CC [M]  /home/odroid/pmctrack/src/modules/pmcs/odroid-xu/smart_power_driver.o
      CC [M]  /home/odroid/pmctrack/src/modules/pmcs/odroid-xu/smart_power_mm.o
      LD [M]  /home/odroid/pmctrack/src/modules/pmcs/odroid-xu/mchw_odroid_xu.o
      Building modules, stage 2.
      MODPOST 1 modules
      CC      /home/odroid/pmctrack/src/modules/pmcs/odroid-xu/mchw_odroid_xu.mod.o
      LD [M]  /home/odroid/pmctrack/src/modules/pmcs/odroid-xu/mchw_odroid_xu.ko
    make[1]: Leaving directory '/scratch/odroid/Kernel/Build/odroidxu3-3.10.y'
    Done!!
    ============================================
    Building libpmctrack ....
    ============================================
    make -C src all
    make[1]: Entering directory '/home/odroid/pmctrack/src/lib/libpmctrack/src'
    gcc -c  -Wall -g -fpic -I ../include -I ../../../modules/pmcs/include/pmc -o core.o core.c
    gcc -c  -Wall -g -fpic -I ../include -I ../../../modules/pmcs/include/pmc -o pmu_info.o pmu_info.c
    gcc -shared  -o ../libpmctrack.so core.o pmu_info.o
    ar rcs ../libpmctrack.a core.o pmu_info.o
    make[1]: Leaving directory '/home/odroid/pmctrack/src/lib/libpmctrack/src'
    Done!!
    ============================================
    Building pmc-events ....
    ============================================
    gcc  -DUSE_VFORK -Wall -g -I ../../modules/pmcs/include/pmc -I../../lib/libpmctrack/include   -c -o pmc-events.o pmc-events.c
    gcc -o ../../../bin/pmc-events pmc-events.o  -L../../lib/libpmctrack -lpmctrack -static
    Done!!
    ============================================
    Building pmctrack ....
    ============================================
    gcc  -DUSE_VFORK -Wall -g -I ../../modules/pmcs/include/pmc -I../../lib/libpmctrack/include   -c -o pmctrack.o pmctrack.c
    gcc -o ../../../bin/pmctrack pmctrack.o  -L../../lib/libpmctrack -lpmctrack -static
    Done!!
    ============================================
    *** BUILD PROCESS COMPLETED SUCCESSFULLY ***    


<a name="load-module"></a>

### Load PMCTrack's kernel module

Load the specific kernel module with support for the Odroid-XU3 and Odroid-XU4 boards:

    $ cd ~/pmctrack
    $ . shrc
    $ sudo insmod ${PMCTRACK_ROOT}/src/modules/pmcs/odroid-xu/mchw_odroid_xu.ko


Make sure that the PMUs have been detected correctly:

    $ pmc-events -I
    [PMU 0]
    pmu_model=armv7.cortex_a7
    nr_fixed_pmcs=1
    nr_gp_pmcs=4
    [PMU 1]
    pmu_model=armv7.cortex_a15
    nr_fixed_pmcs=1
    nr_gp_pmcs=6


If the output of the `pmc-events` command matches the information shown above, then PMCTrack is working correctly!!. For more information on how to use PMCTrack, please visit the ["Getting Started" page][10].

[1]: http://odroid.com/dokuwiki/doku.php?id=en:xu3_building_kernel
[2]: http://odroid.com/dokuwiki/doku.php?id=en:xu3_ubuntu_release_note_20160114
[3]: #get-info
[4]: #patch-build
[5]: #install-kernel
[6]: #boot-kernel
[7]: #build
[8]: #load-module
[9]: https://pmctrack.dacya.ucm.es/install/
[10]: https://pmctrack.dacya.ucm.es/getting-started/