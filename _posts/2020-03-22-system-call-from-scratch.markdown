---
layout: post
title:  "System call from scratch"
description: How to make a system call from simple assembly code
date:   2020-03-22 14:03:36 +0530
categories: linux assembly
---

In this first post we are going to dive into how make a simple system call.
Let's start with some assembly and a simple standard output printer program.

### Legacy system calls

```nasm
        global _start
_start:
        mov     eax, 4	    ; set the system call number
	mov     ebx, 1      ; set file descriptor 1 (stdout)
        mov     ecx, msg    ; set the message reference in rcx register
        mov     edx, 12     ; set 12 (string length) in edx register
        int     0x80        ; trigger an interruption with code 0x80
        mov     rax, 1      ; write 1 in rax register
        int     0x80	    ; trigger an interruption with code 0x80

section .data
msg     db      'First post!',0xa
```

NASM is an assembler for x86 architecture. The following command assemble our code then link it and run it.

```shell
nasm -felf64 first.asm && ld ./first.o && ./a.out
First post!
```

It successfully displays the message but how does it work ?

In order to protect the operating system and the hardware, user programs can not directly access the hardware and some kernel data structure, they need to do it using system calls. In our example the system call is made using the `int` instruction that trigger an interruption with interrupt vector `0x80`.


When the CPU receive this interruption he will look for a handler function in the interrupt descriptor table (IDT). This table associate interruptions/exceptions to a gate which can be used to find the interrupt handler address. The table has a size of 256, there can be at most 256 type of interruption, linux will always use the code [`0x80`](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/irq_vectors.h#L45) for legacy system calls.

When the interruption is made the CPU will jump to kernel mode, and dump the userspace registers. The [handler](https://github.com/torvalds/linux/blob/master/arch/x86/entry/entry_32.S#L1072) of the handler expect the following arguments for the system calls parameters:

```
 * eax  system call number
 * ebx  arg1
 * ecx  arg2
 * edx  arg3
...
```

The register `eax` will store the system call number, `4` in our example.
The handler will then call [`do_int80_syscall_32`](https://github.com/torvalds/linux/blob/master/arch/x86/entry/common.c#L356)

```
__visible void do_int80_syscall_32(struct pt_regs *regs)
{
	enter_from_user_mode();
	local_irq_enable();
	do_syscall_32_irqs_on(regs);
}
```

At this point the code is executed in kernel mode and the regs parameter is a structure containing the userspace registers (ie. the registers set before the `int 0x80` call. Note that the state of a program’s execution is completely represented by CPU register state.

This function will end up doing a lookup in the following [table](https://github.com/torvalds/linux/blob/master/arch/x86/entry/syscalls/syscall_32.tbl).

```
0	i386	restart_syscall		sys_restart_syscall		__ia32_sys_restart_syscall
1	i386	exit			sys_exit			__ia32_sys_exit
2	i386	fork			sys_fork			__ia32_sys_fork
3	i386	read			sys_read			__ia32_sys_read
4	i386	write			sys_write			__ia32_sys_write
```

The system call number 4 is associated with the entry point [`sys_write`](https://github.com/torvalds/linux/blob/master/fs/read_write.c#L620)

```c
SYSCALL_DEFINE3(write, unsigned int, fd, const char __user *, buf,
		size_t, count)
{
	return ksys_write(fd, buf, count);
}
```

This code will effectively write to the standard output our `First post!\n` message.




```nasm
        global _start
_start:
        mov     rax, 1
        mov     rdi, 1
        mov     rsi, message
        mov     rdx, 12
        syscall
        mov     rax, 60
        xor     rdi, rdi
        syscall

section .data  
message db  'First post!',0xa

}
```

#### References: 
- https://lwn.net/Articles/604287/
- https://lwn.net/Articles/604515/
- https://linux-kernel-labs.github.io/refs/heads/master/lectures/interrupts.html

