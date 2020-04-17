.. SPDX-License-Identifier: GPL-2.0+

====================
CFS Data Structures
====================

Main parts of the Linux scheduler are:

**Runqueue:** This is the central data structure of process
scheduling. It keeps track of tasks that are in a runnable state assigned
for a particular processor. Each CPU has its own run queue and stored in a
per CPU array::

    DEFINE_PER_CPU(structrq,runqueues);

Access to the queue requires locking and lock acquire operations must be
ordered by ascending runqueue. Macros for accessing and locking the runqueue
is provided in::

    kernel/sched/sched.h

The runqueue contains scheduling class specific queues and several scheduling
statistics.

**Scheduling entity** : Scheduler uses scheduling entities which contain
sufficient information to actually accomplish the scheduling job of a
task or a task-group. The scheduling entity may be a group of tasks or a
single task.  Every task is associated with a sched_entity structure. CFS
adds support for nesting of tasks and task groups. Each scheduling entity
may be run from its parents runqueue.  Scheduler traverses the
sched_entity hierarchy to pick the next task to run on
the cpu.  The entity gets picked up from the cfs_rq on which it is queued
and its time slice is divided among all the tasks on its my_q.

vruntime is the value by which tasks are ordered on the red-black
tree.  Tasks are arranged in increasing order of vruntime which is
the amount of time a task has spent running on the cpu.vruntime of
a task is updated periodically based on the `scheduler_tick`
function.  scheduler_tick() calls task_tick() hook of CFS.  This hook calls
`task_tick_fair` which internally calls `entity_tick`.
`entity_tick` does two main steps.  First it updates the
runtime statistics of the currently running task. Then it checks if
the current task needs to be pre-empted.  Within `entity_tick`
the `update_curr` is responsible for updating the current task's
runtime statistics including the vruntime.  The function first gets the
scheduled task from the runqueue and the clock value of the main runqueue
: struct rq.  The difference between the start time of the task and the
clock value is calculated and stored in a variable.  Next the vruntime of
the task is calculated in the calc_delta_fair() function.  This function
calls __calc_delta() to calculate the vruntime of the task based on the
formula ::

	vruntime += delta_exec * (NICE_0_LOAD/curr->load.weight);

where:

* delta_exec is the time spent by the task since the last time vruntime
  was updated.
* NICE_0_LOAD is the load of a task with normal priority.
* curr is the shed_entity instance of the cfs_rq struct of the currently
  running task.
* load.weight: sched_entity load_weight.  It is described above.

vruntime progresses slowly for tasks of higher priority. update_curr()
then calls update_min_vruntime() to update the min_vruntime of the
queue.  In `update_min_vruntime` the kernel gets the vruntimes
for leftmost element in the tree  *cfs_rq->rb_leftmost* if it exists and
the scheduled process.  The smallest of the two is chosen.  The maximum
of the current min_vruntime and previously chosen vruntime is taken as
the min_vruntime for the queue to ensure that the the vruntime keeps
increasing and never decreases.  min_vruntime maintains the time of the
task which has run the least on the cpu.  This value is used to compare
against all the tasks in a runqueue.  A task with the least difference
between its vruntime and min_runtime will get the cpu sooner.

After returning from the update_curr() the  entity_tick() then calls
`check_preempt_tick`  to ensure fairness of scheduling.  The vruntime
of the current process is checked against the left most task in the
RB-tree to decide if a task switch is necessary.

**Schedule class:** It is an extensible hierarchy of scheduler modules. The
modules encapsulate scheduling policy details.
They are called from the core code which is independent. Scheduling classes are
implemented through the sched_class structure. dl_sched_class,
fair_sched_class and rt_sched_class class are implementations of this class.

The important methods of scheduler class are:

enqueue_task and dequeue_task
    These functions are used to put and remove tasks from the runqueue
    respectively. The function takes the runqueue, the task which needs to
    be enqueued/dequeued and a bit mask of flags. The main purpose of the
    flags describe why the enqueue or dequeue is being called.
    The different flags used are described in ::

        kernel/sched/sched.h

    enqueue_task and dequeue_task is called for following purposes.
    - When waking a newly created task for the first time. Called with
      ENQUEUE_NOCLOCK
    - When migrating a task from one CPU's runqueue to another. Task will be
      first dequeued from its old runqueue, new cpu will be added to the
      task struct,  runqueue of the new CPU will be retrieved and task is
      then enqueued on this new runqueue.
    - When do_set_cpus_allowed() is called to change a tasks CPU affinity. If
      the task is queued on a runqueue, it is first dequeued with the
      DEQUEUE_SAVE and DEQUEUE_NOCLOCK flags set. The set_cpus_allowed()
      function of the corresponding scheduling class will be called.
      enqueue_task() is then called with ENQUEUE_RESTORE and ENQUEUE_NOCLOCK
      flags set.
    - When changing the priority of a task using rt_mutex_setprio(). This
      function implements the priority inheritance logic of the rt mutex
      code. This function changes the effective priority of a task which may
      inturn change the scheduling class of the task. If so enqueue_task is
      called with flags corresponding to each class.
    - When user changes the nice value of the task. If the task is queued on
      a runqueue, it first needs to be dequeued, then its load weight and
      effective priority needs to be set. Following which the task is
      enqueued with ENQUEUE_RESTORE and ENQUEUE_NOCLOCK flags set.
    - When __sched_setscheduler() is called. This function enables changing
      the scheduling policy and/or RT priority of a thread. If the task is
      on a runqueue, it will be first dequeued, changes will be made and
      then enqueued.
    - When moving tasks between scheduling groups. The runqueue of the tasks
      is changed when moving between groups. For this purpose if the task
      is running on a queue, it is first dequeued with DEQUEUE_SAVE, DEQUEUE_MOVE
      and DEQUEUE_NOCLOCK flags set, followed by which scheduler function to
      change the tsk->se.cfs_rq and tsk->se.parent and then task is enqueued
      on the runqueue with the same flags used in dequeue.

pick_next_task
    Called by __schedule() to pick the next best task to run.
    Scheduling class structure has a pointer pointing to the next scheduling
    class type and each scheduling class is linked using a singly linked list.
    The __schedule() function iterates through the corresponding
    functions of the scheduler classes in priority order to pick up the next
    best task to run. Since tasks belonging to the idle class and fair class
    are frequent, the scheduler optimizes the picking of next task to call
    the pick_next_task_fair() if the previous task was of the similar
    scheduling class.

put_prev_task
    Called by the scheduler when a running task is being taken off a CPU.
    The behavior of this function depends on individual scheduling classes
    and called in the following cases.
    - When do_set_cpus_allowed() is called and if the task is currently running.
    - When scheduler pick_next_task() is called, the put_prev_task() is
      called with the previous task as function argument.
    - When rt_mutex_setprio() is called and if the task is currently running.
    - When user changes the nice value of the task and if the task is
      currently running.
    - When __sched_setscheduler() is called and if the task is
      currently running.
    - When moving tasks between scheduling groups through the sched_move_task()
      and if the task is ćurrently running.

    In CFS class this function is used put the currently running task back
    in to the CFS RB tree. When a task is running it is dequeued from the tree
    This is to prevent redundant enqueue's and dequeue's for updating its
    vruntime. vruntime of tasks on the tree needs to be updated by update_curr
    to keep the tree in sync.
    In DL and RT classes additional tree is maintained for facilitating
    migration between CPUs through push between runqueues. The pervious task
    eligible for pushing if it is active by pushing it to this tree.

set_next_task
    Pairs with the put_prev_task(), this function is called when the next
    task is set to run on the CPU. This function is called in all the places
    where put_prev_task is called to complete the 'change'. Change is defined
    as the following sequence of calls::

         - dequeue task
         - put task
         - change the property
         - enqueue task
         - set task as current task

    It resets the run time statistics for the entity with
    the runqueue clock.
    In case of CFS scheduling class, it will set the pointer to the current
    scheduling entity to the picked task and accounts bandwidth usage on
    the cfs_rq. In addition it will also remove the current entity from the
    CFS runqueue for vruntime update optimization opposite to what was done
    in put_prev_task.
    For the DL and RT classes it will
    - dequeue the picked task from the tree of pushable tasks
    - update the load average in case the previous task belonged to another
      class
    - queues the function to push tasks from current runqueue to other CPUs
      which can preempt and start execution. Balance callback list is used.

task_tick
    Called from scheduler_tick(), hrtick() and sched_tick_remote() to update
    the current task statistics and load averages. Also restarting the HR
    tick timer is done if HR timers are enabled.
    scheduler_tick() runs at 1/HZ and is called from the  timer interrupt
    handler of the Kernel internal timers.
    hrtick() is called from HR Timers to deliver an accurate preemption tick.
    as the regular scheduler tick that runs at 1/HZ can be too coarse when
    nice levels are used.
    sched_tick_remote() Gets called by the offloaded residual 1Hz scheduler
    tick. In order to reduce interruptions to bare metal tasks, it is possible
    to outsource these scheduler ticks to the global workqueue so that a
    housekeeping CPU handles those remotely

select_task_rq
    Called by scheduler to get the CPU to assign a task to and migrating
    tasks between CPUs. Flags describe the reason the function was called.

    Called by try_to_wake_up() with SD_BALANCE_WAKE flag which wakes up a
    sleeping task.
    Called by wake_up_new_task() with SD_BALANCE_FORK flag which wakes up a
    newly forked task.
    Called by sched_exec() wth  SD_BALANCE_EXEC which is called from execv
    syscall.
    DL class decides the CPU on which the task should be woken up based on
    the deadline. and RT class decides based on the RT priority. Fair
    scheduling class     balances load by selecting the idlest CPU in the
    idlest group, or under certain conditions an idle sibling CPU if the
    domain has SD_WAKE_AFFINE set.

balance
    Called by pick_next_task() from scheduler to enable scheduling classes to
    pull tasks from runqueues of other CPUs for balancing task execution
    between the CPUs.

task_fork
    Called from sched_fork() of scheduler which assigns a task to a CPU.
    Fair scheduling class updates runqueue clock, runtime statistics and
    vruntime for the scheduling entity.

yield_task
    Called from SYSCALL sched_yield to yield the CPU to other tasks.
    DL class forces the runtime of the task to zero using a special flag
    and dequeues the task from its trees. RT class requeues the task entities
    to the end of the run list. Fair scheduling class implements the buddy
    mechanism.  This allows skipping onto the next highest priority se at
    every level in the CFS tree, unless doing so would introduce gross
    unfairness in CPU time distribution.

check_preempt_curr
    Check whether the task that woke up should pre-empt the currently
    running task. Called by scheduler,
    - when moving queued task to new runqueue
    - ttwu()
    - when waking up newly created task for the first time.

    DL class compare the deadlines of the tasks and calls scheduler function
    resched_curr() if the premption is needed. In case the deadliined are
    equal migratilbility of the tasks is used a critetia for preemption.
    RT class behaves the same except it uses RT priority for comparison.
    Fair class sets the buddy hints before calling resched_curr() to premempt.


Kernel forwards the tasks to each class based on the scheduling policy assigned
to each task. Tasks assigned with SCHED_NORMAL, SCHED_IDLE and SCHED_BATCH
go to fair_sched_class and tasks assigned with SCHED_RR and SCHED_FIFO go to
rt_sched_class. Tasks assigned with SCHED_DEADLINE policy goes to
dl_sched_class.
