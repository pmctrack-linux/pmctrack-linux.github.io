---
layout: page
title: Getting Started
permalink: /getting-started/
toc: true
---



Once the [build process][1] has been completed, PMCTrack can be used from user space or from the OS scheduler (kernel level).

### Using PMCTrack from user space

To get started, go to the root directory of you local copy of PMCTrack's repository and set up the necessary environment variables as follows:

    $ . shrc


After that, load any of the available flavors of the PMCTrack's kernel module compatible with your processor. Note that builds performed with `pmctrack-manager` (as shown [here][1]), only compile those flavors of the kernel module that may be suitable for your system.

The following table summarizes the properties of the various flavors of the kernel module:


| Name | Path of the .ko file | Supported processors |
| -----| ---------------------| ---------------------|
| intel-core | `src/modules/pmcs/intel-core/mchw_intel_core.ko` | Most Intel multi-core processors are compatible with this module, including recent processors based on the Intel "Broadwell" microarchitecture. |
| amd | `src/modules/pmcs/amd/mchw_amd.ko` | This module has been successfully tested on AMD opteron processors. Nevertheless, it should be compatible with all AMD multicore processors. |
| arm | `src/modules/pmcs/arm/mchw_arm.ko` | This module has been successfully tested on ARM systems featuring 32-bit big.LITTLE processors, which combine ARM Cortex A7 cores with and ARM Cortex A15 cores. Specifically, tests were performed on the ARM Coretile Express Development Board (TC2). |
| odroid-xu | `src/modules/pmcs/odroid-xu/mchw_odroid_xu.ko` | Specific module for Odroid XU3 and XU4 boards. More information on these boards can be found at [www.hardkernel.com](http://www.hardkernel.com) |
| arm64 | `src/modules/pmcs/arm64/mchw_arm64.ko` | This module has been successfully tested on ARM systems featuring 64-bit big.LITTLE processors, which combine ARM Cortex A57 cores with and ARM Cortex A53 cores. Specifically, tests were performed on the ARM Juno Development Board. |
| xeon-phi | `src/modules/pmcs/xeon-phi/mchw_phi.ko` | Intel Xeon Phi Coprocessor |
| core2 | `src/modules/pmcs/phi/mchw_core2.ko` | This module has been specifically designed for the Intel QuickIA prototype system. The Intel QuickIA is a dual-socket asymmetric multicore system that features a quad-core Intel Xeon E5450 processor and a dual-core Intel Atom N330. The module also works with Intel Atom processors as well as "old" Intel multicore processors, such as the Intel Core 2 Duo. Nevertheless, given the numerous existing hacks for the QuickIA in this module, users are advised to use the more general "intel-core" flavor.  |
| perf | `src/modules/pmcs/perf/mchw_perf.ko` | Backend that uses Perf Events's kernel API to access performance monitoring counters. It currently works for Intel, AMD, ARMv7 and ARMv8 processors, only. |


Once the most suitable kernel model for the system has been identified, the module can be loaded in the running PMCTrack-enabled kernel as follows:

    $ sudo insmod <path_to_the_ko_file>


If the command did not return errors, information on the detected Performance Monitoring Units (PMUs) found in the machine can be retrieved by reading from the `/proc/pmc/info` file:

    $ cat /proc/pmc/info 
    *** PMU Info ***
    nr_core_types=1
    [PMU coretype0]
    pmu_model=x86_intel-core.haswell-ep
    nr_gp_pmcs=8
    nr_ff_pmcs=3
    pmc_bitwidth=48
    ***************
    *** Monitoring Module ***
    counter_used_mask=0x0
    nr_experiments=0
    nr_virtual_counters=0
    ***************


Alternatively the `pmc-events` helper command can be used to get the same information in a slightly different format:

    $ pmc-events -I
    [PMU 0]
    pmu_model=x86_intel-core.haswell-ep
    nr_fixed_pmcs=3
    nr_gp_pmcs=8


On systems featuring asymmetric multicore processors, such as the ARM big.LITTLE, the `pmc-event` command will list as many PMUs as different core types exist in the system. On a 64-bit ARM big.LITTLE processor the output will be as follows:

    $ pmc-events -I
    [PMU 0]
    pmu_model=armv8.cortex_a53
    nr_fixed_pmcs=1
    nr_gp_pmcs=6
    [PMU 1]
    pmu_model=armv8.cortex_a57
    nr_fixed_pmcs=1
    nr_gp_pmcs=6


To obtain a listing of the hardware events supported, the following command can be used:

    $ pmc-events -L
    [PMU 0]
    instr_retired_fixed
    unhalted_core_cycles_fixed
    unhalted_ref_cycles_fixed
    instr
    cycles
    unhalted_core_cycles
    instr_retired
    unhalted_ref_cycles
    llc_references
    llc_references.prefetch
    llc_misses
    llc_misses.prefetch
    branch_instr_retired
    branch_mispred_retired
    l2_references
    l2_misses
    ...


#### The `pmctrack` command-line tool

The `pmctrack` command is the most straigthforward way of accessing PMCTrack functionality from user space. The available options for the command can be listed by just typing `pmctrack` in the console:

    $ pmctrack
    Usage: pmctrack [OPTION [OP. ARGS]] [PROG [ARGS]]
    Available options:
            -c      <config-string>
                    set up a performance monitoring experiment using either raw or mnemonic-based PMC string
            -o      <output>
                    output: set output file for the results. (default = stdout.)
            -T      <Time>
                    Time: elapsed time in seconds between two consecutive counter samplings. (default = 1 sec.)
            -b      <cpu or mask>
                    bind launched program to the specified cpu o cpumask.
            -n      <max-samples>
                    Run command until a given number of samples are collected
            -N      <secs>
                    Run command for secs seconds only
            -e
                    Enable extended output
            -A
                    Enable aggregate count mode
            -k      <kernel_buffer_size>
                    Specify the size of the kernel buffer used for the PMC samples
            -b      <cpu or mask>
                    bind monitor program to the specified cpu o cpumask.
            -S
                    Enable system-wide monitoring mode (per-CPU)
            -r
                    Accept pmc configuration strings in the RAW format
            -p      <pmu>
                    Specify the PMU id to use for the event configuration
            -L
                    Legacy-mode: do not show counter-to-event mapping
            -t
                    Show real, user and sys time of child process
    PROG + ARGS:
                    Command line for the program to be monitored. 


Before introducing the basics of this command, it is worth describing the semantics of the `-c` and `-V` options. Essentially, the -c option accepts an argument with a string describing a set of hardware events to be monitored. This string consists of comma-separated event configurations; in turn, an event configuration can be specified using event mnemonics or event hex codes found in the PMU manual provided by the processor's manufacturer. For example the hex-code based string `0x8,0x11` for an ARM Cortex A57 processor specifies the same event set than that of the `instr,cycles` string. Clearly, the latter format is far more intuitive than the former; the user can probably guess that we are trying to specify the hardware events "retired instructions" and "cycles".

The `-V` option makes it possible to specify a set of *virtual counters*. Modern systems enable monitoring a set of hardware events using PMCs. Still, other monitoring information (e.g., energy consumption) may be exposed to the OS by other means, such as fixed-function registers, sensors, etc. This "non-PMC" data is exposed by the PMCTrack kernel module as *virtual counters*, rather than HW events. PMCTrack monitoring modules are in charge of implementing low-level access to *virtual counters*. To retrieve the list of HW events exported by the active monitoring module use `pmc-events -V`. More information on PMCTrack monitoring modules can be found [here][4]. The `pmctrack` command supports three usage modes:

1.  **Time-Based Sampling (TBS)**: PMC and virtual counter values for a certain application are collected at regular time intervals.
2.  **Event-Based Sampling (EBS)**: PMC and virtual counter values for an application are collected every time a given PMC event reaches a given count.
3.  **Time-Based system-wide monitoring mode**: This mode is a variant of the TBS mode, but monitoring information is provided for each CPU in the system, rather than for a specific application. This mode can be enabled with the `-S` switch. 

To illustrate how the TBS mode works let us consider the following example command invoked on a system featuring a quad-core Intel Xeon Haswell processor:

    $ pmctrack  -T 1  -c instr,llc_misses -V energy_core  ./mcf06
    [Event-to-counter mappings]
    pmc0=instr
    pmc3=llc_misses
    virt0=energy_core
    [Event counts]
    nsample    pid      event          pmc0           pmc3         virt0
          1  10767       tick    2017968202       30215772       7930969
          2  10767       tick    1220639346       24866936       7580993
          3  10767       tick    1204660012       24726068       7432617
          4  10767       tick    1524589394       20013147       8411560
          5  10767       tick    1655802083        9520886       8531860
          6  10767       tick    2555712483       18420142       6615844
          7  10767       tick    2222232510       19594864       6385986
          8  10767       tick    1348937378       22795510       5966308
          9  10767       tick    1455948820       22282935       5994934
         10  10767       tick    1324007762       22682355       5951354
         11  10767       tick    1345928005       22477525       5968872
         12  10767       tick    1345868008       22400733       6024780
         13  10767       tick    1370194276       22121318       6024169
         14  10767       tick    1329712408       22154371       6030700 
         15  10767       tick    1365130132       21859147       6076293 
         16  10767       tick    1315829803       21780616       6136962 
         17  10767       tick    1357349957       20889360       6234619 
         18  10767       tick    1377910047       19539232       6519897 
    ...


This command provides the user with the number of instructions retired, last-level cache (LLC) misses and core energy consumption (in uJ) every second. The beginning of the command output shows the event-to-counter mapping for the various hardware events and virtual counters. The "Event counts" section in the output displays a table with the raw counts for the various events; each sample (one per second) is represented by a different row. Note that the sampling period is specified in seconds via the -T option; fractions of a second can be also specified (e.g, 0.3 for 300ms). If the user includes the -A switch in the command line, `pmctrack` will display the aggregate event count for the application's entire execution instead. At the end of the line, we specify the command to run the associated application we wish to monitor (e.g: ./mcf06).

In case a specific processor model does not integrate enough PMCs to monitor a given set of events at once, the user can turn to PMCTrack's event-multiplexing feature. This boils down to specifying several event sets by including multiple instances of the -c switch in the command line. In this case, the various events sets will be collected in a round-robin fashion and a new `expid` field in the output will indicate the event set a particular sample belongs to. In a similar vein, time-based sampling also supports multithreaded applications. In this case, samples from each thread in the application will be identified by a different value in the pid column.

Event-based Sampling (EBS) constitutes a variant of time-based sampling wherein PMC values are gathered when a certain event count reaches a certain threshold *T*. To support EBS, PMCTrack's kernel module exploits the interrupt-on-overflow feature present in most modern Performance Monitoring Units (PMUs). To use the EBS feature from userspace, the "ebs" flag must be specified in `pmctrack` command line by an event's name. In doing so, a threshold value may be also specified as in the following example:

    $ pmctrack -c instr:ebs=500000000,llc_misses -V energy_core  ./mcf06 
    [Event-to-counter mappings]
    pmc0=instr
    pmc3=llc_misses
    virt0=energy_core
    [Event counts]
    nsample    pid      event          pmc0          pmc3         virt0
          1  10839        ebs     500000078        892837        526489 
          2  10839        ebs     500000047       9383500       1946166 
          3  10839        ebs     500000050       9692922       2544250 
          4  10839        ebs     500000007      10017122       2818298 
          5  10839        ebs     500000012       9907918       3055236 
          6  10839        ebs     500000011      10335579       3108215 
          7  10839        ebs     500000046      10735151       3118713 
          8  10839        ebs     500000011      10335980       3119140 
          9  10839        ebs     500000004      10250777       3053100 
         10  10839        ebs     500000019      11382679       2997802 
         11  10839        ebs     500000035       6650139       2587890 
         12  10839        ebs     500000004        474847       2596313 
         13  10839        ebs     500000039        532301       2601074 
         14  10839        ebs     500000019        577618       2617187 
         15  10839        ebs     500000062       6221112       2442504 
         16  10839        ebs     500000037       9177684       2325317 
         17  10839        ebs     500000058       2697348       1106994 
         18  10839        ebs     500000006       3520781       1264404 
         19  10839        ebs     500000055       2777145       1119934 
         20  10839        ebs     500000034       1965457        964843 
         21  10839        ebs     500000004       2290861       1035095 
         22  10839        ebs     500000011       3276917       1217895 
         23  10839        ebs     500000041       4202958       1409973 
         24  10839        ebs     500000034       5343461       1608947
    ...


The `pmc3` and `virt0` columns display the number of LLC misses and energy consumption every 500 million retired instructions. Note, however, that values in the `pmc0` column do not reflect exactly the target instruction count. This has to do with the fact that, in modern processors, the PMU interrupt is not served right after the counter overflows. Instead, due to the out-of-order and speculative execution, several dozen instructions or more may be executed within the period elapsed from counter overflow until the application is actually interrupted. These inaccuracies do not pose a big problem as long as coarse instruction windows are used.

#### Libpmctrack

Another way of accessing PMCTrack functionality from user space is via *libpmctrack*. This library enables to characterize performance of specific code fragments via PMCs and virtual counters in sequential and multithreaded programs written in C or C++. Libpmctrack's API makes it possible to indicate the desired PMC and virtual-counter configuration to the PMCTrack's kernel module at any point in the application's code or within a runtime system. The programmer may then retrieve the associated event counts for any code snippet (via TBS or EBS) simply by enclosing the code between invocations to the `pmctrack_start_count*()` and `pmctrack_stop_count()` functions. To illustrate the use of libpmctrack, several example programs are provided in the repository under `test/test_libpmctrack`.

### Using PMCTrack from the OS scheduler

PMCTrack allows any scheduling algorithm in the Linux kernel (i.e., scheduling class) to collect per-thread monitoring data, thus making it possible to drive scheduling decisions based on tasks' memory behavior or other runtime properties, such as energy consumption. Turning on this mode for a particular thread from the scheduler's code boils down to activating the `prof_enabled` flag in the thread's descriptor. 

To ensure that the implementation of the scheduling algorithm that benefits from this feature remains architecture independent, the scheduler itself (implemented in the kernel) does not configure nor deals with performance counters directly. Instead, the active monitoring module in PMCTrack is in charge of feeding the scheduling policy with the necessary high-level performance monitoring metrics, such as a task's instruction per cycle ratio or its last-level-cache miss rate. As described in the next section, the active monitoring module can be selected by writing in a special file.

The OS scheduler can communicate with the active monitoring module to obtain per-thread data via the following function from PMCTrack's kernel API:

    int pmcs_get_current_metric_value( struct task_struct* task, int metric_id, uint64_t* value );


For simplicity, each metric is assigned a numerical ID, known by the scheduler and the monitoring module. To obtain the up-to date value for a specific metric, the aforementioned function may be invoked from the tick processing function in the scheduler.

Monitoring modules make it possible for a scheduling policy relying on PMC or virtual counter metrics to be seamlessly extended to new architectures or processor models as long as the hardware enables to collect necessary monitoring data.

<a name="monitoring-modules"></a>

### PMCTrack monitoring modules

Monitoring modules are platform-specific components that enable to extend the functionality of PMCTrack's kernel module. The goal of monitoring modules is twofold. First, they make it possible to add support for additional hardware monitoring facilities not managed already by the base PMCTrack distribution. Overall, any kind of insightful monitoring information provided by modern hardware but not modeled directly via performance monitoring counters (PMCs), such as temperature sensors or energy consumption registers, can be exposed by a monitoring module as a *virtual counter* to PMCTrack userland tools. Second, as mentioned earlier, monitoring modules can provide a OS scheduling algorithm implemented in the Linux kernel with high-level monitoring metrics it may require to function, such as performance ratios across cores or power consumption estimates.

<!--
Note that PMCtrack kernel module features an architecture-independent API making it possible to access PMCs within a monitoring module. As such, the module can easily maintain up-to-date values of per-thread high-level metrics required by the a given scheduling algorithm to make internal decisions.
-->

PMCTrack may include several monitoring modules compatible with a given platform. However, only one can be enabled at a time. Monitoring modules available for the current system can be obtained by reading from the `/proc/pmc/mm_manager` file:

    $ cat /proc/pmc/mm_manager 
    [*] 0 - This is just a proof of concept
    [ ] 1 - IPC sampling-based SF estimation model
    [ ] 2 - PMCtrack module that supports Intel CMT
    [ ] 3 - PMCtrack module that supports Intel RAPL


In the example above, four monitoring modules are listed and module #0, marked with "*", is the *active* monitoring module.

In the event several compatible monitoring modules exist, the system administrator may tell the system which one to use by writing in the `/proc/pmc/mm_manager` file as follows:

    $ echo 'activate 3' > /proc/pmc/mm_manager
    $ cat /proc/pmc/mm_manager 
    [ ] 0 - This is just a proof of concept
    [ ] 1 - IPC sampling-based SF estimation model
    [ ] 2 - PMCtrack module that supports Intel CMT
    [*] 3 - PMCtrack module that supports Intel RAPL


The `pmc-events` command can be used to list the virtual counters exported by the active monitoring module, as follows:

    $ pmc-events -V
    [Virtual counters]
    energy_pkg
    energy_dram


From the programmer's standpoint, creating a monitoring module entails implementing the `monitoring_module_t` interface (`<pmc/monitoring_mod.h>`) in a separate .c file of the PMCTrack kernel module sources. Several sample monitoring modules are provided along with the PMCTrack distribution; its source code can be found in the `*_mm.c` files found in `src/modules/pmcs`.

The `monitoring_module_t` interface consists of several callback functions:

```C
typedef struct {
    char info[MAX_CHARS_EST_MOD];   
    struct list_head    links;      
    int     (*probe_module)(void);                                  
    int     (*enable_module)(void); 
    void    (*disable_module)(void);
    int     (*on_read_config)(char* s, unsigned int maxchars); 
    int     (*on_write_config)(const char *s, unsigned int len);
    int     (*on_fork)(unsigned long clone_flags, pmon_prof_t*); 
    void    (*on_exec)(pmon_prof_t*);
    int     (*on_new_sample)(pmon_prof_t* prof, int cpu, pmc_sample_t* sample,
                            int flags, void* data);                  
    void    (*on_migrate)(pmon_prof_t* p, int prev_cpu, int new_cpu);
    void    (*on_exit)(pmon_prof_t* p);                         
    void    (*on_free_task)(pmon_prof_t* p);    
    void    (*on_switch_in)(pmon_prof_t* p);    
    void    (*on_switch_out)(pmon_prof_t* p);                           
    int     (*get_current_metric_value)(pmon_prof_t* prof, int key, uint64_t* value); 
    void    (*module_counter_usage)(monitoring_module_counter_usage_t* usage);
    int     (*on_syswide_start_monitor)(int cpu, unsigned int virtual_mask);
    void    (*on_syswide_stop_monitor)(int cpu, unsigned int virtual_mask);
    void    (*on_syswide_refresh_monitor)(int cpu, unsigned int virtual_mask);  
    void    (*on_syswide_dump_virtual_counters)(int cpu, unsigned int virtual_mask, 
                                    pmc_sample_t* sample);
} monitoring_module_t;
```



The programmer typically implements only the subset of callbacks required to carry out the necessary internal processing. Many of these callback functions are invoked to make the monitoring module aware of key scheduling events, such as when a thread enters the system (`on_fork()`), when it terminates the execution (`on_exit()`) or when a context switch is performed (`on_switch_in()` and `on_switch_out()`). These notifications make it possible to initialize, save and restore monitoring-related registers so as to keep track of information of different threads separately. The `get_current_metric_value()` operation must be implemented if the monitoring module exposes per-thread high-level metrics to a scheduling algorithm implemented in the kernel. Another key operation is `module_counter_usage()`, which is meant to expose essential metadata about the monitoring module to PMCTrack userland tools, such as the set of virtual counters implemented by the monitoring module or the set of physical performance counters used (if any). More information on the remaining callbacks can be found in the source code (`src/modules/pmcs/include/pmc/monitoring_mod.h`).

PMCTrack's kernel module enables monitoring modules to take full control of performance monitoring counters to obtain high-level per-thread monitoring information. This information can be used to feed the OS scheduler with simple PMC metrics such the number of instructions per cycle (IPC), or to apply sophisticated estimation models to obtain other meaningful metrics, such as the estimated power consumption. Notably, in doing so, monitoring module developers do not have to deal with performance counter registers directly on a given architecture. Instead, developers use an architecture-independent interface enabling to easily configure HW events and gather PMC data. Specifically, when a new is created (the `on_fork()` callback is invoked) the monitoring module can impose the set(s) of performance events to be monitored by using a function of PMCTrack's kernel module API. Whenever new PMC samples are collected for a thread (i.e., the sampling period expires), the `on_new_sample()` operation of the monitoring module interface is invoked, passing the samples as a parameter.

As shown above, several operations in the `monitoring_module_t` interface accept a parameter of type `pmon_proc_t*`. This parameter is a structure where PMCTrack's kernel module stores monitoring information for a thread. A pointer to this structure is stored in the new `prof` field introduced in the task descriptor (`struct task_struct`) when applying PMCTrack kernel patch. The definition of the `pmon_proc_t` can be found in the kernel module sources (`<pmc/mc_experiments.h>`). For monitoring module developers, the most relevant fields in `pmon_proc_t` are as follows:

*   `void* monitoring_mod_priv_data`: pointer that the active monitoring module can use to store private per-thread data. In the event that the monitoring module uses private data, this pointer must be initialized in the `on_fork()` operation and freed up (if necessary) in the `on_free_task()` operation.
*   `core_experiment_t* pmcs_config`: pointer to a structure that describes the current set of hardware events being monitored for the thread. Note that several sets of HW events can be monitored for a thread. In case several event sets exist, the different event sets will be monitored in a round-robin fashion. This feature turns out useful when the number of hardware events to be monitored exceeds the number of per-core physical counters. Notably, PMCTack allows to specify different sets of HW events for different core types in asymmetric multicore systems, such as embedded platforms featuring the ARM big.LITTLE processor.
*   `core_experiment_set_t pmcs_multiplex_cfg[]`: describes the sets of HW events configured for the various core types in the system. 

In order to indicate the set (or sets) of hardware events to monitor for a thread, the following functions can be used in a monitoring module:

    /* 
     * Given a null-terminated array of raw-formatted PMC configuration 
     * string, store the associated low-level information into an array of core_experiment_set_t.  
     */
    int configure_performance_counters_set(const char* strconfig[], 
                        core_experiment_set_t pmcs_multiplex_cfg[], 
                        int nr_coretypes);
    
    /* Copy the configuration of a experiment set into another */
    int clone_core_experiment_set_t(core_experiment_set_t* dst,
                                    core_experiment_set_t* src);


Specifically, `configure_performance_counters_set()` enables to indicate the sets of hardware events to be monitored encoded in a null-terminated array of strings (`strconfig` parameter). Each array item encodes a set of events using PMCTrack's raw string format. This low-level format stands in contrast to the mnemonic-based format used by PMCTrack userland tools. Using mnemonics is not well-suited to in-kernel event monitoring as translating mnemonics into the actual hex values written in PMC registers may involve traversing rather long event tables. To avoid the associated overhead, monitoring module developers must specify event configurations using the raw format. In using this format, the hardware event (hex) codes and the event-to-physical-counter mappings have to be specified by the programmer. To obtain the raw configurations string for a given mnemonic-based representation monitoring module developers may turn to PMCTrack's `pmc-events` helper command. For example, the command below, executed on a system with a modern Intel processor, enables to obtain the raw format encoding the following three HW events: instructions retired, cycles and last-level-cache misses.

    $ pmc-events instr,cycles,llc_misses
    pmc0,pmc1,pmc3=0x2e,umask3=0x41


On a system featuring an ARM Cortex A57 processor, the associated raw string can be obtained using the same command:

    $ pmc-events instr,cycles,llc_misses
    pmc1=0x8,pmc2=0x11,pmc3=0x17 


In the raw string for the Intel processor, performance counters #0, #1 and #3 will be used to monitor the events. By contrast, performance counters #1, #2 and #3 are selected in the raw string for the ARM processor.

The obtained raw strings can be used in a monitoring module to indicate the set of hardware events to use on the processor in question. For example, on the Intel system, the following array can be passed as the first parameter of `configure_performance_counters_set()` to monitor these events in a monitoring module:

    char* events[]={"pmc0,pmc1,pmc3=0x2e,umask3=0x41",NULL};


Assuming `prof` is a pointer of a per-thread monitoring structure (`prof_t`), the `configure_performance_counters_set()` can be invoked as follows to impose a set of monitoring events for a thread on a symmetric multicore system:

    configure_performance_counters_set(events,&prof->pmcs_multiplex_cfg[0],1);


The code above can be invoked directly from the `on_fork()` callback of the monitoring module. Notably, if the monitoring module uses the same set of HW events for all threads, performing the translation between the raw format string and the underlying `core_experiment_set_t` structure every time the the `on_fork()` callback is invoked (a thread enters the system) may not be the most suitable option. In this scenario, the monitoring module can perform the translation just once when the monitoring module is activated (`on_activate()` callback). To make that possible, the monitoring module must define a global `core_experiment_set_t` structure initialized with `configure_performance_counters_set()` when the monitoring module is activated. When a thread is created, the `clone_core_experiment_set_t()` function can be used to impose the hardware counter configuration, by creating a copy of the global `core_experiment_set_t` structure. This approach is employed in various monitoring modules already available in PMCTrack such as the one implemented in `src/modules/pmcs/ipc_sampling_sf_mm.c`.

[1]: /install
[2]: pmctrack-on-odroid-xu3-4
[3]: http://www.hardkernel.com
[4]: #monitoring-modules