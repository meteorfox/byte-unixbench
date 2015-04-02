# byte-unixbench
Automatically exported from code.google.com/p/byte-unixbench

**UnixBench** is the original BYTE UNIX benchmark suite, updated and revised by many people over the years.

The purpose of UnixBench is to provide a basic indicator of the performance of a Unix-like system; hence, multiple tests are used to test various aspects of the system's performance. These test results are then compared to the scores from a baseline system to produce an index value, which is generally easier to handle than the raw scores. The entire set of index values is then combined to make an overall index for the system.

Some very simple graphics tests are included to measure the 2D and 3D graphics performance of the system.

Multi-CPU systems are handled. If your system has multiple CPUs, the default behaviour is to run the selected tests twice -- once with one copy of each test program running at a time, and once with N copies, where N is the number of CPUs. This is designed to allow you to assess:

- the performance of your system when running a single task
- the performance of your system when running multiple tasks
- the gain from your system's implementation of parallel processing

Do be aware that this is a system benchmark, not a CPU, RAM or disk benchmark. The results will depend not only on your hardware, but on your operating system, libraries, and even compiler.

## Usage
### Running the Tests

All the tests are executed using the "Run" script in the top-level directory.

The simplest way to generate results is with the commmand:
    ./Run

This will run a standard "index" test (see "The BYTE Index" below), and
save the report in the "results" directory, with a filename like
    hostname-2007-09-23-01
An HTML version is also saved.

If you want to generate both the basic system index and the graphics index,
then do:
    ./Run gindex

If your system has more than one CPU, the tests will be run twice -- once
with a single copy of each test running at once, and once with N copies,
where N is the number of CPUs.  Some categories of tests, however (currently
the graphics tests) will only run with a single copy.

Since the tests are based on constant time (variable work), a "system"
run usually takes about 29 minutes; the "graphics" part about 18 minutes.
A "gindex" run on a dual-core machine will do 2 "system" passes (single-
and dual-processing) and one "graphics" run, for a total around one and
a quarter hours.

### Detailed Usage
The Run script takes a number of options which you can use to customise a
test, and you can specify the names of the tests to run.  The full usage
is:

    Run [ -q | -v ] [-i <n> ] [-c <n> [-c <n> ...]] [test ...]

The option flags are:

  -q            Run in quiet mode.
  -v            Run in verbose mode.
  -i <count>    Run <count> iterations for each test -- slower tests
                use <count> / 3, but at least 1.  Defaults to 10 (3 for
                slow tests).
  -c <n>        Run <n> copies of each test in parallel.

The -c option can be given multiple times; for example:

    ./Run -c 1 -c 4

will run a single-streamed pass, then a 4-streamed pass.  Note that some
tests (currently the graphics tests) will only run in a single-streamed pass.

The remaining non-flag arguments are taken to be the names of tests to run.
The default is to run "index".  See "Tests" below.

When running the tests, I do *not* recommend switching to single-user mode
("init 1").  This seems to change the results in ways I don't understand,
and it's not realistic (unless your system will actually be running in this
mode, of course).  However, if using a windowing system, you may want to
switch to a minimal window setup (for example, log in to a "twm" session),
so that randomly-churning background processes don't randomise the results
too much.  This is particularly true for the graphics tests.

## Notes about Multiple CPUs
If your system has multiple CPUs, the default behaviour is to run the selected
tests twice -- once with one copy of each test program running at a time,
and once with N copies, where N is the number of CPUs.  (You can override
this with the "-c" option; see "Detailed Usage" above.)  This is designed to
allow you to assess:

 - the performance of your system when running a single task
 - the performance of your system when running multiple tasks
 - the gain from your system's implementation of parallel processing

The results, however, need to be handled with care.  Here are the results
of two runs on a dual-processor system, one in single-processing mode, one
dual-processing:

```
  Test                    Single     Dual   Gain
  --------------------    ------   ------   ----
  Dhrystone 2              562.5   1110.3    97%
  Double Whetstone         320.0    640.4   100%
  Execl Throughput         450.4    880.3    95%
  File Copy 1024           759.4    595.9   -22%
  File Copy 256            535.8    438.8   -18%
  File Copy 4096          1261.8   1043.4   -17%
  Pipe Throughput          481.0    979.3   104%
  Pipe-based Switching     326.8   1229.0   276%
  Process Creation         917.2   1714.1    87%
  Shell Scripts (1)       1064.9   1566.3    47%
  Shell Scripts (8)       1567.7   1709.9     9%
  System Call Overhead     944.2   1445.5    53%
  --------------------    ------   ------   ----
  Index Score:             678.2   1026.2    51%
```
As expected, the heavily CPU-dependent tasks -- dhrystone, whetstone,
execl, pipe throughput, process creation -- show close to 100% gain when
running 2 copies in parallel.

The Pipe-based Context Switching test measures context switching overhead
by sending messages back and forth between 2 processes.  I don't know why
it shows such a huge gain with 2 copies (ie. 4 processes total) running,
but it seems to be consistent on my system.  I think this may be an issue
with the SMP implementation.

The System Call Overhead shows a lesser gain, presumably because it uses a
lot of CPU time in single-threaded kernel code.  The shell scripts test with
8 concurrent processes shows no gain -- because the test itself runs 8
scripts in parallel, it's already using both CPUs, even when the benchmark
is run in single-stream mode.  The same test with one process per copy
shows a real gain.

The filesystem throughput tests show a loss, instead of a gain, when
multi-processing.  That there's no gain is to be expected, since the tests
are presumably constrained by the throughput of the I/O subsystem and the
disk drive itself; the drop in performance is presumably down to the
increased contention for resources, and perhaps greater disk head movement.

So what tests should you use, how many copies should you run, and how should
you interpret the results?  Well, that's up to you, since it depends on
what it is you're trying to measure.

#### Implementation


The multi-processing mode is implemented at the level of test iterations.
During each iteration of a test, N slave processes are started using fork().
Each of these slaves executes the test program using fork() and exec(),
reads and stores the entire output, times the run, and prints all the
results to a pipe.  The Run script reads the pipes for each of the slaves
in turn to get the results and times.  The scores are added, and the times
averaged.

The result is that each test program has N copies running at once.  They
should all finish at around the same time, since they run for constant time.

If a test program itself starts off K multiple processes (as with the shell8
test), then the effect will be that there are N * K processes running at
once.  This is probably not very useful for testing multi-CPU performance.

## History
**UnixBench** was first started in 1983 at Monash University, as a simple synthetic benchmarking application. It was then taken and expanded by **Byte Magazine**. Linux mods by **Jon Tombs**, and original authors **Ben Smith**, **Rick Grehan**, and **Tom Yager**.The tests compare Unix systems by comparing their results to a set of scores set by running the code on a benchmark system, which is a SPARCstation 20-61 (rated at 10.0).

**David C. Niemi** maintained the program for quite some time, and made some major modifications and updates, and produced UnixBench 4. He later gave the program to **Ian Smith** to maintain. Ian subsequently made some major changes and revised it from version 4 to version 5.

## Included Tests
UnixBench consists of a number of individual tests that are targeted at specific areas. Here is a summary of what each test does:

### Dhrystone
Developed by Reinhold Weicker in 1984. This benchmark is used to measure and compare the performance of computers. The test focuses on string handling, as there are no floating point operations. It is heavily influenced by hardware and software design, compiler and linker options, code optimization, cache memory, wait states, and integer data types.

### Whetstone
This test measures the speed and efficiency of floating-point operations. This test contains several modules that are meant to represent a mix of operations typically performed in scientific applications. A wide variety of C functions including sin, cos, sqrt, exp, and log are used as well as integer and floating-point math operations, array accesses, conditional branches, and procedure calls. This test measure both integer and floating-point arithmetic.

### Execl Throughput
This test measures the number of execl calls that can be performed per second. Execl is part of the exec family of functions that replaces the current process image with a new process image. It and many other similar commands are front ends for the function execve().

### File Copy
This measures the rate at which data can be transferred from one file to another, using various buffer sizes. The file read, write and copy tests capture the number of characters that can be written, read and copied in a specified time (default is 10 seconds).

### Pipe Throughput
A pipe is the simplest form of communication between processes. Pipe throughtput is the number of times (per second) a process can write 512 bytes to a pipe and read them back. The pipe throughput test has no real counterpart in real-world programming.

### Pipe-based Context Switching
This test measures the number of times two processes can exchange an increasing integer through a pipe. The pipe-based context switching test is more like a real-world application. The test program spawns a child process with which it carries on a bi-directional pipe conversation.

### Process Creation
This test measure the number of times a process can fork and reap a child that immediately exits. Process creation refers to actually creating process control blocks and memory allocations for new processes, so this applies directly to memory bandwidth. Typically, this benchmark would be used to compare various implementations of operating system process creation calls.

### Shell Scripts
The shells scripts test measures the number of times per minute a process can start and reap a set of one, two, four and eight concurrent copies of a shell scripts where the shell script applies a series of transofrmation to a data file.

### System Call Overhead
This estimates the cost of entering and leaving the operating system kernel, i.e. the overhead for performing a system call. It consists of a simple program repeatedly calling the getpid (which returns the process id of the calling process) system call. The time to execute such calls is used to estimate the cost of entering and exiting the kernel.

### Graphical Tests
Both 2D and 3D graphical tests are provided; at the moment, the 3D suite in particular is very limited, consisting of the "ubgears" program. These tests are intended to provide a very rough idea of the system's 2D and 3D graphics performance. Bear in mind, of course, that the reported performance will depend not only on hardware, but on whether your system has appropriate drivers for it.
