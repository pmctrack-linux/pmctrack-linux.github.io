---
layout: page
title: Install
permalink: /install
toc: true
---

<!--
**Permalink:**
http://pmctrack.dacya.ucm.es/install/
‎[Edit](https://pmctrack.dacya.ucm.es/wp-admin/post.php?post=11&action=edit#post_name)
-->

### System requirements

To support PMCTrack, a patched Linux kernel must be installed on the machine. A number of kernel patches for various Linux versions can be found in the [`src/kernel-patches`][1] directory of [PMCTrack's repository][2]. The name of each patch file encodes the Linux kernel version where the patch must be applied to as well as the processor architecture supported. The format is as follows:

    pmctrack_linux-<kernel_version>_<architecture>.patch 


To build the kernel for PMCTrack, the following option must be enabled when configuring the kernel:

    CONFIG_PMCTRACK=y


The kernel headers for the patched Linux version must be installed on the system as well. This is necessary for a successful out-of-tree build of PMCTrack's kernel module. An out-of-tree-ready Makefile can be found in the sources for the different flavors of the kernel module.

Most PMCTrack user-level components are written in C, and do not depend on any external library, (beyond the libc, of course). A separate Makefile is provided for *libpmctrack* as well as for the various command-line tools. As such, it should be straightforward to build these software components on most Linux distributions.

The PMCTrack-GUI application, a Python front-end for the `pmctrack` command-line tool, has been recently integrated into PMCTrack's main repository. This application extends the capabilities of the PMCTrack stack with features such as an SSH-based remote monitoring mode or the ability to plot the values of user-defined performance metrics in real time. This GUI application runs on Linux and Mac OS X and has the following software dependencies:

*   Python v2.7
*   Matplotlib (Python library)
*   sshpass (command)
*   WxPython v3.0

On Debian or Ubuntu the necessary software can be installed as follows:

    $ sudo apt-get install python2.7 python-matplotlib python-wxgtk3.0 sshpass 


On Mac OS X, PMCTrack-GUI has been succesfully tested after installing the above software dependencies using MacPorts as follows:

    ## Install packages
    $ sudo port install py27-matplotlib py27-numpy py27-scipy py27-ipython py27-wxpython-3.0 sshpass
    
    ## Set up default configuration for matplotlib 
    $ mkdir  ~/.matplotlib
    $ cp /opt/local/Library/Frameworks/Python.framework/Versions/2.7/lib/python2.7/site-packages/matplotlib/mpl-data/matplotlibrc  ~/.matplotlib
    
    ## Select MacPorts Python27 interpreter by default
    $sudo port select --set python python27
    $sudo port select --set ipython ipython27


On both Linux and Mac OS X, the `bin` directory, found in the root of PMCTrack's repository, must be in the PATH so that PMCTrack-GUI works correctly. In addition, for a successful operation of PMCTrack-GUI's SSH mode, PMCTrack must be installed on the target machine (being accessed via SSH) and the "right" value for the PATH variable should be set automatically when login in via SSH. To make this possible add the following lines to the `.bash_profile` or `.bashrc` files in the user's HOME directory of the target machine:

    cd $PMCTRACK_ROOT_DIR ; . shrc


### Building PMCTrack from source for ARM and x86 processors

The `PMCTRACK_ROOT` environment variable must be defined for a successful execution of the various PMCTrack command-line tools. The `shrc` script found in the repository's root directory can be used to set the `PMCTRACK_ROOT` variable appropriately as well as to add command-line tools' directories to the PATH. To make this possible, run the following command in the root directory of the repository:

    $ . shrc


Now kernel-level and user-level components can be easily built with the `pmctrack-manager` script as follows:

    $ pmctrack-manager build
    **** System information ***
    Processor_vendor=Intel
    Kernel_HZ=250
    Processor_bitwidth=64
    ***********************************************
    Press ENTER to start the build process...
    
    *************************************************
    *** Building supported PMCTrack kernel modules **
    *************************************************
    Building kernel module intel-core....
    ============================================
    make -C /lib/modules/3.17.3.pmctrack-x86+/build M=/home/bench/pmctrack/src/modules/pmcs/intel-core modules
    make[1]: Entering directory '/usr/src/linux-headers-3.17.3.pmctrack-x86+'
      CC [M]  /home/bench/pmctrack/src/modules/pmcs/intel-core/mchw_core.o
      CC [M]  /home/bench/pmctrack/src/modules/pmcs/intel-core/mc_experiments.o
      CC [M]  /home/bench/pmctrack/src/modules/pmcs/intel-core/pmu_config_x86.o
      CC [M]  /home/bench/pmctrack/src/modules/pmcs/intel-core/cbuffer.o
      CC [M]  /home/bench/pmctrack/src/modules/pmcs/intel-core/monitoring_mod.o
      CC [M]  /home/bench/pmctrack/src/modules/pmcs/intel-core/syswide.o
      CC [M]  /home/bench/pmctrack/src/modules/pmcs/intel-core/intel_cmt_mm.o
      CC [M]  /home/bench/pmctrack/src/modules/pmcs/intel-core/intel_rapl_mm.o
      CC [M]  /home/bench/pmctrack/src/modules/pmcs/intel-core/ipc_sampling_sf_mm.o
      LD [M]  /home/bench/pmctrack/src/modules/pmcs/intel-core/mchw_intel_core.o
      Building modules, stage 2.
      MODPOST 1 modules
      CC      /home/bench/pmctrack/src/modules/pmcs/intel-core/mchw_intel_core.mod.o
      LD [M]  /home/bench/pmctrack/src/modules/pmcs/intel-core/mchw_intel_core.ko
    make[1]: Leaving directory '/usr/src/linux-headers-3.17.3.pmctrack-x86+'
    Done!!
    ============================================
    Building kernel module core2....
    ============================================
    make -C /lib/modules/3.17.3.pmctrack-x86+/build M=/home/bench/pmctrack/src/modules/pmcs/core2 modules
    make[1]: Entering directory '/usr/src/linux-headers-3.17.3.pmctrack-x86+'
      CC [M]  /home/bench/pmctrack/src/modules/pmcs/core2/mchw_core.o
      CC [M]  /home/bench/pmctrack/src/modules/pmcs/core2/mc_experiments.o
      CC [M]  /home/bench/pmctrack/src/modules/pmcs/core2/pmu_config_x86.o
    /home/bench/pmctrack/src/modules/pmcs/core2/pmu_config_x86.c: In function 'init_pmu_props':
    /home/bench/pmctrack/src/modules/pmcs/core2/pmu_config_x86.c:155:6: warning: unused variable 'model_cpu' [-Wunused-variable]
      int model_cpu=0;
          ^
      CC [M]  /home/bench/pmctrack/src/modules/pmcs/core2/cbuffer.o
      CC [M]  /home/bench/pmctrack/src/modules/pmcs/core2/monitoring_mod.o
      CC [M]  /home/bench/pmctrack/src/modules/pmcs/core2/syswide.o
      CC [M]  /home/bench/pmctrack/src/modules/pmcs/core2/ipc_sampling_sf_mm.o
      LD [M]  /home/bench/pmctrack/src/modules/pmcs/core2/mchw_core2.o
      Building modules, stage 2.
      MODPOST 1 modules
      CC      /home/bench/pmctrack/src/modules/pmcs/core2/mchw_core2.mod.o
      LD [M]  /home/bench/pmctrack/src/modules/pmcs/core2/mchw_core2.ko
    make[1]: Leaving directory '/usr/src/linux-headers-3.17.3.pmctrack-x86+'
    Done!!
    ============================================
    Building libpmctrack ....
    ============================================
    make -C src all
    make[1]: Entering directory '/home/bench/pmctrack/src/lib/libpmctrack/src'
    cc -c -DHZ=250 -Wall -g -fpic -I ../include -I ../../../modules/pmcs/include/pmc -o core.o core.c
    cc -c -DHZ=250 -Wall -g -fpic -I ../include -I ../../../modules/pmcs/include/pmc -o pmu_info.o pmu_info.c
    cc -shared -DHZ=250 -o ../libpmctrack.so core.o pmu_info.o
    ar rcs ../libpmctrack.a core.o pmu_info.o 
    make[1]: Leaving directory '/home/bench/pmctrack/src/lib/libpmctrack/src'
    Done!!
    ============================================
    Building pmc-events ....
    ============================================
    cc  -DUSE_VFORK -Wall -g -I ../../modules/pmcs/include/pmc -I../../lib/libpmctrack/include   -c -o pmc-events.o pmc-events.c
    cc -o ../../../bin/pmc-events pmc-events.o  -L../../lib/libpmctrack -lpmctrack -static
    Done!!
    ============================================
    Building pmctrack ....
    ============================================
    cc  -DUSE_VFORK -Wall -g -I ../../modules/pmcs/include/pmc -I../../lib/libpmctrack/include   -c -o pmctrack.o pmctrack.c
    cc -o ../../../bin/pmctrack pmctrack.o  -L../../lib/libpmctrack -lpmctrack -static
    Done!!
    ============================================
    *** BUILD PROCESS COMPLETED SUCCESSFULLY ***


The `pmctrack-manager` retrieves key information from the system and builds the command-line tools as well as the different flavors of the PMCTrack kernel module compatible with the current platform. If the build fails, build errors can be found in the `build.log` file created in the current directory.

### Building PMCTrack from source for the Intel Xeon Phi

In order to build the various PMCTrack components from source for the Xeon Phi Coprocessor the `k1om-mpss-linux-gcc` cross-compiler must be used. Such a compiler is bundled with Intel MPSS. For a successful compilation, `pmctrack-manager` has to know the location of the Intel MPSS installation in the file system. In addition, the out-of-tree compilation of PMCTrack's kernel module for the Xeon Phi requires the build to be performed against a freshly built Linux kernel tree (MPSS version) with the PMCTrack patch. The entire source kernel tree must be found in the same system where the PMCTrack compilation is performed.

Once these requirements are met, `pmctrack-manager` can be used as follows to perform the build for the Xeon Phi:

    $ pmctrack-manager build-phi <mpss-root-dir> <kernel-sources-dir>

[1]: https://github.com/jcsaezal/pmctrack/tree/master/src/kernel-patches
[2]: https://github.com/jcsaezal/pmctrack