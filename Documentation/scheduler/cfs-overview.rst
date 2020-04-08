.. SPDX-License-Identifier: GPL-2.0+

=============
CFS Overview
=============

History
-------

Linux 2.6.23 introduced a modular scheduler core and a Completely
Fair Scheduler (CFS) implemented as a scheduling module. Scheduler
has been improving since kernel version 2.4. In kernel 2.4  there was
one running queue for all processes.  During every schedule the queue
was locked and every task time-slice was update.  This implementation
caused interactivity issues.  A re-write in kernel version 2.5 assigned a
running queue for each processor a running queue and was able to achieve
a O(1) run time irrespective of the number of tasks in the system.  This
scheduler did good until the Rotating Staircase Deadline (RSDL) scheduler
was introduced.  The RDSL scheduler attempted to reduce the scheduler
complexity. The CFS scheduler was inspired from the RDSL scheduler and
the current CFS scheduler originated.  The improvements were not only
in performance and interactivity but also simplicity of the scheduling
logic and modularized scheduler code. Since kernel 2.6.23, there has
been improvements to the CFS scheduler in the areas of optimization,
load balancing and group scheduling features.

RDSL scheduler removed the interactivity estimation code from the
previous linux scheduler.  RDSL was fair by giving equal time slices
to all processes. There was no classification of IO bound or CPU bound
processes.  CFS adopts this concept of fairness.  CFS takes in to use the
length of the sleep time in the interactive process so processes which
sleep less will get more CPU time.

CFS uses a time ordered red-black tree for each CPU. A brief overview
and usage of red-black tree is provided in the following documentation ::

    Documentation/rbtree.txt

Every running process, has a node in the red-black tree. The process at
the left-most position of the red-black tree is the one to be scheduled
next.  The non-left-most lookup is hardly ever done and the left-most
node pointer is always cached.
