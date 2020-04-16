.. SPDX-License-Identifier: GPL-2.0+

X86 Context Switch
------------------

The x86 architecture context switching logic is as follows.
After the switching of MM in the scheduler context_switch() the call
to the x86 implementation of :c:macro:`switch_to()`
is made.  For x86 arch it is located at ::

    arch/x86/include/asm/switch_to.h

Since 4.9, switch_to() has been split into two parts: a
`prepare_switch_to()` macro and the inline assembly portion of
has been moved to an actual assembly file ::

    arch/x86/entry/entry_64.S

* There is still a C portion of the switch which occurs via a jump in
  the middle of the assembly code.  The source is located in
  `arch/x86/kernel/process_64.c` since 2.6.24

The main function of the prepare_switch_to() is to handle the case
when stack uses virtual memory.  This is configured at build time and
is mostly enable in most modern distributions.  This function accesses
the stack pointer to prevent a double fault.Switching to a stack that
has top-level paging entry that is not present in the current MM will
result in a page fault which will be promoted to double fault and the
result is a panic. So it is necessary to probe the stack now so that
the vmalloc_fault can fix the page tables.

The main steps of the inline assembly function __switch_to_asm() are:

* store the callee saved registers to the old stack which will be switched
  away from
* swap the stack pointers between the old and the new task
* move the stack canary value to the current cpu's interrupt stack
* if return trampoline is enabled, overwrite all entries in the RSB on
  exiting a guest, to prevent malicious branch target predictions from
  affecting the host kernel
* restore all registers from the new stack previously pushed in reverse
  order

The main steps of the c function __switch_to() which the assembly
code jumps to is as follows:

* retrieve the thread :c:type:`struct thread_struct <thread_struct>`
  and fpu :c:type:`struct fpu <fpu>` structs from the next and previous
  tasks
* get the current cpu TSS :c:type:`struct tss_struct <tss_struct>`
* save the current FPU state while on the old task
* store the FS and GS segment registers before changing the thread local
  storage
* reload the GDT for the new tasks TLS
* save the ES and DS segments of the previous task and load the same from
  the nest task
* load the FS and GS segment registers
* update the current task of the cpu
* update the top of stack pointer for the CPU for entry trampoline
* initialize FPU state for next task
* set sp0 to point to the entry trampoline stack
* call _switch_to_xtra() to  handles debug registers, i/o
  bitmaps and speculation mitigation
* write the task's CLOSid/RMID to IA32_PQR_MSR
