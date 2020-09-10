# System Calls

As we saw in class, the `strace` utility allows us to trace system calls on running programs. We can learn quite a bit about a program just by inspecting its system calls.

Here are a few example usages of `strace`:

```bash
# Trace a run of 'ls':
$ strace ls

# Trace only file-related system calls
$ strace -e trace=file ls

# Get a nice summary of unique system calls used
$ strace -c ls

# Search for a specific system call (stat in this case):
$ strace ls 2>&1 | grep '^stat'
# Note that we search the start of the string (^) because the system call's
# name comes first, followed by its parameters and return value.
```

When you run a command, such as `strace cat`, each system call will be printed interactively to your terminal. So the general workflow is: run **strace** on a command, which will then print a list of **system calls**. You can run `strace` on any binary file; if you compile your own C code,  `strace a.out` will display the system calls being used by your code (most likely the calls are invoked by the C library, not your code directly).

## Part 1: Tracing System Calls

For the first part of this lab, you will trace several programs and record the results. See detailed instructions below.

1. First, run a trace on `ls`. Record all of the unique system calls (just their names) used by `ls`. To avoid doing a lot of tedious work, you could probably automate most of this with a shell pipeline or command line flag.

(list the syscalls here, use '*' to create a bulleted list in Markdown format)

2. How many unique system calls are in your list?

3. Next, get the examples for `fork` and `readdir` from the schedule page. Go through the code an understand the logic. Then compile them and list their system calls below. Were there any unexpected or interesting results, or any differences in the output?

4. Next, trace several commands you already know and look for new system calls that you haven't seen before (or look up some new commands if you'd like). Describe three system calls below (figure out what they do), and include both the command that generated the system calls as well as the syscall parameters/return values:

syscall 1: 

syscall 2:

syscall 3:

5. For your next mission, you are going to try to predict the purpose of a program simply by inspecting its system calls. Go into the `/bin` directory and look for a program/utility you've never used before. Trace it with `strace` and try to understand what the program is doing without reading its documentation.

(I predict program XYZ is performing task ABC...)

(After reading the documentation for program XYZ, I found its purpose is DEF)

# Part II: Intercepting System Calls

Alright, so we have a somewhat reasonable idea of what system calls are: 
privileged functions that are processed in kernel space. Let's imagine that
we want to add our own system call to Linux. The process involves adding a
new entry to the [system call table](./syscall_32.tbl), which is basically
a big list of **function pointers**. This list maps system calls (represented
internally as integers) to their corresponding function implementations. If
we wanted to add a new system call, we'd simply update the table and provide
an implementation. In the past, system calls could be added or modified at 
run time as loadable **kernel modules**, but newer versions of the Linux
kernel protect the system call table (probably a good idea from a security
standpoint).

In this portion of the lab, we will use dynamic loading to intercept user space
programs' system calls and inject code of our own. This allows us to do many
interesting things. You might be wondering how we can intercept system calls --
after all, they are handled in kernel space. While we could certainly modify
the Linux source code and recompile the kernel, we have another option:
`LD_PRELOAD`.

## LD_PRELOAD

The `LD_PRELOAD` environment variable specifies libraries that should be loaded
before a program starts. We also know that most 'system calls'  we use in
our C programs are actually C library functions that call the 'real'
system calls, so we can exploit this fact to intercept the system calls before
they even happen! Something like:

```
┌─────────────┐             ┌─────────────┐             ┌─────────────┐
│ Application │────────────▶│  C Library  │────────────▶│   Kernel    │
└─────────────┘             └─────────────┘             └─────────────┘

                             -- Becomes --

┌─────────────┐             ┌─────────────┐             ┌─────────────┐
│ Application │──┐       ┌─▶│  C Library  │────────────▶│   Kernel    │
└─────────────┘  │       │  └─────────────┘             └─────────────┘
           ┌─────┘       └─────┐
           │  ┌─────────────┐  │
           └─▶│  Our Code   │──┘
              └─────────────┘
```

To do this, we will build a shared object (.so) file, add it to the `LD_PRELOAD`
environment variable, and then intercept any system calls we are interested in.
In fact, we can intercept any call (we could inject code before `printf`, for
instance, if we wanted to).

### File Logger

For our first trick, let's build a loadable library that logs each file opened
by the user. The extra-studious folks out there are probably already thinking
about intercepting calls to `open()` -- and that's just what we'll do!

`open-log.c` logs each call to `open()`. Pay particular attention to the `dlsym()`
function: it finds the original address of `open()` and saves it so we can still
call the original version of the function once we're done messing with the
implementation of `open()`. We also pass the `RTLD_NEXT` option. From the man
pages:

```
       RTLD_NEXT
              Find the next occurrence of the desired symbol in the search order after the current
              object.   This  allows  one to provide a wrapper around a function in another shared
              object, so that, for example, the definition of a function  in  a  preloaded  shared
              object (see LD_PRELOAD in ld.so(8)) can find and invoke the "real" function provided
              in another shared object (or for that matter, the "next" definition of the  function
              in cases where there are multiple layers of preloading).
```

The weird C incantation near the top of the file creates a function pointer to the "original" `open()`:

```
int (*orig_open)(const char * pathname, int flags) = NULL;
```

It can be called like so:

```
(*orig_open)(pathname, flags);
```

Use the provided Makefile to build open-log.so. Once built, we'll add it to our `LD_PRELOAD`
and try it on a **single** command:

```
$ LD_PRELOAD=$(pwd)/open-log.so uptime
open-log.c:27:initialize(): Original open() location: 0x7f7de12d5470
open-log.c:28:initialize(): New open() location: 0x7f7de15ea1b9
open-log.c:33:open(): Opening file: /proc/uptime
open-log.c:33:open(): Opening file: /proc/loadavg
 23:12:21 up 1 day, 13:01,  9 users,  load average: 0.00, 0.00, 0.00
```

You can also make this change more permanent by exporting `LD_PRELOAD`. Then all following commands
will load your shared library:

```
export LD_PRELOAD=$(pwd)/open-log.so
```

**Hypothetically,** let's say you want to make these changes permanent for every user that logs in. To do so, you would add the following line to `/etc/environment` as root:

```
LD_PRELOAD=/full/path/to/the/shared/object.so
```

...but it probably isn't a good idea unless you **really** want your system calls intercepted all the time.


### Rick Ropen()

Using the code provided for the file logger, modify it so that:

1. If the user tries to open any files that end in `.java`, report that they do not exist. Hint: think about what `open()` should return; you'll also need to set the `errno` (see `man errno.h`)

2. If the user opens a `.txt` file, you should instead redirect the request to open `roll.txt` provided in this repo. In other words, if you have a file named `hello.txt` that contains 'Hello world!', when you open it you will see the contents of `roll.txt` instead. [Some Context](https://en.wikipedia.org/wiki/Rickrolling).


## Finishing Up

Check in your lab notebook (edit this README.md file) and code changes to your repo before the deadline.

