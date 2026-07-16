---
# Feel free to add content and custom Front Matter to this file.
# To modify the layout, see https://jekyllrb.com/docs/themes/#overriding-theme-defaults

title: "The PMCSched framework: scheduling and resource management made easy"
permalink: /pmcsched/
toc: true
#redirect_to: https://zildj1an.github.io/pmcsched-website/
---



PMCSched is an open-source framework designed to simplify the creation and testing of novel scheduling and resource-management strategies. This webpage introduces the framework, outlines its key abstractions, and provides a tutorial on its usage.

PMCSched was born as a continuation of [PMCTrack](https://pmctrack-linux.github.io/), an OS-oriented performance monitoring tool for Linux. With the PMCSched framework, we take PMCTrack’s potential one step further by easing scheduling development. In particular, PMCSched was designed to facilitate the development of scheduling or resource management algorithms in the operating system (kernel) space, while also allowing for cooperation with user space runtimes. Unlike other existing frameworks that require patching the Linux kernel to function, PMCSched makes it possible to incorporate new scheduling-related OS-level support in Linux via a kernel module (creating a PMCSched _plugin_) that can be loaded in unmodified (vanilla) kernels, making its adoption easier in production systems. 

## Main abstractions {#sec:pmcsched-abstractions}

PMCSched leverages PMCTrack's APIs for hardware performance monitoring (hardware PMCs) and [cache partitioning](https://www.intel.com/content/www/us/en/developer/articles/technical/introduction-to-cache-allocation-technology.html). Besides that, PMCSched prepares key data structures, such as `pmcsched_thread_data_t` for thread representation, `group_app_t` and `sched_app_t` for tracking applications at different levels, and `sched_thread_group_t` for managing scheduling decisions per _core group_, enabling scalable implementations. The framework also allows for efficient communication between applications and the OS kernel via shared memory, exposed through `/proc/pmc/schedctl`. Inspired by Solaris' `schedctl()`, PMCSched enables direct access to scheduling-related data using `mmap()`, allowing optimizations in loadable kernel modules without additional system calls. This capability helped us implement cache-partitioning policies and runtime-OS kernel interactions.

Building a PMCSched plugin boils down to instantiating an interface of scheduling operations (`sched_ops_t`) and implementing the corresponding interface functions in a separate .c file. This way, all plugins adhere to a "contract" -- this is, a predefined set of requirements, which is a common practice in kernel-module development. This interface is represented by the `sched_ops_t` structure, defined as follows:

```c
typedef struct sched_ops
{
  char* description;
  sched_policy_mm_t policy;
  unsigned long flags;
  struct list_head link_schedulers;
  pmcsched_counter_config_t* counter_config;

  /* Callbacks */
  int (*on_fork_thread)       (pmcsched_thread_data_t* t,
                            unsigned char is_new_app);
  void (*on_exec_thread)      (pmon_prof_t* prof);
  void (*on_active_thread)    (pmcsched_thread_data_t* t);
  void (*on_inactive_thread)  (pmcsched_thread_data_t* t);
  void (*on_exit_thread)      (pmcsched_thread_data_t* t);
  void (*on_free_thread)      (pmcsched_thread_data_t* t,
                            unsigned char is_last_thread);
  void (*on_migrate_thread)   (pmcsched_thread_data_t* t,
                            int prev_cpu, int new_cpu);
  int (*on_read_plugin)       (char *aux);
  int (*on_write_plugin)      (char *line);
  void (*on_migrate_thread)   (pmcsched_thread_data_t* t,
                            int prev_cpu, int new_cpu);
  void (*sched_timer_periodic)(void);
  void (*sched_kthread_periodic)
                              (sized_list_t* migration_list);
  int (*on_new_sample)        (pmon_prof_t* prof, int cpu,
                            pmc_sample_t* sample,int flags,void* data);
} sched_ops_t;
```

The structure `sched_ops_t` consists of a set of fields and callbacks. The purpose of these callbacks is as follows:

- **`on_fork_thread`**: Called when a thread invokes the `fork()` system call. It receives a `pmcsched_thread_data_t` parameter with thread information and an additional boolean parameter, `is_new_app`, to distinguish between newly created processes and existing process threads. This is an ideal location for plugins to initialize per-thread metrics.

- **`on_exec_thread`**: Invoked when a thread calls the `exec()` system call.

- **`on_active_thread`**: Executes when a thread becomes runnable, receiving the thread’s PMCSched descriptor as a parameter. Similarly, `on_inactive_thread` is invoked when the thread blocks, sleeps, or terminates.

- **`on_exit_thread`**: Triggered when a thread terminates execution.

- **`on_free_thread`**: Executes when the kernel frees the memory associated with a thread’s task structure. If the current thread is the last in its process, the `is_last_thread` parameter is set to `1`, signaling plugins to free process-wide data structures.

- **`on_migrate_thread`**: Called when a thread migrates from one core to another, enabling tracking of thread movement for load balancing.

- **`on_write_plugin`**: PMCSched exposes the `/proc/pmc/sched` special file for framework configuration. This callback allows plugins to expose configurable parameters, enabling users to update plugin-specific settings dynamically.

- **`on_read_plugin`**: Invoked when the `read()` system call is used on `/proc/pmc/sched`, allowing plugins to include relevant information in the output.

- **`sched_timer_periodic`**: Allows plugins to perform periodic operations per core group, such as recalculating scheduling metrics.

- **`sched_kthread_periodic`**: Called periodically by a kernel thread (`kthread`), allowing the execution of blocking operations such as thread migrations.

- **`on_new_sample`**: Defined by plugins that use PMCs to gather scheduling-relevant statistics per thread. Invoked when PMCTrack collects new samples, allowing plugins to compute high-level metrics from hardware event data.

This structure provides a flexible interface for implementing scheduling and resource-management strategies in PMCSched. The remaining fields in `sched_ops_t` serve the following purposes:

- **`policy`**: Stores a constant enumerated value that uniquely identifies the scheduling policy.

- **`description`**: Contains a human-readable description of the scheduling policy, displayed when reading the `/proc/pmc/sched` file.

- **`flags`**: Determines whether the plugin relies on PMCSched to handle locking synchronization (`PMCSCHED_CPUGROUP_LOCK`) or manages the locks independently (`PMCSCHED_CUSTOM_LOCK`). Plugins choosing the latter must handle their own synchronization, such as managing lists of active/inactive threads and avoiding race conditions, but gain finer control over locking mechanisms.

- **`link_schedulers`**: Allows the plugin’s descriptor to be inserted into the doubly-linked list of plugins maintained by PMCSched. Many Linux kernel data structures include `list_head` fields for integration into generic doubly-linked lists.

- **`counter_config`**: Defines the hardware performance counter configuration for PMC events gathered on a per-thread basis. This configuration follows PMCTrack’s raw event format. Additionally, it specifies the high-level metrics the plugin needs to compute from the gathered event values. PMCTrack provides an API to automate metric calculations and expose virtual counters to its components. If the plugin does not utilize performance monitoring counters, `counter_config` must be set to `NULL`.

## Creating a PMCSched plugin

To illustrate the process of plugin creation, let us walk through a simple example and its associated code. First, we must implement the predefined `sched_ops_t` interface (described in the previous section) in a separate `.c` source file. We begin by developing our new plugin by creating a new file named `example_plugin.c`. As explained, its functions will handle particular scheduling events. The minimal required callbacks, along with a policy ID, optional flags, and string description, are defined as follows:

```c
sched_ops_t thesis_plugin = 
{
    .policy                 = SCHED_EXAMPLE,
    .description            = "Example plugin",
    .flags                  = PMCSCHED_CPUGROUP_LOCK,
    .sched_timer_periodic   = sched_timer_periodic_example,
    .sched_kthread_periodic = sched_kthread_example,
    .on_exec_thread         = on_exec_thread_example,
    .on_active_thread       = on_active_thread_example,
    .on_inactive_thread     = on_inactive_thread_example,
    .on_fork_thread         = on_fork_thread_example,
    .on_exit_thread         = on_exit_thread_example,
    .on_migrate_thread      = on_migrate_thread_example,
};
```

The following code is an example of a possible `on_active_thread_example`, actions will be performed only on even-numbered invocations of the plugin’s functions. For this purpose, it uses an atomic counter to track invocations and only execute its main logic on even counts. When executed, it will unconditionally add the task to the list of active threads for both its associated application and the application group. It will also check if the task represents a newly created application. If the `DEBUG` option is enabled, the function will output relevant information to the kernel buffer, including the current invocation count. Implementing conditional behavior based on invocation frequency could be useful for various scheduling strategies, such as periodic task migrations or load balancing operations that don’t need to occur on every function call.

```c
#define DPRINTK(fmt, args...) \
do { if (IS_ENABLED(DEBUG)) \
trace_printk(fmt, ##args); } while (0)

static atomic_t invocation_count = ATOMIC_INIT(0);

static void

 on_active_thread_thesis(pmcsched_thread_data_t *t) {

    sched_thread_group_t *cur_group = get_cur_group_sched();
    app_t_pmcsched *app = get_group_app_cpu(t, 
        cur_group->cpu_group->group_id);
    int count = atomic_inc_return(&invocation_count);

    t->cur_group = cur_group;
    t->cmt_data = &app->app_cache.app_cmt_data;

    insert_sized_list_tail(&app->app_active_threads, t);
    insert_sized_list_tail(&cur_group->active_threads, t);

    if (sized_list_length(&app->app_active_threads) == 1) {
        insert_sized_list_tail(&cur_group->active_apps, app);
        DPRINTK("App active (invocation %d)\n", count);
    } /* Else, a thread of a multi-threaded app activated */

    if (count % 2 == 0)
        /* Some periodic action (core logic) ... */

    if (count >= (1 << 30))
        atomic_set(&invocation_count, 0);
}
```

Registering our new plugin in PMCSched requires declaring the plugin's descriptor in the framework's main header file, `pmcsched.h`. Since we implement the functions in a separate file (`example_plugin.c`), the descriptor must be declared as `extern`:

```c
extern struct sched_ops thesis_plugin;
```

Two additional changes in the header file are necessary to finish registering our `SCHED_EXAMPLE` plugin: first, in the enumeration of scheduling policies, and second, in the array of available schedulers. We start by adding the plugin's ID to the enumeration of available plugins:

```c
/* Supported scheduling policies */
typedef enum
{
    /* Examples of previous scheduling plugins */
    SCHED_DUMMY_MM=0,
    SCHED_GROUP_MM,
    SCHED_BUSYBCS_MM,
    /* New plugin */
    SCHED_EXAMPLE,
    NUM_SCHEDULERS
}
sched_policy_mm_t;
```

We now include the plugin's descriptor in the array of available plugins:

```c
static __attribute__ ((unused)) struct sched_ops*
    available_schedulers[NUM_SCHEDULERS] = 
    {
        /* Examples of previous scheduling plugins */
        &dummy_plugin,
        &group_plugin,
        &busybcs_plugin,
        /* Our new plugin */
        &thesis_plugin,
    };
```

The last step is to include the source file of our new plugin in the list of .c files to be compiled, which is found in the architecture-specific Makefile of PMCTrack's kernel module. For instance, to target Intel processors, the example plugin's object file (`example_plugin.o`) is added as shown below:

```c
MODULE_NAME=mchw_intel_core
obj-m += $(MODULE_NAME).o 
PMCSCHED-objs= pmcsched.o dummy_plugin.o group_plugin.o busy_plugin.o example_plugin.o
```

### Activating a plugin

Once the PMCTrack kernel module is loaded into the system (see PMCTrack's official documentation), the scheduling plugins can be selected and configured by writing to special files in Linux's procfs. To do so, we first activate PMCSched via the /proc/pmc/mm_manager file, managed by PMCTrack:

```bash
# Check PMCSched's monitoring module ID (platform specific)     
$ cat /proc/pmc/mm_manager
[*] 0 - This is just a proof of concept
[ ] 1 - IPC sampling SF estimation module
[ ] 2 - PMCSched
[ ] 3 - AMD QoS extensions (monitoring and allocation)    

## Activate PMCSched 
$ echo 'activate 2' > /proc/pmc/mm_manager

## Make sure that PMCSched has been activated 
$ cat /proc/pmc/mm_manager
[ ] 0 - This is just a proof of concept
[ ] 1 - IPC sampling SF estimation module
[*] 2 - PMCSched
[ ] 3 - AMD QoS extensions (monitoring and allocation)
```

Reading from `/proc/pmc/sched` allows us to determine which PMCSched plugin is currently active, as well as to retrieve the ID of our new plugin:

```bash
$ cat /proc/pmc/sched
The developed schedulers in PMCSched are:
[*] 0 - Dummy default plugin (Proof of concept)
[ ] 1 - Group Scheduling Plugin (Proof of concept)
[ ] 2 - Busy scheduler
[ ] 3 - Example scheduler
- To change the active scheduler echo 'scheduler <number>'
---
(Plugin specific output)
```

Once this ID is known -- referred to as `scheduler_id` --, the active plugin can be changed with the following command: `echo scheduler_id > /proc/pmc/sched`.

### Leveraging PMCs

Arguably, one of PMCSched's coolest features is its ability to collect information regarding Performance Monitoring Counters (PMCs), using the APIs provided by PMCTrack. You can configure your plugin to collect certain metrics and events, such as instruction count, cycles, LLC misses and LLC references. This is particularly interesting to profile entering applications.
Let us illustrate how to collect a number of interesting PMCs:

- Instructions per cycle (normalized).
- LLC accesses per instruction (normalized).
- LLC misses per instruction (normalized).
- LLC misses per cycle (normalized).

Firstly, we need to prepare the descriptors for the various performance metrics:

```c
 static metric_experiment_set_t metric_description= {
  .nr_exps=1, /* 1 set of metrics */
  .exps={
    /* Metric set 0 */
    {
      .metrics= {
        PMC_METRIC("IPC",op_rate,INSTR_EVT,CYCLES_EVT,1000),
        PMC_METRIC("RPKI",op_rate,LLC_ACCESSES,INSTR_EVT,1000000),
        PMC_METRIC("MPKI",op_rate,LLC_MISSES,INSTR_EVT,1000000),
        PMC_METRIC("MPKC",op_rate,LLC_MISSES,CYCLES_EVT,1000000),
        PMC_METRIC("STALLS_L2",op_rate,STALLS_L2_MISS,CYCLES_EVT,1000),
      },
      .size=NR_METRICS,
      .exp_idx=0,
    },
  }
};
```

using a number of metrics and their indexes, that we have to define upfront:

```c
enum event_indexes {
  INSTR_EVT=0,
  CYCLES_EVT,
  LLC_ACCESSES,
  LLC_MISSES,
  STALLS_L2_MISS,
  L2_LINES,
  PMC_EVENT_COUNT
};

enum metric_indices {
  IPC_MET=0,
  RPKI_MET,
  MPKI_MET,
  MPKC_MET,
  STALLS_L2_MET,
  NR_METRICS,
};
```

Finally, we can prepare the `pmcsched_counter_config_t`, which is the configuration exposed to PMCTrack. In the example below, we set the profiling mode to `TBS_SCHED_MODE` (as opposed to event based sampling, with `EBS_SCHED_MODE`).

```c
static pmcsched_counter_config_t cconfig={
    .counter_usage={
        .hwpmc_mask=0x3b, /*  bitops -h 0,1,3,4,5 */
        .nr_virtual_counters=CMT_MAX_EVENTS,
        .nr_experiments=1,
        .vcounter_desc={"llc_usage","total_llc_bw","local_llc_bw"},
    },
    .pmcs_descr=&pmc_configuration,
    .metric_descr={&metric_description,NULL},
    .profiling_mode=TBS_SCHED_MODE,
};
```

which we can pass as part of the plugin definition, using field counter_config. We also specify that for every new sample collected, we want PMCSched to call our plugin's function `profile_thread_example()`.

```c
sched_ops_t lfoc_plugin = {
    .policy                 = SCHED_EXAMPLE_PMCs,
    .description            = "Plugin that uses PMCs",
    .counter_config=&cconfig,
	(...)
    .on_new_sample = profile_thread_example,
};
```

The profiling function can then update global instruction counters from the sample:

```c
static int profile_thread_example(pmon_prof_t* prof,
    int cpu,pmc_sample_t* sample,int flags,void* data)
{
	pmcsched_thread_data_t* t = prof->monitoring_mod_priv_data;
	(...)
	lt->instr_counter += sample->pmc_counts[0];
}
```

and use this information to decide, depending on the algorithm, how to classify the application.


<!--

## Publications

Here's a list of relevant publications that leverage PMCSched: 

- Carlos Bilbao, Juan Carlos Saez, Manuel Prieto-Matias, _Flexible system software scheduling for asymmetric multicore systems with PMCSched: A case for Intel Alder Lake_. Concurrency and Computation: Practice and Experience, 2023. DOI: 10.1002/cpe.7814

-  Carlos Bilbao, Juan Carlos Saez, Manuel Prieto-Matias, _Divide&Content: A Fair OS-Level Resource Manager for Contention Balancing on NUMA Multicores_, IEEE Transactions on Parallel and Distributed Systems, 2023. DOI: 10.1109/TPDS.2023.3309999

- Javier Rubio, Carlos Bilbao, Juan Carlos Saez, Manuel Prieto-Matias, _Exploiting Elasticity via OS-runtime Cooperation to Improve CPU Utilization in Multicore Systems_. 32nd Euromicro International Conference on Parallel, Distributed and Network-Based Processing (PDP), Dublin, Irlanda, 2024.. DOI: 10.1109/PDP62718.2024.00014.

- Carlos Bilbao, Juan Carlos Saez, Manuel Prieto-Matias, _Rapid Development of OS Support with PMCSched for Scheduling on Asymmetric Multicore Systems_. Euro-Par 2022: Parallel Processing Workshops. Also in book Lecture Notes in Computer Science (LNCS, volume 13835). DOI: 10.1007/978-3-031-31209-0_14


-->

## New support for Docker containers

As part of [PMCTrack's v4.0 release](https://github.com/jcsaezal/pmctrack/tree/v4.0), new functionality has been added to PMCSched for managing containerized applications from within resource-management and scheduling plugins. This support facilitates the development of plugins that operate with  applications running in Docker containers. Many real-world cloud workloads and HPC applications spawn multiple processes, such as those based on Apache Spark, Hadoop, MPI, etc. Managing workloads that feature these applications is challenging at the kernel level, as the Linux scheduler makes scheduling decisions at the thread and process levels, but not at the container level. The new introduced container support has been designed to greatly simplify container-level decision making from PMCSched plugins.

To this end, we had to substantially augment the functionality of the PMCSched framework. The new container support allows plugin developers to seamlessly manage all threads in the same container as a single group, by leveraging a data structure that represents a Docker container as a set of threads belonging to the same Linux *cgroup*. This structure can be handled from a PMCSched plugin in much the same way as a multithreaded application, and enables developers to keep track of the container's runnable threads and other relevant metrics across different parts of a multicore system: within the same socket, inside a set of cores sharing a last-level cache, etc. 

To ensure that a plugin that leverages this new support manages a particular containerized application, the user must first create the associated container (e.g., via Docker's CLI tools), and then activate the container's *cgroup* in the kernel, by writing the `"register_cgroup <PID>"` string to `/proc/pmc/sched`, where PID is the process ID of the *init* process in the associated Docker container. This PID can be easily retrieved via the `docker inspect` command.

## Contact

You can contact the two main project contributors:

- Juan Carlos Sáez Alcaide (jcsaezal at ucm.es)
- Carlos Bilbao (cbilbao at ucm.es)