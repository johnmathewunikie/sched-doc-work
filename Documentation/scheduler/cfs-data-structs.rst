.. SPDX-License-Identifier: GPL-2.0+

====================
CFS Data Structures
====================

Main parts of the Linux scheduler are:

**Running Queue:** This is the central data structure of process
scheduling. It manages tasks that are in a runnable state waiting for
a processor. Each CPU has its own run queue. The scheduler picks a task
from the queue and assigns it to the CPU core. The main members of the
:c:type:`struct rq <rq>` are:

:c:member:`nr_running`
    Total number of tasks on the runqueue.

:c:member:`nr_switches`
    Number of context switches.

:c:member:`cfs`
    Running queue structure for cfs scheduler.

:c:member:`rt`
    running queue structure for rt scheduler

:c:member:`next_balance`
    Timestamp to next load balance check.

:c:member:`curr`
    Points to currently running task of this running queue.

:c:member:`idle`
    Points to currently idle task of this running queue.

:c:member:`lock`
    Spin lock of running queue. task_rq_lock() and task_rq_unlock()
    can be used to lock running queue which a specific task runs on.

:c:member:`clock`
    Clock value for the queue set to time now.

Each rq struct points to a cfs_rq struct which represents the rb tree. The
main members of the :c:type:`struct cfs_rq <cfs_rq>` are:

:c:member:`nr_running`
    Number of runnable tasks on this queue.

:c:member:`min_vruntime`
    Smallest vruntime of the queue.

:c:member:`curr`
    scheduling entity of the current process.

:c:member:`next`
    scheduling entity of the next process.

:c:member:`load`
    Cumulative load_weight of tasks for load balancing.  load_weight is
    the encoding of the tasks priority and vruntime. The load of a
    task is the metric indicating the number of cpus needed to make
    satisfactory progress on its job. Load of a task influences the time
    a task spends on the cpu and also helps to estimate the overall cpu
    load which is needed for load balancing.  Priority of the task is not
    enough for the scheduler to estimate the vruntime of a process. So
    priority value must be mapped to the capacity of the standard cpu
    which is done in the array :c:type:`sched_prio_to_weight[]`. The
    array contains mappings for the nice values from -20 to 19. Nice
    value 0 is mapped to 1024.  Each entry advances by ~1.25 which means
    if for every increment in nice value the task gets 10% less cpu and
    vice versa. The load_weight derived is stored in a :c:type:`struct
    load_weight <load_weight>` which contains both the value and its
    inverse. Inverse value enables arithmetic speed up by changing
    divisions in to multiplications. The cfs_rq stores the cumulative
    load_weight of all the tasks in the runqueue.

**Scheduling entity** : Scheduler uses scheduling entities which contain
sufficient information to actually accomplish the scheduling job of a
task or a task-group. The scheduling entity may be a group of tasks or a
single task.  Every task is associated with a sched_entity structure. CFS
adds support for nesting of tasks and task groups. The  main members of
the :c:type:`struct sched_entity <sched_entity>` are :

:c:member:`load`
    load_weight of the scheduling entity. This is different from the
    cfs_rq load. This value is also calculated differently between group
    and task entities.  If group scheduling is enabled the sched_entity
    load is calculated in the `calc_group_shares` else it is
    the maximum allowed load for the task group.

:c:member:`run_node`
    Node of the CFS RB tree.

:c:member:`on_rq`
    If entity is currently on a runqueue.

:c:member:`exec_start`
    Timestamp of a task when it starts running.

:c:member:`sum_exec_runtime`
    To store the time a task has been running in total.

:c:member:`vruntime`
    vruntime of the task explained below.

Few members are added when CFS is enabled.

:c:member:`parent`
    parent of this scheduling entity. Enables hierarchy of scheduling
    entities.

:c:member:`cfs_rq`
    runqueue on which this entity is to be queued.

:c:member:`my_q`
    runqueue owned by this entity/group.

Each scheduling entity may be run from its parents runqueue.  Scheduler
traverses the sched_entity hierarchy to pick the next task to run on
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

**Schedule class:**  It is an extensible hierarchy of scheduler
modules.  The modules encapsulate scheduling policy details.  They are
called from the core code which is independent. Scheduling classes are
implemented through the sched_class structure.	fair_sched_class and
rt_sched_class class are implementations of this class.  The main members
of the :c:type:`struct sched_class <sched_class>` are :

For the fair_sched_class the hooks (implemented as <function name>_fair)
does the following:

:c:member:`enqueue_task`
    Update the fair scheduling stats and puts scheduling entity in to
    rb tree and increments the nr_running variable.

:c:member:`dequeue_task`
    Moves the entity out of the rb tree when entity no longer runnable and
    decrements the nr_running variable. Also update the fair scheduling
    stats.

:c:member:`yield_task`
    Use the buddy mechanism to skip onto the next highest priority se at
    every level in the CFS tree, unless doing so would introduce gross
    unfairness in CPU time distribution.

:c:member:`check_preempt_curr`
    Check whether the task that woke up should pre-empt the running task.

:c:member:`pick_next_task`
    Pick the next eligible task. This may not be the left most task in
    the rbtree. Instead a buddy system is used which provides benefits
    of cache locality and group scheduling.

:c:member:`task_tick`
    Called from scheduler_tick(). Updates the runtime statistics of the
    currently running task and checks if this task needs to be pre-empted.

:c:member:`task_fork`
    scheduler setup for newly forked task.

:c:member:`task_dead`
    A task struct has one reference for the use as "current". If a task
    dies, then it sets TASK_DEAD in tsk->state and calls schedule one
    last time.	The schedule call will never return, and the scheduled
    task must drop that reference.

Kernel forwards the tasks to each class based on the scheduling policy
assigned to each task. Tasks assigned with SCHED_NORMAL, SCHED_IDLE and
SCHED_BATCH go to fair_sched_class and tasks assigned with SCHED_RR and
SCHED_FIFO go to rt_sched_class
