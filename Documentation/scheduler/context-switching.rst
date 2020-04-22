.. SPDX-License-Identifier: GPL-2.0+

==========================
Process context switching
==========================

Context Switching
-----------------

Context switching, the switching from a running task to another,
is handled by the context_switch() function defined in
kernel/sched.c.  It is called by __schedule() when a new process has
been selected to run.

 The execution flow is as follows:

* Calls prepare_task_switch() to set up locking and calls architecture
  specific hooks.It must be paired with a subsequent finish_task_switch
  after the context switch.


* Calls macro :c:macro:`arch_start_context_switch()`
  A facility to provide batching of the reload of page tables and other
  process state with the actual context switch code for paravirtualized
  guests.  By convention, only one of the batched update (lazy) modes
  (CPU, MMU) should be active at any given time, entry should never
  be nested, and entry and exits should always be paired.  This is for
  sanity of maintaining and reasoning about the kernel code.  In this
  case, the exit (end of the context switch) is in architecture-specific
  code, and so doesn't need a generic definition.


* The next few steps consist of handling the transfer of real and
  anonymous address spaces between the switching tasks.  Four possible
  context switch types are:

  - kernel task switching to another kernel task
  - user task switching to a kernel task
  - kernel task switching to user task
  - user task switching to user task

For a kernel task switching to kernel task enter_lazy_tlb() is called
which is an architecture specific implementation to handle a context
without an mm.  Architectures implement lazy tricks to minimize tlb
flushes here.  Then the active address space from the previous task is
borrowed (transferred) to the next task.  The active address space of
the previous task is set to NULL.

For a user task switching to kernel task it will have a real address
space.  This address space is pinned by calling mmgrab().  This makes
sure that the address space will not get freed even after the previous
task exits.

For a user task switching to user task the architecture specific
switch_mm_irqs_off() or switch_mm() functions.  The main functionality
of these calls is to switch the address space between the user space
processes.  This includes switching the page table pointers either via
retrieved valid ASID for the process or page mapping in the TLB.

For a kernel task switching to a user task, the context_switch() function
replaces the address space of prev kernel task with the next from the user
task.  Same as for exiting process in this case, the context_switch()
function saves the pointer to the memory descriptor used by prev in the
runqueue’s prev_mm field and resets prev task active address space.

* Next, the prepare_lock_switch() function is called for a lockdep release
  of the runqueue lock to handle the special case of the scheduler
  in which the runqueue lock will be released by the next task.

* Then, the architecture specific implementation of the switch_to()
  function is called to switch the register state and the stack.  This
  involves saving and restoring stack information and the processor
  registers and any other architecture-specific state that must be
  managed and restored on a per-process basis.

* Calls finish_task_switch() to release the spin lock of the runqueue and
  enables the local interrupts.  Then, it checks whether prev is a zombie
  task that is being removed from the system. If so, it invokes
  put_task_struct() to free the process descriptor reference counter and
  drop all remaining references to the process.  It also reconciles locking
  set up by prepare_task_switch() and takes care of any other required
  architecture-specific cleanup actions.

