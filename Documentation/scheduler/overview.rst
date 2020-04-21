.. SPDX-License-Identifier: GPL-2.0+

====================
Scheduler overview
====================

Linux kernel implements priority based scheduling.  Therefore, more
than one process are allowed to run at any given time and each process
is allowed to run as if it were the only process on the system.  The
process scheduler coordinate which process runs when.  In that context,
it has the following tasks:

- share CPU cores equally among all currently running processes
- pick appropriate process to run next if required, considering scheduling
  class/policy and process priorities
- balance processes between multiple cores in SMP systems


Scheduler attempts to be responsive for I/O bound processes and efficient
for CPU bound processes. Scheduler also applies different scheduling
policies for real time and normal processes based on their respective
priorities.  Higher priorities in the kernel have a numerical smaller
value. Real time priorities range from 1 (highest) – 99 whereas normal
priorities range from 100 – 139 (lowest).


Process
=======

Processes are the most fundamental abstraction in a Unix system,
after files.  Its consists of object code, data, resources, and state.

Each process in the system is represented by :c:type:`struct task_struct
<task_struct>`.  Since task_struct data type must be able to capture all
information of a process, it is relatively large. When a process/thread is
created, the kernel allocates a new task_struct for it.  The kernel then
stores this task_struct in a circular linked list call task_list.  Macro
next_task and prev_task allow a process to obtain its next task and
its previous task respectively.  The frequently used fields of the task
struct are:

| *state:*  The running state of the task.  The possible states are:

- TASK_RUNNING:  The task is currently running or in a run queue waiting
  to run.
- TASK_INTERRUPTIBLE:  The task is sleeping waiting for some event to occur.
  This task can be interrupted by signals.  On waking up the task transitions
  to TASK_RUNNING.
- TASK_UNINTERRUPTIBLE:  Similar to TASK_INTERRUPTIBLE but does not wake
  up on signals. Needs an explicit wake-up call to be woken up.  Contributes
  to loadavg.
- __TASK_TRACED: Task is being traced by another task like a debugger.
- __TASK_STOPPED:  Task execution has stopped and not eligible to run.
  SIGSTOP, SIGTSTP etc causes this state.  The task can be continued by
  the signal SIGCONT.
- TASK_PARKED:  State to support kthread parking/unparking.
- TASK_DEAD:  If a task dies, then it sets TASK_DEAD in tsk->state and calls
  schedule one last time.  The schedule call will never return.
- TASK_WAKEKILL:  It works like TASK_UNINTERRUPTIBLE with the bonus that it
  can respond to fatal signals.
- TASK_WAKING:  To handle concurrent waking of the same task for SMP.
  Indicates that someone is already waking the task.
- TASK_NOLOAD:  To be used along with TASK_UNINTERRUPTIBLE to indicate
  an idle task which does not contribute to loadavg.
- TASK_NEW:  Set during fork(), to guarantee that no one will run the task,
  a signal or any other wake event cannot wake it up and insert it on
  the runqueue.

| *exit_state* : The exiting state of the task.  The possible states are:

- EXIT_ZOMBIE:  The task is terminated and waiting for parent to collect
  the exit information of the task.
- EXIT_DEAD:  After collecting the exit information the task is put to
  this state and removed from the system.

| *static_prio:*  Nice value of a task.  The value of this field does
   not change.  Value ranges from -20 to 19.  This value is mapped
   to nice value and used in the scheduler.

| *prio:*  Dynamic priority of a task.  Previously a function of static
   priority and tasks interactivity.  Value not used by CFS scheduler but used
   by the rt scheduler.  Might be boosted by interactivity modifiers.  Changes
   upon fork, setprio syscalls, and whenever the interactivity estimator
   recalculates.

| *normal_prio:* Expected priority of a task.  The value of static_prio
   and normal_prio are the same for non real time processes.  For real time
   processes value of prio is used.

| *rt_priority:* Field used by real time tasks. Real time tasks are
   prioritized based on this value.

| *sched_class:*  Pointer to sched_class CFS structure.

| *sched_entity:*  Pointer to sched_entity CFS structure.

| *policy:*  Value for scheduling policy.  The possible values are:

* SCHED_NORMAL:  Regular tasks use this policy.

* SCHED_BATCH:  Tasks which need to run longer without pre-emption
  use this policy.  Suitable for batch jobs.

* SCHED_IDLE:  Policy used by background tasks.

* SCHED_FIFO & SCHED_RR:  These policies for real time tasks.  Handled
  by real time scheduler.

| *nr_cpus_allowed:*  Bit field containing tasks affinity towards a set of
   cpu cores.  Set using sched_setaffinity() system call.


Thread
=======

From a process context, thread is an execution context or flow of
execution in a process.  Every process consists of at least one thread.
In a multiprocessor system multiple threads in a process enables
parallelism.  Each thread has its own task_struct.  Threads are created
like normal tasks but the clone() is provided with flags that enable
the sharing of resources such as address space ::

	clone(CLONE_VM | CLONE_FS | CLONE_FILES | CLONE_SIGHAND, 0);

The scheduler schedules task_structs. So Linux doesn't differentiate
between thread and process.

The Scheduler Entry Point
=========================

The main scheduler entry point is arch independent schedule() function
defined in kernel/sched.c.  It implements the scheduler and its objective
is to find a process in the runqueue list and then assign the CPU
to it.  It is invoked, directly or in a lazy(deferred) way from many
different places in the kernel.  A lazy invocation does not call the
function by its name, but gives the kernel a hint(by setting a flag)
that the scheduler needs to be called soon.  The need_resched flag is a
message to the kernel that the scheduler should be invoked as soon as
possible because another process deserves to run.

Following are some places that notify the kernel to schedule:

* scheduler_tick() : This function is called on every timer interrupt
  with HZ frequency and calls scheduler on any task that has used up
  its quantum of CPU time.

* Running task goes to sleep state : Right before a task goes to sleep,
  schedule() will be called to pick the next task to run and the change
  its state to either TASK_INTERRUPTIBLE or TASK_UNINTERRUPTIBLE.  For
  instance, prepare_to_wait() is one of the functions that makes the
  task go to the sleep state.

* try_to_wake_up() : This function awakens a sleeping process by setting
  the task’s state to TASK_RUNNING and putting it back on the run
  queue of the local CPU, for example, the function is invoked to wake up
  processes included in a wait queue or to resume execution of processes
  waiting for a signal.

* yield() : A process voluntarily yields the CPU by calling this function,
  it directly calls schedule() but it is strongly recommended not to use it.

* wait_event() : The process is put to sleep (TASK_UNINTERRUPTIBLE)
  until the condition evaluates to true.  The condition is checked each
  time the wait-queue wq is woken up.

* cond_resched() : It gives the scheduler a chance to run a
  higher-priority process.

* cond_resched_lock() :  If a reschedule is pending, drop the given
  lock, call schedule, and on return reacquire the lock.

* do_task_dead() : Changes the the task state to TASK_DEAD and calls
  schedule to pick next task to run.

* preempt_schedule() : The function checks whether local interrupts are
  enabled and the preempt_count field of current is zero; if both
  conditions are true, it invokes schedule() to select another process
  to run.

* preempt_schedule_irq() : It sets the PREEMPT_ACTIVE flag in the
  preempt_count field, temporarily sets the big kernel lock counter
  to -1, enables the local interrupts, and invokes schedule() to
  select another process to run.  When the former process will resume,
  preempt_schedule_irq() restores the previous value of the big kernel
  lock counter, clears the PREEMPT_ACTIVE flag, and disables local
  interrupts.  The schedule() function will continue to be invoked as
  long as the TIF_NEED_RESCHED flag of the current process is set.

Calling functions mentioned above leads to a call to __schedule(), note
that preemption must be disabled before it is called and enabled after
the call using preempt_disable and preempt_enable functions family.


The steps during invocation are:
--------------------------------
1. Disables pre-emption to avoid another task pre-empting the scheduling
   thread as the linux kernel is pre-emptive.
2. Retrieves running queue based on current processor and obtain the
   lock of current rq, to allow only one thread to modify the runqueue
   at a time.
3. Examine the state of the previously executed task when the schedule()
   was called.  If it is not runnable and has not been pre-empted in kernel
   mode, then it should be removed from the runqueue.  However, if it has
   non-blocked pending signals, its state is set to TASK_RUNNING and it
   is left in the runqueue.
4. The next action is to check if any runnable tasks exist in the CPU's
   runqueue.  If not, idle_balance() is called to get some runnable tasks
   from other CPUs.
5. Next the corresponding class is asked to pick the next suitable task
   to be scheduled on the CPU by calling the hook pick_next_task().  This
   is followed by clearing the need_resched flag which might have been
   set previously to invoke the schedule() function call in the first
   place.  pick_next_task() is also implemented in core.c.  It iterates
   through the list of scheduling classes to find the class with the
   highest priority that has a runnable task.  If the class is found,
   the scheduling class hook is called.  Since most tasks are handled
   by the sched_fair class, a short cut to this class is implemented in
   the beginning of the function.
6. schedule() checks if pick_next_task() found a new task or if it picked
   the same task again that was running before.  If the latter is the case,
   no task switch is performed and the current task just keeps running.
   If a new task is found, which is the more likely case, the actual task
   switch is executed by calling context_switch().  Internally,
   context_switch() switches to the new task's memory map and swaps
   register state and stack.
7. To finish up, the runqueue is unlocked and pre-emption is
   re-enabled. In case pre-emption was requested during the time in which
   it was disabled, schedule() is run again right away.

Scheduler State Transition
==========================

A very high level scheduler state transition flow with a few states can
be depicted as follows.

.. ditaa::
   :alt: digraph of Scheduler state transition.

                                       *
                                       |
                                       | task
                                       | forks
                                       v
                        +------------------------------+
                        |           TASK_NEW           |
                        |        (Ready to run)        |
                        +------------------------------+
                                       |
                                       |
   int                                 v
                     +------------------------------------+
                     |            TASK_RUNNING            |
   +---------------> |           (Ready to run)           | <--+
   |                 +------------------------------------+    |
   |                   |                                       |
   |                   | schedule() calls context_switch()     | task is pre-empted
   |                   v                                       |
   |                 +------------------------------------+    |
   |                 |            TASK_RUNNING            |    |
   |                 |             (Running)              | ---+
   | event occurred  +------------------------------------+
   |                   |
   |                   | task needs to wait for event
   |                   v
   |                 +------------------------------------+
   |                 |         TASK_INTERRUPTIBLE         |
   |                 |        TASK_UNINTERRUPTIBLE        |
   +-----------------|           TASK_WAKEKILL            |
                     +------------------------------------+
                                       |
                                       | task exits via do_exit()
                                       v
                        +------------------------------+
                        |          TASK_DEAD           |
                        |         EXIT_ZOMBIE          |
                        +------------------------------+


Scheduler provides trace points tracing all major events of the scheduler.
The tracepoints are defined in ::

  include/trace/events/sched.h

Using these treacepoints it is possible to model the scheduler state transition
in an automata model. The following journal paper discusses such modeling:

Daniel B. de Oliveira, Rômulo S. de Oliveira, Tommaso Cucinotta, **A thread
synchronization model for the PREEMPT_RT Linux kernel**, *Journal of Systems
Architecture*, Volume 107, 2020, 101729, ISSN 1383-7621,
https://doi.org/10.1016/j.sysarc.2020.101729.

To model the scheduler efficiently the system was divided in to generators
and specifications. Some of the generators used were "need_resched",
"sleepable" and "runnable", "thread_context" and "scheduling context".
The specifications are the necessary and sufficient conditions to call
the scheduler.	New trace events were added to specify the generators
and specifications.  In case a kernel event referred to more then one
event,extra fields of the kernel event was used to distinguish between
automation events.  The final model was done parallel composition of all
generators and specifications composed of 15 events, 7 generators and
10 specifications.  This resulted in 149 states and 327 transitions.
