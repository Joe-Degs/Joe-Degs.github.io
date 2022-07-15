---
layout: post
title: "Tracing the exit syscall"
categories: tracing linux debugging
---

---
Yesterday in the afternoon I was coding like a normal person then something happened, 
I encountered a nasty bug (pretty normal for me), turned out to be a feature in
the end but I only found out after spending more that an hour on it. This is
roughly what happened...

The piece of software I was writing was
supposed to get some inputs from the command line and then parse them for use,
 if the user's inputs are not as expected by the program,
the program prints a helpful message and then `exit`s gracefully with a very large exit
status. For brevity I'll show only the exit part of the program in this post.
```c
/* main.c */
#include <stdlib.h>

void main(void) { exit(1000); }
```

Somebody will say Joe, why? why would you do this. Just exit with a small non
zero number like a normal person. As a spokesperson for normal people, `1000` is
our magic number and that is the exit status we want to use today.

I compiled this program, run it and checked the exit code and guess what, the
program did not exit with status `1000`
```
[root@centos tracing-exit]# gcc -o exe main.c
[root@centos tracing-exit]# ./exe
[root@centos tracing-exit]# echo $?
232 
```
What is going on. What happens when I call `exit`. You have my attention linux, what did you do
when I called exit. Tell me, I promise to share it with everybody.

## peeping at syscall exit
To find out what is happening behind the scenes, I decided to pull out the big
guns ftrace and strace utilities for this one. I want to be sure that `exit` gets
called with the status I supplied.

Lets start with ftrace
```
[root@centos tracing-exit]# strace ./exe
execve("./exe", ["./exe"], 0x7ffeea87df60 /* 21 vars */) = 0
... (bunch of stuff I don't care about today)
exit_group(1000)                        = ?
+++ exited with 232 +++
```
Okay, the right status gets passed to the `exit_group` syscall, but it still exits
with `232` which is not what I am expecting. And ohh the syscall that ends up
getting called at execution time is `exit_group` instead of `exit`, lets keep
that in mind.

Lets now use ftrace, see if we get something different.
```
[root@centos tracing-exit]# trace-cmd record -e syscalls -F ./exe
CPU0 data recorded at offset=0x4b0000
    4096 bytes in size
CPU1 data recorded at offset=0x4b1000
    4096 bytes in size
```

Okay let's take a look at the report from the ftrace.
```
[root@centos tracing-exit]# trace-cmd report
cpus=2
... (bunch of stuff I don't care about today)
             exe-1884  [001] 69358.906632: sys_enter_exit_group: error_code: 0x000003e8
```
Here the function `sys_exit_group` is reported to have been called at
runtime with parameter `error_code` given the value of `0x3e8` which corresponds
to the value I passed to `exit`. ftrace uses _enter_ and _exit_ infixes to specify
the entry and exit point of traced functions in trace reports.

Lets try to locate the exact kernel function that gets called when
`sys_exit_group` is called.
```
[root@centos tracing-exit]# trace-cmd list -f sys_exit_group
SyS_exit_group
```
nice!

Now lets get the function call graph of `SyS_exit_group` with depth 4.
```
[root@centos tracing-exit]# trace-cmd record -p function_graph --max-graph-depth 4 \
    -g SyS_exit_group -n rw_verify_area -O nofuncgraph-irqs -e syscalls -F ./exe
  plugin 'function_graph'
CPU0 data recorded at offset=0x4b0000
    8192 bytes in size
CPU1 data recorded at offset=0x4b2000
    4096 bytes in size

[root@centos tracing-exit]# trace-cmd report
... (bunch of stuff I don't care about today)
             exe-7177  [000] 75489.302037: sys_enter_exit_group: error_code: 0x000003e8
             exe-7177  [000] 75489.302041: funcgraph_entry:                   |  SyS_exit_group() {
             exe-7177  [000] 75489.302044: funcgraph_entry:                   |    do_group_exit() {
             exe-7177  [000] 75489.302045: funcgraph_entry:                   |      do_exit() {
             exe-7177  [000] 75489.302046: funcgraph_entry:        0.632 us   |        profile_task_exit();
             exe-7177  [000] 75489.302049: funcgraph_entry:        1.004 us   |        exit_signals();
             exe-7177  [000] 75489.302051: funcgraph_entry:        0.196 us   |        queued_spin_unlock_wait();
             exe-7177  [000] 75489.302052: funcgraph_entry:        1.167 us   |        acct_update_integrals();
             exe-7177  [000] 75489.302055: funcgraph_entry:        0.273 us   |        sync_mm_rss();
             exe-7177  [000] 75489.302057: funcgraph_entry:        0.826 us   |        hrtimer_cancel();
             exe-7177  [000] 75489.302059: funcgraph_entry:        0.323 us   |        exit_itimers();
... (bunch of stuff I don't care about today)
```
Okay, now we see that `SyS_exit_group` calls `do_group_exit` which in turn calls `do_exit`
which from the look of things is the function with most of the exit logic. Enough
looking at trace reports, lets jump into the kernel source and find out what exactly is
going on and why our program is not exiting with status `1000`

## grepping through source code for answers
After using all the tracing tools and techniques I know about, I still don't have
an answer to why a program terminating execution with `exit(1000)` returns `232` 
as its exit code and not `1000`.

Let's specifically take a look at the `do_exit` function definition in the linux kernel release
`3.10.0` (the kernel release on this machine).
```c
/* https://elixir.bootlin.com/linux/v3.10/source/kernel/exit.c#L707 */

void do_exit(long code)
```
 `do_exit` accepts a `long` type which is either 4 or 8 bytes depending on the
the machine's architecture. Either way `1000` should  fit as a value
for the `code` parameter.
So why don't I get a `1000` exit code when the program runs and exits.

Taking a look around the code for a bit I discovered this block of code
```c
/* https://elixir.bootlin.com/linux/v3.10/source/kernel/exit.c#L899 */

SYSCALL_DEFINE1(exit, int, error_code)
{
	do_exit((error_code&0xff)<<8);
}
```
This is the definition of the `exit` syscall. From the code above, the lower 8 bits
of `error_code` are masked off and shifted 8 times to the left.
Lemme crank out python and do some bit maths.
```python
>>> 1000 & 0xff
232
>>> bin(232)
'0b11101000'
>>> 232 << 8
59392
>>> bin(232 << 8)
'0b1110100000000000'
```
The conclusion, exit codes in linux are only 1 byte (0-255),
and if we try to use a number beyond that it gets wrapped around. 
But I'm still a little confused as to why the number passed to `do_exit` is shifted 8
times to the left after masking off. Because following that logic with the `1000` shows
that `59392` gets passed to `do_exit` and not `232`.

## final words
So the fun part about this exercise is, I did some kernel tracing which
is not something I get to do even occassionally. But the more important thing is after all
the tracing and reading of kernel source, I discovered that the behaviour I was
looking for is documented in `man 3 exit`. I wouldn't have done all this if I had 
[RTFM](https://en.wikipedia.org/wiki/RTFM) first. 

### resources
- [ftrace: trace your kernel functions](https://jvns.ca/blog/2017/03/19/getting-started-with-ftrace/)
- [Analyze the linux kernel with ftrace](https://opensource.com/article/21/7/linux-kernel-ftrace)
- [Understanding system calls on linux with strace](https://opensource.com/article/19/10/strace)
- [You can be a kernel hacker](https://www.youtube.com/watch?v=0IQlpFWTFbM&t=1564s)
- [Tracing with ftrace](https://www.youtube.com/watch?v=mlxqpNvfvEQ)
- [Learning the linux kernel with tracing](https://www.youtube.com/watch?v=JRyrhsx-L5Y)
- [See what you computer is doing with ftrace utilities](https://www.youtube.com/watch?v=68osT1soAPM&t=137s)
- [Ftrace documentation](https://docs.kernel.org/trace/ftrace.html)
