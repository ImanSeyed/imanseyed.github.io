---
title: "The Flow of the Kernel Upon Receiving a SIGSEGV for Null-Dereference"
categories: [linux-kernel]
tags: [c, x86, segmentation-fault, tracing, vma]
---

You might have seen "[\*(char\*)0 = 0; - What Does the C++ Programmer Intend With This Code?](https://www.youtube.com/watch?v=dFIqNZ8VbRY)" where JF Bastien discusses this line of code:

```c
*(char*)0 = 0;
```

JF covers the majority of the topics related to this code, but he stops when he says it crashes in the pagetable lookup (on systems that support it, of course), which makes sense because everything about this area is architecture/hardware dependent and would take decades to cover everything about it, but we have all the time in the world, so why not go deeper?

I want to go over everything I can about `x86_64` (and a little about ancient 32-bit `x86` CPUs) and why we get segmentation faults, so prepare to read some kernel code with me.

## Beware of Undefined Behavior
JF mentions that this is an undefined behavior in C and C++, however, for our scenario, I limited the architecture so we can write it as an inline assembly to be safe from the compiler (not randomly inserting a `ud2`), and have more fun.
```c
int main() {
	asm volatile("movb %%rax, (%%rax)" ::: "memory"); 
}
```
> Don't worry about `rax`'s default value; ABI is on our side. By default, [ELF_PLAT_INIT()](https://elixir.bootlin.com/linux/v6.10/source/arch/x86/include/asm/elf.h#L108) will set it to zero, so (%%rax) is equivalent to null-dereference.
{: .prompt-tip}

## Using Ftrace to Trace the Process
I will make it easier for you by filtering the related functions. This is a brief idea of what we should look for:

```bash
$ sudo trace-cmd record -p function ./ka-boom
$ sudo trace-cmd report [...] | [...]
...
lock_mm_and_find_vma <-- do_user_addr_fault
down_read_trylock <-- lock_mm_and_find_vma
find_vma <-- lock_mm_and_find_vma
...
__bad_area_nosemaphore <-- do_user_addr_fault
force_sig_fault <-- __bad_area_nosemaphore
```

## Big Picture
When the program attempts to access page zero, a page fault occurs which leads us to [`do_user_addr_fault()`](https://elixir.bootlin.com/linux/v6.10/source/arch/x86/mm/fault.c#L1211). This function is only for `x86`, and it checks a variety of things, including permission of the page (`PROT_*`), or wether the address can be reached by the task that caused the page fault. In our situation, we reach to [this piece of code](https://elixir.bootlin.com/linux/v6.10/source/arch/x86/mm/fault.c#L1361):
```c
retry:
	vma = lock_mm_and_find_vma(mm, address, regs); // <=== HERE
	if (unlikely(!vma)) {
		bad_area_nosemaphore(regs, error_code, address);
		return;
	}
```

The `vm_area_struct` represents a collection of page-aligned regions of memory that are *in use* and are tied to one another in terms of protection and purpose. It might be anything, such as a block of anonymous memory allocated with `mmap()` or bytes allocated on the heap using `malloc()`. Do you see the problem? We haven't mapped anything, thus there is no area at all, hence `lock_mm_and_find_vma()` fails to locate the `vma`. `vma` is `NULL`, causing us to call `__bad_area_nosemaphore()` with [`SEGV_MAPERR`](https://elixir.bootlin.com/linux/v6.10/source/include/uapi/asm-generic/siginfo.h#L227) which states the reason for our `SIGSEGV`:
```c
/*
 * SIGSEGV si_codes
 */
#define SEGV_MAPERR	1	/* address not mapped to object */
```
But is this always the case? *NO!*

## Page Zero in SVr4
If you dig deeper into the kernel, you'll find [an interesting block of code](https://elixir.bootlin.com/linux/v6.10/source/fs/binfmt_elf.c#L1276) inside of `load_elf_binary()` that emulates the ABI behavior of previous Linux versions:
```c
if (current->personality & MMAP_PAGE_ZERO) {
  /* [...] */
	error = vm_mmap(NULL, 0, PAGE_SIZE, PROT_READ | PROT_EXEC,
	MAP_FIXED | MAP_PRIVATE, 0);
}
```

`vm_mmap()` allocatets page zero with the `PROT_READ` and `PROT_EXEC` permissions. Based on this code, we can assume that while `current->personality` has `MMAP_PAGE_ZERO`, reading from page zero should not result in a segmentation fault. The question is, when does this `personality` applies? Arch-specific code determines which personality bits to turn on for incoming `exec`s. If you want to dig deep into it, you can look for `SET_PERSONALITY` and `SET_PERSONALITY2` macros in your preferred architecture.

Another thing that might pique your interest is the `PROT_EXEC`. Read-implies-exec behaviour is expected by older software because it is the way hardware was designed to work. Actually, certain 32-bit programs running on `x86_64` and older `x86` 32-bit CPUs still suffer from this. See [this comment](https://elixir.bootlin.com/linux/v6.10/source/arch/x86/include/asm/elf.h#L266) for details.

## Learn More

The Linux kernel's virtual memory management is rather complex in comparison to other operating systems that you may meet, therefore I'd highly recommend reading the useful book below authored by Mel Gorman to understand more. It is pretty old, yet it still significantly speeds up the learning process:
- https://www.kernel.org/doc/gorman/pdf/understand.pdf

