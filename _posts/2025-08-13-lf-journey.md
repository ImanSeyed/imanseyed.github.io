---
title: "My Linux Kernel Development Journey: From First Patch to Race Condition Hell"
categories: [linux-kernel]
tags: []
---

Over the course of several months in early 2025, I contributed multiple patches to the Linux kernel mainline, focusing primarily on concurrency issues, string safety improvements, and hardware-specific driver fixes. This work involved identifying and resolving deadlocks in CPU frequency management, fixing argument parsing problems in a kbuild script, and investigating complex race conditions reported by Syzbot.

This post shares the technical approaches and tools that helped me contribute to the kernel, hoping it might provide some help to others interested in kernel development.

## Easy Entry Points: String Safety Improvements

### Why String Functions Matter

One of the most approachable ways to contribute to kernel security is fixing unsafe string operations. The difference between `snprintf()` and `scnprintf()` might seem minor, but it's crucial for security:

- `snprintf()` returns the number of characters that *would* have been written (including truncated characters)
- `scnprintf()` returns the number of characters *actually* written

Consider this pattern:
```c
len = snprintf(buf, sizeof(buf), "format %s", user_input);
return len; // This could be larger than sizeof(buf)!
```

Using `scnprintf()` prevents this sort of bugs entirely.

### Finding String Safety Issues

The Linux Kernel Security Project (KSPP) maintains a list of ongoing security hardening efforts. One easy starting point is [removing all strcpy() uses in favor of strscpy()](https://github.com/KSPP/linux/issues/88), which provides a systematic approach to improving string safety across the kernel. Don't do it without understanding the code. These fixes are generally welcomed by subsystem maintainers, but not all maintainers get satisfied by a simple replacement. You might need to come up with a better approach instead.

## Deadlock Hunting: CPU Frequency Management

**WARNING**: Deadlocks aren't the easiest bugs for beginners to fix, avoid them at all cost. Let's say I'm a bit... greedy.

### The Discovery Process

It's worth understanding one of the most powerful tools in kernel debugging: **lockdep**. Lockdep is a runtime lock validator that tracks lock dependencies and detects potential deadlocks before they occur in production.

When you enable `CONFIG_PROVE_LOCKING`, lockdep monitors every lock acquisition and builds a dependency graph. If it detects a scenario where Thread A holds Lock X and wants Lock Y, while Thread B holds Lock Y and wants Lock X, it immediately reports a potential deadlock, even if the actual deadlock hasn't occurred yet.

With lockdep enabled on my development laptop, I encountered this report:
```
WARNING: possible circular locking dependency detected
```

The warning pointed to the CPU frequency subsystem where `store_local_boost()` was acquiring `cpus_read_lock()` while already holding other locks that could create a circular dependency.

### The Evolution of a Fix

```bash
a0982afa0992 cpufreq: drop redundant cpus_read_lock() from store_local_boost()
```

This seemingly simple patch actually went through an interesting evolution that demonstrates an important principle: **sometimes the best solution is to remove code, not add more.**

#### Initial Approach: Fix the Lock Ordering

My first instinct was to fix the circular dependency by acquiring locks in the correct order. The lockdep report showed that `store_local_boost()` was acquiring `cpu_hotplug_lock` after the policy lock, violating the expected hierarchy. So I developed a complex patch that:

1. Acquired `cpu_hotplug_lock` in the store() handler before the policy lock
2. Used `scoped_guard()` with GCC's cleanup attribute for automatic lock cleanup
3. Restructured the locking to respect the proper hierarchy

Here's what that looked like:

```diff
diff --git a/drivers/cpufreq/cpufreq.c b/drivers/cpufreq/cpufreq.c
index 21fa733a2..b349adbeb 100644
--- a/drivers/cpufreq/cpufreq.c
+++ b/drivers/cpufreq/cpufreq.c
@@ -622,10 +622,7 @@ static ssize_t store_local_boost(struct cpufreq_policy *policy,
    if (!policy->boost_supported)
            return -EINVAL;
-   cpus_read_lock();
    ret = policy_set_boost(policy, enable);
-   cpus_read_unlock();
-
    if (!ret)
            return count;
@@ -1006,16 +1003,28 @@ static ssize_t store(struct kobject *kobj, struct attribute *attr,
 {
    struct cpufreq_policy *policy = to_policy(kobj);
    struct freq_attr *fattr = to_attr(attr);
+   int ret = -EBUSY;
    if (!fattr->store)
            return -EIO;
-   guard(cpufreq_policy_write)(policy);
+   /*
+    * store_local_boost() requires cpu_hotplug_lock to be held, and must be
+    * called with that lock acquired *before* taking policy->rwsem to avoid
+    * lock ordering violations.
+    */
+   if (fattr == &local_boost)
+           cpus_read_lock();
-   if (likely(!policy_is_inactive(policy)))
-           return fattr->store(policy, buf, count);
+   scoped_guard(cpufreq_policy_write, policy) {
+           if (likely(!policy_is_inactive(policy)))
+                   ret = fattr->store(policy, buf, count);
+   }
-   return -EBUSY;
+   if (fattr == &local_boost)
+           cpus_read_unlock();
+
+   return ret;
 }

```

#### Learning About Scoped Guards

`guard()` and `scoped_guard()` are neat usages of modern C techniques for resource management. They use GCC's `__attribute__((cleanup))` to automatically call cleanup functions when variables go out of scope:

```c
#define CLASS(_name, var)						\
	class_##_name##_t var __cleanup(class_##_name##_destructor) =	\
		class_##_name##_constructor


#define guard(_name) \
    CLASS(_name, __UNIQUE_ID(guard))

void hmm(void) {
    guard(mutex)(&foo->lock);  /* Automatically unlocked when function exits */
    /* ... critical section code ... */
}


#define __scoped_guard(_name, _label, args...)				\
	for (CLASS(_name, scope)(args);					\
	     __guard_ptr(_name)(&scope) || !__is_cond_ptr(_name);	\
	     ({ goto _label; }))					\
		if (0) {						\
_label:									\
			break;						\
		} else

#define scoped_guard(_name, args...)	\
	__scoped_guard(_name, __UNIQUE_ID(label), args)

/*
 * When 'scope' goes out of scope, the cleanup function automatically
 * releases the lock
 */
```

This pattern prevents lock leaks and makes the code more maintainable by ensuring locks are always released properly. You can use it for memory management as well, like how `systemd` uses it.

#### The Better Solution: Question the Lock's Purpose

During code review, maintainers asked a crucial question: **"Why does this lock exist at all?"**

After some analysis, we discovered that `store_local_boost()` was acquiring `cpu_hotplug_lock` redundantly. The calling context already provided the necessary protection. It was only called from places where protection was already in place.

#### The Final Fix

The final solution was to simply remove the lock acquisition:

```diff
diff --git a/drivers/cpufreq/cpufreq.c b/drivers/cpufreq/cpufreq.c
index 731ecfc17..759dd178a 100644
--- a/drivers/cpufreq/cpufreq.c
+++ b/drivers/cpufreq/cpufreq.c
@@ -622,10 +622,7 @@ static ssize_t store_local_boost(struct cpufreq_policy *policy,
        if (!policy->boost_supported)
                return -EINVAL;
-       cpus_read_lock();
        ret = policy_set_boost(policy, enable);
-       cpus_read_unlock();
-
        if (!ret)
                return count;

```

## Not Every Fix Is a Cure for Cancer

```bash
f757f6011c92 kbuild: fix argument parsing in scripts/config
```

The `scripts/config` script previously assumed that `--file` was always the first argument, which caused issues when it appeared later. This commit updates the parsing logic to scan all arguments for --file and set the config file correctly. It also fixes `--refresh` so that it respects `--file` by passing `KCONFIG_CONFIG=$FN` to make oldconfig (a change that had been marked as a TODO!).

You can look for `TODO`, `FIXME`, and `XXX` comments to address current issues in the kernel, but be aware that not all of them are as straightforward as this one.
## Race Condition Analysis

### Understanding Syzbot

[Syzbot](https://syzkaller.appspot.com/) is Google's automated kernel fuzzing system that continuously tests the Linux kernel by generating random system calls and monitoring for crashes, hangs, or other anomalies. It's an incredible resource for finding real bugs that affect production systems.

### The ZRAM Mystery

One of the most challenging bugs I investigated was [Syzbot report extid+1a281a451fd8c0945d07](https://syzkaller.appspot.com/bug?extid=1a281a451fd8c0945d07), involving a race condition in the ZRAM module.

**The Problem**: During device reset operations, there's a narrow window where the compression algorithm pointer becomes NULL, but concurrent sysfs reads can still access it, causing crashes.

**The Complexity**: The race window logically doesn't even exist (at least that's how it seems) and occurs under lock protection, making it theoretically impossible according to the locking semantics.

### Race Condition Debugging Techniques

For learning about the fundamental concepts behind race conditions, deadlocks, and concurrent programming in systems code, I highly recommend [*Is Parallel Programming Hard, And, If So, What Can You Do About It?*](https://mirrors.edge.kernel.org/pub/linux/kernel/people/paulmck/perfbook/perfbook.html) by Paul McKenney. This book is freely available and covers everything from basic synchronization primitives to advanced lock-free programming techniques used in the kernel.

## Hardware-Specific Contributions

### Community Collaboration

```bash
2b9f84e7dc86 platform/x86: thinkpad_acpi: disable ACPI fan access for T495* and E560
```

Sometimes kernel development is about community collaboration rather than solo debugging. When a friend reported fan control issues on his ThinkPad, I found an existing solution on [Bugzilla](https://bugzilla.kernel.org/show_bug.cgi?id=219643) that needed proper formatting and additional hardware coverage.

This demonstrates another entry point for kernel contributions: taking existing community solutions and helping them reach mainline quality.

## Final Notes

Huge thanks to Shuah Khan for helping me at the Linux Foundation and giving me the courage to actually contribute to the kernel. Also, thank you to the other maintainers and reviewers who were kind enough to keep my sanity during contributions—or strict (especially the inotify maintainers)—from whom I learned a lot.
