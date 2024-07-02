# Processes and Threads

## 3.1 Introduction
This assignment, “Procs”, provides the basic
building blocks for the Weenix operating system: threads, processes, and synchronization primitives. These objects are implemented fully in kernel code, and interactions with user-level processes and threads will not be possible until you implement virtual memory at the end of the semester. Note that Weenix as it stands is not truly concurrent, i.e. no two threads will be running at the same time. Instead,
threads take turns each occupying some processor time before switching to other threads; you'll see this in action in this assignment!

Much of the code to do basic kernel operations such as memory allocation and page table management
is provided for you. These features are either beyond the scope of the project or
are too time consuming for too little value. While modifications to the support
code are allowed, significant changes to the provided interfaces make errors
far more likely and make it harder to ask for help. However, it is important for
you to be able to take ownership of the code, so by the end of the project,
you should have a strong (high-level) understanding of all the code involved.
High-level familiarity with all the parts of Weenix is critical for the final project, VM, which involves putting final touches on the kernel across many different parts of the codebase.

At the end of this first assignment, the kernel will be able to run several
threads and processes concurrently in kernel mode.
A strong test suite for this project is critical, because bugs in Procs will present in very strange ways if you wait until a later project to write exhaustive tests. For this assignment, test code must be added
directly into the boot sequence for the kernel, but in later assignments there
will be several additional ways to add tests in a cleaner, more modular way.

You'll be writing an implementation of processes (proc.c) and then move onto scheduling and synchronization
primitives (sched.c) for this assignment. Good luck!

## 3.2 Kernel Memory Management
As mentioned above, this has been fully implemented for you, but, because you need to use this code often, take a moment
to look around `kernel/include/mm/kmalloc.h`, `kernel/include/mm/slab.h`
and `kernel/mm/slab.c` to familiarize yourself with the memory management
interface. The functions `kmalloc()` and `kfree()` exist as kernel-space analogs
of the standard C library `malloc()` and `free()` functions. However, these
should be used sparingly (for example, for memory allocation during testing). Most of the kernel memory management you will be doing will make use of the
`slab_allocators`. A `slab_allocator`, as used in Solaris and Linux, works like
a cache for memory chunks of a particular size. This reduces both the loss of
memory through fragmentation and the overhead of manipulating the heap
directly. Refer to `kernel/include/mm/slab.h` for the slab allocation and freeing functions you will be using.

## 3.3 Boot Sequence
Most of the boot sequence is handled by the support code. The last thing the
boot loader does is execute the function `kmain()`, which initializes the support
subsystems (slab allocators, devices, etc.) and a process called the idle process (PID 0). Once the idle process has been initialized, the current process becomes the idle process. We are still in the boot sequence here, which means that we do not have a thread context in which to properly execute anything (note that `curthr` is set to `NULL` in `proc_idleproc_init()`). We cannot block, and we cannot execute user code. The idle process is a special process that is core-specific — this means that it won't be on the list of processes and is only used in `core_switch()` when switching the current (non-idle) thread/process (remember that each Weenix process only ever has 1 thread) to another non-idle thread/process. Feel free to take a look at `core_switch()` in `kernel/proc/sched.c` to understand this better. 

At the end of `kmain()`, `initproc_start()` is called, which creates the first kernel thread and  process (PID 1), schedules it on the kernel's run queue, and switches contexts, so that this initial process can run. The main routine that the initial process executes is called `initproc_run` — right now, it performs some initialization (which will become more relevant in later projects) and returns. This is where you will invoke your test code for this project. In the future, when your operating system is nearly complete, it will execute a binary program in user mode, but for now, be content to put together your own testing system for threads and processes in kernel mode.

## 3.4 Processes and Threads

Weenix is capable of running multiple processes and threads. There are a few
important things to note about the Weenix threading model.
* There is no kernel mode preemption in Weenix (all threads run until they
explicitly yield control of the processor). It is possible (but not required)
to implement user mode preemption. Think about why this is much easier
to do.
* Weenix is only running on one processor, so synchronization primitives
don’t require atomic compare-and-swap instructions or memory barriers.
* Each process only has one thread associated with it, however the threading code is actually structured so that it would be easy to have multiple
threads per process. For example, each process keeps a list of its threads,
even though that list should never have more than one entry. If a function
seems unnecessary to you, think of it in the context of multiple threads per
process. For instance, when exiting a thread, you must alert the process that one of its threads exited, even though each process should only ever
have one thread.
* Think of a process as a collection of some number of threads and some
metadata. Killing a process is equivalent to killing its threads and vice versa.

The lifecycle of threads and processes is relatively straightforward. When
a process is created, a thread should be added to it and the process should be
made runnable by adding its thread to the run queue. If a process exits and
it still has child processes, it should reassign those children to the init process (the first non-idle process, PID 1),
which, after performing some system setup, will sit in a loop collecting orphaned
processes. When a thread attempts to exit the current process, the process must
clean itself up, because each process only has one thread. Once a thread finishes its current task, it makes a call to the process code (`proc_thread_exiting()`) to indicate it is exiting so that the process can do any final cleanup on the thread data structure.

Once its thread has exited, a process can exit and one of its ancestors
(either its parent or the init process) will call `do_waitpid()` and clean it up fully.
The reason for this somewhat odd deallocation system is that some elements
of the process and thread data structures can only be cleaned up from another
thread’s context. You should determine what these elements are—everything else can be cleaned up by the process itself. Also note that processes which are children of the idle process will not be cleaned up
using this method; they are dealt with separately.

Here is a diagram of the life cycle of threads/processes in Weenix: 
![Initial process creation](./figures/initproc.jpeg)
![Life cycle of a thread and process](./figures/life_cycle_updated.jpeg)

As a side note, you will be using linked lists extensively throughout Weenix.
By this point, you may have looked at the code and seen that some data structures contain list objects or link objects. We provide you with a circular, doubly linked list implementation where the links are stored inside of the objects that
are in the list (hence the link fields inside the objects). Most of this list implementation is provided as a set of inline functions which can be found in the `kernel/util/list.c` and a set of macros in `kernel/include/util/list.h`. 

The trickiest part of this assignment is `do_waitpid()`. Although do_waitpid may seem quite conceptually simple, it is very easily 
to accidentally introduce bugs that will pop up much later on, and debugging procs and, say, VM simultaneously is never fun (source:
us). Testing edge cases of do_waitpid, and all other functions for that matter, is critical to help avoid any future catastrophes
in the development of Weenix. Take a look at section 3.8: Testing for some ideas as to what to test.

* `do_waitpid()`: It will block the current process and it will wait for one of its child processes or a specified child process to finish executing. Once a child process has finished executing, its parent process will be notified and eventually the child process will be destroyed and cleaned up, with the child status will be returned optionally.

## 3.5 Scheduler
Once you have created processes and threads, you will need a way to run these
threads and switch between them. In most operating systems, you must worry
about thread priorities or quality of service assurances, which requires wait
queue as well as run queue optimizations. The Weenix scheduler is much simpler - it consists only of first-in-first-out queues, a function to switch between the contexts of
two threads, and a few higher-level functions to abstract away the details of the
queues from the caller. There is also a `core_switch()` function which handles the actual
updating of `curproc` and `curthr` - this is implemented for you, but it's important to
understand at least the basics of what it's doing in order to implement `sched_switch()`
correctly.

In particular, there is a single run queue from which threads are dequeued
(only when the running thread yields control explicitly) and switched onto the
processor. There are also many wait queues in the system, most of which will
be used as a part of some mutex. When a thread reaches the front of its wait
queue, it can be woken up, which requires putting it on the run queue. 
When the thread reaches the head of the run queue, it should then be switched onto the
processor by the thread running the switching routine (think about which thread should be doing the switching).  

Switching between threads is the most tricky code in this area, but again, this part
is implemented in `core_switch()`. It takes the first thread off of the run queue, 
sets the global variables for current thread and current process to
point at the new thread and its process, then switches into the new context, all
with interrupts blocked. 

If you're feeling confused, the uthreads assignment is quite similar in nature to
procs, the difference being that uthreads works with threads and procs works with
processes. Reading over your implementation of the uthreads assignment as well as
checking out lecture notes is quite helpful!

## 3.6 Synchronization Primitives

Since the kernel is multi-threaded, we need some way to ensure that certain
critical paths are not executed simultaneously by multiple threads. We make use of mutexes in Weenix, 
which you can find implemented in `kernel/proc/kmutex.S`.  While they are unnecessary for Procs, they will be very useful in the Drivers, VFS, and S5FS projects, so familiarity with the interface is helpful.

> **Note**: The implementation of kmutex has been provided in assembly form. We will release the full C source code after the `uthreads` project has completed.

## 3.7 Handling Errors

All Weenix system calls, as well as the underlying routines called within system calls, will return NEGATIVE error codes. For example if you are returning an invalid argument error `EINVAL`, make sure that you return `-EINVAL`. (Even though what will eventually be presented to users will be that the current thread's errno is set to the appropriate errno value and -1 is returned from the system calls.)

## 3.8 Testing
It is your responsibility to think of boundary conditions which could potentially cause problems in your kernel. Testing is an important part of software development, and if you cannot demonstrate that your kernel works properly, we will assume that it does not.

We provide a separate file `kernel/test/proctest.c` where one test has already been set up and written for you. You can use `test_assert()` to make assertions in your tests (see definition in `kernel/include/test/usertest.h`). For lack of a better location, you should run all your test code from the init process’s main function (`initproc_run()` in `kernel/main/kmain.c`). In this case, this would mean calling `proctest_main()` in `initproc_run()`.

To be in the best shape possible before moving on, you should be able to test all of the following situations:
* Run several threads and processes concurrently. Devise a way to show
that multiple threads are running and that they are working properly.
* Demonstrate that threads and processes exit cleanly.
* Ensure all the edge cases of `do_waitpid()` work correctly.
* Try exiting your kernel both by running `proc_kill_all()` and by allowing
all threads to terminate normally. 
    * Note that if `proc_kill_all()` is called by the init process, the init process will terminate and therefore the kernel will exit, and so any testing after the call to `proc_kill_all()` will not run. To test `proc_kill_all()` without causing the kernel to exit, call `proc_kill_all()` from some process other than the init process. In that case, all processes EXCEPT the init process should be terminated, so the init process can go on to run subsequent tests.
* Create several child processes and force them to terminate out of order, making sure they are cleaned up properly.
* Cancellation. You can have test threads check their `kt_cancelled` field directly to test if this is working.

Keep in mind that this is not an exhaustive list, but you should certainly be able to demonstrate each of these tests passing by the end of this assignment.

When running your tests in Weenix, if you have any outputs from the debugging print statements, those should be printed. If they are not, make sure you are providing either `DBG_TEST` or `DBG_PRINT` as the debug mode for `dbg` (the two debug modes enabled by default in the stencil - check out `kernel/util/debug.c` and [[Debugging]] for more information). 

After all the tests are executed, Weenix should remain sleeping forever—this will change in the next project, Drivers. This is what your output should look like, before adding any additional tests:
<img width="1217" alt="weenix-procs-tests-expected" src="https://github.com/brown-cs1690/handout/assets/87140171/0f51cb17-7f59-4b58-aebb-d7344684dbf8">

## 3.9 Frequently Asked Questions

- **If Weenix is a non-preemptive kernel, what are functions like `preemption_enable()` and `preemption_disable()` for?**
    - The codebase for Weenix actually does provide for a preemptive kernel, but we're not requiring you to implement the kernel in this way.

- **Weenix is shutting down early. Why?**
    - Make sure you have a while loop in `initproc_run()` that waits (`do_waitpid()`) until there are no more child processes before returning.

- **After implementing Procs, what should I expect to see when I run Weenix?**
    - Until you start Drivers, halting cleanly will just result in a black screen (unless there is output from your tests in `initproc_run()`) and sleeping forever.

- **How should I debug my implementation?**
    - See the [[Debugging]] page for lots of info on various strategies for debugging in Weenix!

- **Which process is PID 0? PID 1?**
    - The idle process (which waits in a loop to perform context switches between other threads) has PID 0. The init process (which performs some setup and then waits in a loop to collect orphaned processes) has PID 1.

- **I’m feeling overwhelmed by the codebase—where should I start?**
    - We recommend the following:
        - Read this entire handout!(!!)
        - Read the gear up slides - find them [on the course schedule](https://docs.google.com/spreadsheets/d/1bWh8gbxZwjx-SoJxS3Q59kHvMlxxqGakrb6RXjO0xxw)
        - Check out all of the provided support code referenced by the handout - you don’t need to understand it deeply yet, but you’ll be using it a lot, and it will be hard to write any code of your own without a basic idea of what’s already done for you. Especially important are 
            - `kernel/include/mm/slab.h` 
            - `kernel/include/util/list.h`
            - `kernel/include/util/debug.h`
        - Check out the following sections of the Weenix Wiki - [[Tools]], [[Weenix Operating System]], [[Getting Started with Weenix]], and [[Debugging]]. Not everything in Debugging is relevant right now, but you should read enough of these sections to get what you need for Procs, and to reinforce that they’ll be useful as references later on.
        - Now, after all that reading, open up the Weenix source code, and run `make nyi` to see a list of the functions you need to implement for Procs.
        - Note that the Weenix source is organized by project, so (until VM) you won’t need to jump between dozens of files at a time while writing code.
        - Read through the files where you’ll need to write code for Procs, and their corresponding header files. (For a `.c` file named `kernel/project/file.c`, the corresponding header is usually `kernel/include/project/file.h`)
        - Often, the files with the smallest number of functions to implement are a good place to start writing code, so for Procs, try filling out the functions in `kernel/proc/kthread.c`.
        - Cheers!
