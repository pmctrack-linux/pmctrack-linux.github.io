---
layout: page
title: Migrating PMCTrack patches to another kernel version
permalink: /migrating-patches/
toc: false
---

PMCTrack v2.0 and later works on top of vanilla kernels starting from v5.9.x. On systems requiring an older kernel, a patch must be applied. A number of PMCTrack kernel patches for various Linux versions can be found in the `src/kernel-patches` directory. If a patch is not available for the desired kernel version, a custom patch can be easily created with git.

This tutorial describes how to adapt an existing kernel patch to a newer kernel version. To illustrate how this is done we will create a PMCTRack kernel patch for Linux v5.7.19, reusing an existing patch available for Linux v5.4.35.

The first step is to clone a git repository with the kernel sources. We will use the official repository for kernel stable versions (vanilla):

```bash
user@host:~/tutorial$ git clone git://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git
...
user@host:~/tutorial$ cd linux-stable-git/
user@host:~/tutorial/linux-stable-git$ 
```

Next, create new branch with the source code of the original kernel version for which a patch is available (v5.4.35 in our case):

```bash
user@host:~/tutorial/linux-stable-git$ git checkout -b pmctrack-linux-v5.4.35 v5.4.35
Updating files: 100% (56302/56302), done.
Switched to a new branch 'pmctrack-linux-v5.4.35'
```

Apply existing kernel patch on top of this branch using the `patch` command:

```bash
user@host:~/tutorial/linux-stable-git$ patch -p1 < ~/pmctrack/src/kernel-patches/pmctrack_linux-5.4.35_x86.patch 
patching file arch/arm64/kernel/perf_event.c
patching file arch/x86/events/core.c
patching file drivers/hwmon/hwmon.c
patching file include/linux/pmctrack.h
patching file include/linux/sched.h
patching file init/Kconfig
patching file kernel/Makefile
patching file kernel/exit.c
patching file kernel/fork.c
patching file kernel/irq/manage.c
patching file kernel/irq_work.c
patching file kernel/pmctrack.c
patching file kernel/sched/core.c
patching file kernel/sched/sched.h
```

Add the two new files created by the patch, and commit all the changes:

```bash
user@host:~/tutorial/linux-stable-git$ git add include/linux/pmctrack.h kernel/pmctrack.c
user@host:~/tutorial/linux-stable-git$ git commit -am "PMCTrack kernel patch"
[pmctrack-linux-v5.4.35 38fc98c25c31] PMCTrack kernel patch
 14 files changed, 549 insertions(+), 3 deletions(-)
 create mode 100644 include/linux/pmctrack.h
 create mode 100644 kernel/pmctrack.c
```

Write down the commit hash. According to the output above, the hash is `38fc98c25c31`.

Make sure that the branch has been  actually created:

```bash
user@host:~/tutorial/linux-stable-git$ git branch
* pmctrack-linux-v5.4.35
```

Let's now create another branch with the sources of the new kernel where we want to apply the patch. In this case, it's v5.7.19:

```bash
user@host:~/tutorial/linux-stable-git$ git checkout -b pmctrack-linux-v5.7.19 v5.7.19
Updating files: 100% (23895/23895), done.
Switched to a new branch 'pmctrack-linux-v5.7.19'
```

Finally apply patch via `git cherry-pick` and using the hash of the previous commit:

```bash
user@host:~/tutorial/linux-stable-git$ git cherry-pick 38fc98c25c31
Auto-merging kernel/sched/sched.h
Auto-merging kernel/sched/core.c
CONFLICT (content): Merge conflict in kernel/sched/core.c
Auto-merging kernel/irq_work.c
Auto-merging kernel/irq/manage.c
Auto-merging kernel/fork.c
Auto-merging kernel/exit.c
Auto-merging kernel/Makefile
Auto-merging init/Kconfig
Auto-merging include/linux/sched.h
Auto-merging drivers/hwmon/hwmon.c
Auto-merging arch/x86/events/core.c
Auto-merging arch/arm64/kernel/perf_event.c
warning: inexact rename detection was skipped due to too many files.
warning: you may want to set your merge.renamelimit variable to at least 3497 and retry the command.
error: could not apply 38fc98c25c31... PMCTrack kernel patch
hint: after resolving the conflicts, mark the corrected paths
hint: with 'git add <paths>' or 'git rm <paths>'
hint: and commit the result with 'git commit'
```

As we can see a conflict was detected during the merge procedure. So we investigate the issue with `git diff`:

```bash
user@host:~/tutorial/linux-stable-git$ git diff kernel/sched/core.c
diff --cc kernel/sched/core.c
index ebecf1cc3b78,26b20bd665c6..000000000000
--- a/kernel/sched/core.c
+++ b/kernel/sched/core.c
...
...
++<<<<<<< HEAD
 +              psi_sched_switch(prev, next, !task_on_rq_queued(prev));
 +
++=======
+ #ifdef CONFIG_PMCTRACK
+               pmcs_save_callback(prev->pmc, cpu);
+ #endif                
++>>>>>>> 38fc98c25c31 (PMCTrack kernel patch)
                trace_sched_switch(preempt, prev, next);
  
                /* Also unlocks the rq: */
```

The issue is easy to fix, as the newer kernel version just added an invocation to `psi_sched_switch()` before `trace_sched_switch()`. All we have to do is accept changes from both branches by editing the conflicting code snippet in the file as follows:

```C
...
              psi_sched_switch(prev, next, !task_on_rq_queued(prev));
              
#ifdef CONFIG_PMCTRACK
              pmcs_save_callback(prev->pmc, cpu);
#endif               
              trace_sched_switch(preempt, prev, next);
...
```

Finally, confirm the changes (add file) and complete the cherry-pick process:

```bash
user@host:~/tutorial/linux-stable-git$ git add kernel/sched/core.c
user@host:~/tutorial/linux-stable-git$ git cherry-pick --continue
...
[pmctrack-linux-v5.7.19 5dac8ab09b93] PMCTrack kernel patch
 Date: Fri Mar 10 12:15:16 2023 +0100
 14 files changed, 552 insertions(+), 3 deletions(-)
 create mode 100644 include/linux/pmctrack.h
 create mode 100644 kernel/pmctrack.c
```

The source code of the new kernel now includes the PMCTrack patch, and is ready to be compiled!

To save the newly created PMCTrack patch for the v5.7.11 in the upper directory, just use `git diff` as follows:

```bash
user@host:~/tutorial/linux-stable-git$ git diff v5.7.19 > ../pmctrack-linux-v5.7.19_x86.patch
```

<!-- NOTES
Tutorial on alderlake

export PS1="user@host\[\033[00m\]:\[\033[01;34m\]\w\[\033[00m\]\$ " 
-->