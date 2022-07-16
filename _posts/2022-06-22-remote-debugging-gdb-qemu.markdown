---
layout: post
title: "Setting up gdb and qemu for cross-platform debugging"
tags: debugging cross-platform qemu risc-v
category: systems
---
![blurred photo of source code](/assets/debugging.png)

## what this is all about
Okay hear me out. I recently got myself some fresh new tools for doing the
learning thing, as you usually do in this business.
And this time it was [risc-v](https://en.wikipedia.org/wiki/RISC-V) build tools 
for cross compiling code on my 
[x86_64](https://en.wikipedia.org/wiki/X86-64) virtual machine running debian 9.

And look, I don't even want to talk about the trouble I had to go through to make this happen,
so I will just let that slide. My vm is of the x86_64 architecture, meaning 
 I get to only run programs compiled for x86 architecture on it
(well not entirely true but I'm ready to let this also slide for now), but I want to
run `riscv64` binaries on my x86_64 machine because I understand and can write it,
well this is an over statement.

All things been equal I understand risc-v assembly better than the others as 
is supposed to be. If you've ever spent a day writing code you know they rarely if ever work
and if they do they don't work as intended.
It is this problem that has made it a necessity for me to learn to debug  code
with a debugger and what better tool to use than the hardest and most obscure one available to mankind... 
[*gdb*](https://en.wikipedia.org/wiki/GNU_Debugger)

The only problem I have learning to debug code compiled to riscv64 executable binaries
on an x86_64 machine is that, x86_64 and riscv64 are two *completely* different
architectures and they kind of sort of don't see eye to eye. 
But I want to compile, run and debug riscv64 and nobody is stopping me so why not?

## crossing the platforms
As is the case, I am running code in an x86_64 vm, but I really want to do the nifty stuff in risc-v.
I do not want to be seeing x86_64 assembly in my life, I have been traumatized enough.
I just want to lie down, sip my gari, run and disassemble risc-v binaries.

Remember I said something along the lines of you can only run binaries of the same architecture as that of the
machine, yeah that was not entirely right. Because the
linux kernel has something called [`binfmt_misc`](https://en.wikipedia.org/wiki/Binfmt_misc) which allows you to register 
interpreters for other cpu architectures in the kernel, allowing you to execute binaries
of non-native cpu architectures just as you would native binaries.
Lots of [*brazy*](https://www.dictionary.com/e/slang/goin-brazy/) stuff going on in the kernel really.

So how do you even get the interpreters for the different architectures in the
first place. Well you do that by installing an emulator or interpreter in my
case [`qemu`](https://www.qemu.org/).

If qemu is installed from a distro's repositories, the binfmt_misc thing get's
setup by default. But if you compile from source, well it is assumed you know what you are
doing so you get to configure the whole stuff yourself. The funny thing is I
don't know what I'm doing but I compiled qemu from source 
so lets look into how I added binfmt support for my kernel.

We need full root for this. To check if binfmt is mounted in the kernel, we
execute the following command
```
# grep binfmt /proc/mounts
```

then we mount it if its not already mounted with this command
```
# mount binfmt_misc -t /proc/sys/fs/binfmt_misc binfmt_misc rw,relatime 0 0
```

Enabling and disabling binfmt_misc is as easy as

enable binfmt
```
# echo 1 > /proc/sys/fs/binfmt_misc/status 
```

disable binfmt
```
# echo 0 > /proc/sys/fs/binfmt_misc/status 
```

### registering the interpreter(s)
If you compile qemu from source, you will have in the qemu source file directory
a directory named `scripts`, inside this directory is a file named
`qemu-binfmt-conf.sh`, you can't miss it. This file allows you to configure
binfmt_misc to use the qemu emulator(s) you have compiled and installed as
interpreters for non-native executable binaries.
Assuming I installed the qemu binaries into the `/usr/local/bin`
directory. I will have to add binfmt support with the following command
```
# ./qemu-binfmt-conf.sh --qemu-path /user/local/bin --persistent yes
```
the `--persistent` option is well to make it persistent so you don't have to set
it up everything you restart your machine.

### executing cross compiled binaries
binfmt_misc is now enabled and we have registered couple of qemu interpreters
for non-native architectures.
Let's move on to compiling the code and executing it.
```c
#include <stdio.h>

void main(void)
{
        printf("Hello, riscv64 on x86_64!")
}
```

```
joe@debian:../intro $ make
riscv64-linux-gnu-gcc -static -ggdb -g -O0 -c -o main.o main.c
riscv64-linux-gnu-gcc -static -ggdb -g -O0 -o exe main.o
```

```
joe@debian:../intro $ file exe
exe: ELF 64-bit LSB executable, UCB RISC-V, version 1 (SYSV), statically linked, BuildID[sha1]=9f8f41e415dad7b519556c738e6336103f8acbd8, for GNU/Linux 4.15.0, with debug_info, not stripped
```

Okay that checks out, we have a simple risc-v executable binary. Let's check the cpu
architecture of the machine we are about to run the binary on
```
joe@debian:../intro $ uname -m
x86_64
```

Let's give it a go, see if it runs
```
joe@debian:../intro $ ./exe
Hello, riscv64 on x86_64!
```
Ayyy! it works! this is what I love to see

Seriously Joe, this is cool and all but we still haven't done any gdb
debugging yet, I thought that was all this post was about.

## whiping out gdb(-multiarch)
I have a *cross compiled* executable binary that
I can execute on a machine with an incompatible architecture, which is crazy
when you think about it, really think about it for a second. 
All that is left to do is setup the debugger so I can
debug when the problems start popping their heads up.
I have installed on my machine `gdb-multiarch` which is recommended for cross 
platform debugging. Let's whip it out and see what we can accomplish
```
joe@debian:../intro $ gdb-multiarch ./exe
... ( bunch of text I'm not interested in )
Reading symbols from ./exe
(gdb)
```

Cool stuff! we have successfully loaded our binary file into gdb, let me now try
to run it, see if it works
```
(gdb) run
Starting program: /home/joe/dev/c/nix-sys-programming/chap-two/gdb-debugin/intro/exe
/build/gdb-Nav6Es/gdb-10.1/gdb/i387-tdep.c:592: internal-error: void i387_supply_fxsave(regcache*, int, const void*): Assertion `tdep->st0_regnum >= I386_ST0_REGNUM' failed.
A problem internal to GDB has been detected,
further debugging may prove unreliable.
Quit this debugging session? (y or n)
```
hmm this does not look right and I don't know the first thing about what that error
even means. Let me skip around the internet, see if I find something.

## remote is the right way or so I'm told
I still can't figure out what to do with that error so I gave up on it, and then
found a better way to do the debugging using remote targets in gdb.
Turns out you can spin up a gdb server, connect your local debugger to it, and
do the debugging, it's a whole thing.

After looking around the internet for sometime, I found the correct way to do it
with qemu, qemu rules you know. I can setup a qemu gdb server thingy, connect
to it with gdb-multiarch and then I can finally debug the executable binary, as
i've been wanting to do for the past 4 hours :(

Let's look into how it is supposed to be done. First I am supposed start
the qemu user emulator for the architecture I want to run in this case riscv64
on a port of my choosing
```
joe@debian:../intro $ qemu-riscv64 -g 2022 exe

```
There is no output but it should be okay. I think it is running.

Next we start gdb and connect to the server
```
joe@debian:../intro $ gdb-multiarch
... ( bunch of text I'm not interested in )
(gdb)
```

Cool cool cool. connect to the server... *sighs*
```
(gdb) target remote :2022
Remote debugging using :2022
warning: No executable has been specified and target does not support
determining executable automatically.  Try using the "file" command.
0x0000000000010538 in ?? ()
(gdb)
```

Okay things are sort of okay. But gdb says, I have not specified any executable so
let me go and specify one for gdb
```
(gdb) file exe
A program is being debugged already.
Are you sure you want to change the file? (y or n) y
Reading symbols from exe...
(gdb)
```
this output from gdb is confusing. It said I did not specify any
executables but I just connected to a server hosting the file I wanted to debug.
I don't want to change any files gdb, all I want to do is the debugging.

Let me try running it this time and see if it works. I'll set a breakpoint at
`main` and then continue to execute the binary.
```
(gdb) break main
Breakpoint 1 at 0x10622: file main.c, line 5.
(gdb) continue
Continuing.

Breakpoint 1, main () at main.c:5
5               printf("Hello, riscv64 on x86_64!\n");
(gdb)
```
Wuuhhhh! it works. 

Let me finish it off
```
(gdb) continue
Continuing.
[Inferior 1 (process 1) exited normally]
(gdb)
```
everything works fine now. 

Okay one last thing. Let me try to disassemble main and see if it also works.
```
(gdb) disas main
Dump of assembler code for function main:
   0x000000000001061a <+0>:     addi    sp,sp,-16
   0x000000000001061c <+2>:     sd      ra,8(sp)
   0x000000000001061e <+4>:     sd      s0,0(sp)
   0x0000000000010620 <+6>:     addi    s0,sp,16
   0x0000000000010622 <+8>:     auipc   a0,0x3e
   0x0000000000010626 <+12>:    addi    a0,a0,1150 # 0x4eaa0
   0x000000000001062a <+16>:    jal     ra,0x15710 <puts>
   0x000000000001062e <+20>:    li      a5,0
   0x0000000000010630 <+22>:    mv      a0,a5
   0x0000000000010632 <+24>:    ld      ra,8(sp)
   0x0000000000010634 <+26>:    ld      s0,0(sp)
   0x0000000000010636 <+28>:    addi    sp,sp,16
   0x0000000000010638 <+30>:    ret
End of assembler dump.
(gdb)
```
yeah! all is well. I can concentrate on learning how to actually
do the debugging. 

Okay one last thing before I go.

## automating all the things
This is fun and all that but I am learning and I don't want to be manually doing
these stuff all the time. I only want to write code, compile, whip out my
debugger and start the debugging. And so I wrote a little makefile to take care
the manual soul crashing work for me.
```make
# Makefile
cc = riscv64-linux-gnu-gcc
cflags = -static -ggdb -g -O0
port = 2022

exe: $(patsubst %.c, %.o, $(wildcard *.c))
        $(cc) $(cflags) -o $@ $^

%.o: %.c
        $(cc) $(cflags) -c -o $@ $^

# calling `make debug` from the shell will compile you c code, start the server and
# connect to it automatically making the debugging breezy...
debug: exe
        @qemu-riscv64 -g $(port) $< &
        @gdb-multiarch $< -iex "target remote :$(port)"

clean:
        rm *.o exe
```
now I can concentrate all my energies on writing terrible code and trying to
debug it. But first...

Let's put this to the test by creating a small c program and using the `Makefile`
to compile and load the binary into the debugger.
```c
/* main.c */

void main(void)
{
        int arr[] = {1, 2, 3};
        return 0;
}
```

```
joe@debian:../intro $ make debug
... ( bunch of stuff we don't care about )

0x0000000000010538 in ?? ()
Reading symbols from exe...
(gdb) break main
Breakpoint 1 at 0x10620: file main.c, line 3.
(gdb)
```

Let's continue running the program until after the array `arr` is declared
and inspect the variable from gdb
```
(gdb) continue
Continuing.

Breakpoint 1, main () at main.c:3
3               int arr[] = {1, 2, 3};
(gdb) next
4               return 0;
(gdb) print arr
$1 = {1, 2, 3}
(gdb)
```
Ayyyyyyyy!!!!!!!!. See?? It all works now.

## final words
I am done setting things up, I really did not see myself spending the better part of my
day doing this, but it was good fun you know.
I am currently using the following resource to learn debugging, c and risc-v
- [Learning C with gdb](https://recurse.com/blog/5-learning-c-with-gdb)
- [GWU OS Resources](https://github.com/gwu-cs-os/resources)
- [Makefile: 95% of what you need to know](https://www.youtube.com/watch?v=DtGrdB8wQ_8&t=366s)
- [RISC-V Specs](https://riscv.org/technical/specifications)
