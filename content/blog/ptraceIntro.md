+++
date = 2018-06-19
title = "Ptrace and You"
description = "Ptrace Tutorial Introduction"
+++

## Ptrace & You: A comprehensive overview of ptrace

Our current project involves heavy use of `ptrace`. While `man 2 ptrace` offers
an in depth technical description, it is very difficult to follow for beginners (hence all
the introduction to `ptrace` blogs around). While these blogs helped me get started, I feel
there is a lack of resources for in-depth, comprehensive view of `ptrace` and it's many features
and pitfalls.

After a year of using `ptrace`, I hope to have something useful to say for newcomers
and those trying break through the description on the man page. Of course, one post is not
enough to cover even the main functionality of `ptrace`. So this is a multi-part series on
`ptrace`!

For another point of view, I found these blogs to be excellent:
- [Write you an strace in 70 lines of code](https://blog.nelhage.com/2010/08/write-yourself-an-strace-in-70-lines-of-code/)
- [Adventures into ptrace(2) Hell](https://www.cyphar.com/blog/post/20160703-remainroot-ptrace-hell)

### What is ptrace?
Ptrace is a Linux system call which allows a process (the _tracer_) to trace
another process (the _tracee_) through `ptrace`, the tracer can intercept events in the
_tracee_ such as: system calls, signals, execves, and more.

The _tracer_ can also read and write to arbitrary locations in: program memory, or CPU
registers, of the _tracee_. This functionality is extremely powerful, programs such as
`gdb` and `strace` are powered by `ptrace`!

### Ptrace at the kernel level
From the `ptrace` [man page](https://linux.die.net/man/2/ptrace),
"The ptrace API (ab)uses the standard UNIX parent/child  signaling  over  waitpid(2)."

Ptrace, in some ways, replaces the _tracee_'s true parent with our _tracer_. When one
of many ptrace-events happens, the kernel puts the _tracee_ in a stopped state, forwards
the message to the _tracer_, the tracer may perform arbitrary computation here. The
_tracer_ then sends one of several `ptrace` continue events through a `ptrace` system call,
the kernel allows the stopped _tracee_ to run once more.

### Ptrace overhead

Ptrace is not known to be fast, and incurs significant overhead. Surprisingly,
I have not been able to find any report/publication quantifying the overhead of `ptrace`.
However, a very rough test, running a simple for loop _n_ times (for a sufficiently
large _n_), doing a trivial system call, like `getpid()`:

```C
#include <unistd.h>
#include <stdio.h>
#include <stdint.h>
#include <sys/syscall.h>

int main(){
  for(int i = 0; i < 10000000; i++){
    int res = syscall(SYS_getpid);
  }

  return 0;
}
```

On my laptop, this program takes 0.82 seconds of user time (averaged from 3 runs using the `time`
command). While ptraced, it takes about 13.18 seconds, a 15x slowdown. Note this
is a worst case scenario when a program continuously calls many system calls. `getpid()`
is one the faster system call, so the time spent in the kernel is dwarfed by the
context switching overhead.

The overhead comes from the number of context switches required to
intercept an event. Consider a system call event, usually this is requires two context
switches: userspace to kernelspace, and back to userspace. With `ptrace` this is doubled:
_tracee_ to kernel, kernel to _tracer_, _tracer_ to kernel, and kernel to _tracee_. To add
to the cost, system calls are intercepted once before the system call is executed in the
_pre-hook_ and once after the system call, in the _post-hook_. Thus we can see where the
overhead comes from. (In a future blog, we will see how using `seccomp + bpf + ptrace`, we
can mitigate this overhead, and the number of context switches).

### Using ptrace

Let's have a look at the `ptrace` system call API:

```C
long ptrace(enum __ptrace_request request, pid_t pid, void *addr, void *data);
```

As we will see, a lot of functionality and complexity is packed into this API.

- `request` refers to the type of event we want from `ptrace`. Some of the more
  common events include PTRACE_PEEKDATA, PTRACE_POKEDATA, PTRACE_GETREGS, PTRACE_SETREGS,
  PTRACE_SETOPTIONS, PTRACE_CONT, PTRACE_SYSCALL, and PTRACE_ATTACH. In the next post,
  we will review these requests.

- `pid` refers to the process idea that this action is for. It must be specified since
  we can be tracing multiple threads or processes from a single tracer.

- `addr` and `data`, these arguments' meaning is context depended based on the `request`.
  `addr` usually refers to a memory location on the tracee's memory space, while `data`
  is some value we wish `ptrace` to use.

In the next section we will use `ptrace` to trace the system calls of an arbitrary process.

### Future Ptrace Blog Posts
In the future I hope to also create blog posts for:
- System call tracing with `ptrace`
- Selectively Tracing System Calls: Using `ptrace` with `seccomp + bpf`
- Speeding up `ptrace` read/writes with `process_vm_read` & `process_vm_write`
- Ptrace with multiprocesses
- Ptrace and signals
- Ptrace with threads
