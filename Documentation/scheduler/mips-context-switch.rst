.. SPDX-License-Identifier: GPL-2.0+

==============================================
MIPS Architecture And Scheduler implementation
==============================================

Multi-threading in MIPS CPUs
-----------------------------
The MIPS architecture defines four coprocessors.

- CP0: supports virtual memory system and exception handling.
- CP1: reserved for the floating point coprocessor, the FPU
- CP2: available for specific implementations.
- CP3: reserved for floating point operations in the release 1 implementation
       of MIPS64.

MIPS32 and MIPS64 architectures provide support for optional components known
as Modules or Application Specific Extensions. The MT module enables the
architecture to support multi-threaded implementations. This includes support
for virtual processors and light weight thread contexts. Implementation of MT
features depends on the individual MIPS cores. The virtual processing element (VPE)
maintains a complete copy of the processor state as seen by the software system
which includes interrupts, register set, and MMU. This enables a single processor
to appear to an SMP operating system like two separate cores if it has 2 VPE's.
For example two separate OS can run on each VPE such as Linux and and an RTOS.

A lighter version of VPE enables threading at the user/application software level.
It is called Thread Context (TC). TC, is the hardware state necessary to support
a thread of execution. This includes a set of general purpose registers (GPRs),
a program counter (PC), and some multiplier and coprocessor state. TC's have
common execution unit. MIPS ISA provides instructions to utilize TC.

The Quality of service block of the MT module allows the allocation of processor
cycles to threads, and sets relative thread priorities. This enables 2 thread
prioritization mechanisms. The user can prioritize one thread over the other as
well as allocate a specific ratio of the cycles to specific threads. These
mechanisms help to allocate bandwidth a set of threads effectively. QoS block
improves system level determinism  and predictability. QosS block can be replaced
by more application specific blocks.

MIPS Context Switch
-------------------

Context switch behavior specific to MIPS begins in the way :c:macro:`switch_to()`
macro is implemented. The main steps in the MIPS implementation of the macro are:

* Handle the FPU affinity management feature . This feature is enabled by the
  :c:macro:`CONFIG_MIPS_MT_FPAFF` at build time The macro checks if the FPU was
  used in the most recent time slice. In case FPU was not used, the restriction of
  having to run on a cpu with FPU is removed.
* For the previous task, disable the fpu and clear the bit indicating the FPU was
  used in this quantum for the task.
* If fpu is enabled in the next task, check FCSR for any unmasked exceptions
  pending, clear them and send a signal.
* if MIPS DSP modules is enabled, save the dsp context of the previous task and
  restore the dsp context of the next task.
* If coprocessor 2 is present set the access allowed field of the coprocessor 2.
* if coprocessor 2 access allowed field was set in previous task, clear it.
* clear the the access allowed field of the coprocessor 2.
* clear the llbit on MIPS release 6 such that instruction eretnc can be used
  unconditionally when returning to userland in entry.S. LLbit is used to specify
  operation for instructions that provide atomic read-modify-write. LLbit is set
  when a linked load occurs and is tested by the conditional store. It is cleared,
  during other CPU operation, when a store to the location would no longer be
  atomic. In particular, it is cleared by exception return instructions.
  eretnc instruction enables to return from interrupt, exception, or error trap
  without clearing the LLbit.
* clear the global variable ll_bit used by mips exception handler.
* write the thread pointer to the mips userlocal register if the cpu supports
  this feature. This register is not interpreted by hardware and can be used to
  share data between privileged and unprivileged software.
* if hardware watchpoint feature is enabled during build the watchpoint registers
  are restored from the next task.
* Finally the mips processor specific implementation of the :c:func:`resume()`
  function is called. It restores the registers of the next task including the
  stack pointer. The implementation is in assembly::

    arch/mips/kernel/r4k_switch.S
