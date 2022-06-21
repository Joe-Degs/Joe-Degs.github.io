## cross platform debugging with gdb and qemu
Okay hear me out. I recently got myself some fresh new tools for doing the
learning thing, as you usually do in this business.
And this time it was risc-v build tools for cross compiling c programs in my 
x86_64 virtual machine running debian 9.

And see, I don't even want to talk about the trouble I had to go through to make this happen,
so I will just let that slide. Since my vm is of the x86_64 architecture,
it means I get to only run programs compiled for x86 architecture on it
(wellll not really, but we assume as we always do in the business), and I want to
run risc-v binaries on my x86_64 machine becuase I understand and can write it,
well this is an over statement.
All things been equal I understand risc-v assembly better than the others as 
is supposed to be. If your c programs don't work, as is the case normally, you want
to debug it with a  debugger and what better tool to use than the hardest one
available to mankind.

The only problem I have learning to debug my c programs compiled to elf binaries
on an x86_64 machine is that, x86 and risc-v are two *completely* different
architectures and they kind of sort of don't see eye to eye (insert eye to eye
meme). But I want to and nobody is stopping me so why not?

### crossing the platforms
As is the case, I am running an x86_64 vm, but i really want to do the nifty stuff in risc-v,
I do not want to be seeing x86 assembly in my life, I have been traumatized enough.
I just want to lie down, sip my gari, run and disassemble risc-v binaries.

Remember when I said you get to only binaries of the same architecture as the
machine's they are running on, yeah that was not entirely right. Becuase the
linux kernel has something called `binfmt_misc` which allows you to register 
interpreters for other architectures in the kernel and if you run code of another
architecture it looks through your `binfmt_misc` interpreter set for one to
execute the binary with. Lots of brazy stuff going on in the kernel right?

So how do you even get the interpreters for the different architectures in the
first place. Well you do that by installing an emulator or interpreter in my
case `qemu`(insert link)

If you install `qemu` from your distro's repos, the `binfmt_misc` thing get's
setup as added bonus. And if you don't well it is assumed you know what you are
doing so you get to configure the whole stuff yourself. The funny thing is I
don't know what I'm doing and I compiled qemu from source 
so lets look into how to add the `binfmt_misc` thing to the kernel.
We need full root for this. First you have check if you have `binfmt_misc`
enabled in your kernel.

First we check if we have binfmt mounted ( do the following as the root user )
```sh
grep binfmt /proc/mounts
```

If you don't have binfmt mounted you do it like this
```
# mount binfmt_misc -t /proc/sys/fs/binfmt_misc binfmt_misc rw,relatime 0 0
```

To enable or disable `binfmt_misc` you have  to write a 1 or 0 into the binfmt
status file like this
```sh
echo 1 > /proc/sys/fs/binfmt_misc/status # to enable binfmt

echo 0 > /proc/sys/fs/binfmt_misc/status # to disable binfmt
```

If you compile qemu from source, you have in the qemu source file directory
a directory named `scripts`, inside this directory is a file named
`qemu-binfmt-conf.sh` you can't miss it. This file allows you to configure
`binfmt_misc` to use the qemu interpreters you might have just installed.
Assuming I installed installed the qemu binaries into the `/usr/local/bin`
directory. I add binfmt support with the following command
```sh
./qemu-binfmt-conf.sh --qemu-path /user/local/bin --persistent yes
```
Okay now everything is done let's try it out see if it works as all the internet
blogs are saying.
First let's check if we are truly executing a risc-v binary
```
joe@debian:../intro $ file exe
exe: ELF 64-bit LSB executable, UCB RISC-V, version 1 (SYSV), statically linked, BuildID[sha1]=9f8f41e415dad7b519556c738e6336103f8acbd8, for GNU/Linux 4.15.0, with debug_info, not stripped
```
Okay that checks out, it is really a risc-v binary. Now let's check the cpu
architecture of the machine we are about to run the binary on
```
joe@debian:../intro $ uname -m
x86_64
```

yep it is a x86_64 machine.. sadly
Now let's try running the following c code

```c
#include <stdio.h>

void main(void)
{
        printf("Hello, risc-v on x86!")
}
```

```
joe@debian:../intro $ make
riscv64-linux-gnu-gcc -static -ggdb -g -O0 -c -o main.o main.c
riscv64-linux-gnu-gcc -static -ggdb -g -O0 -o exe main.o
joe@debian:../intro $ ./exe
Hello, risc-v on x86!
```
Ayyy.. this is what I love to see

 I hear you saying, Joe this is cool and all but we still haven't done any gdb
debugging yet, I thought that was all this thing was about. And I hear you, soo
lets dig into that see what we find.

I have a binary that's running, lemme whip out the debugger and start debugging.
I have installed on my machine `gdb-multiarch` which is good for cross 
platform debugging, I guess I am about to find out.
```
joe@debian:../intro $ gdb-multiarch ./exe
... ( bunch of text I'm not interested in )
Reading symbols from ./exe
(gdb)
```

Cool stuff! we have successfully loaded out binary file into gdb, let me run the
file and see if it works.
```
(gdb) run
Starting program: /home/joe/dev/c/nix-sys-programming/chap-two/gdb-debugin/intro/exe
/build/gdb-Nav6Es/gdb-10.1/gdb/i387-tdep.c:592: internal-error: void i387_supply_fxsave(regcache*, int, const void*): Assertion `tdep->st0_regnum >= I386_ST0_REGNUM' failed.
A problem internal to GDB has been detected,
further debugging may prove unreliable.
Quit this debugging session? (y or n)
```

This does not look right and I don't know the first thing about what that error
even means. Let me go and look around on the internet to see what all things
mean and what I have to do next.

## doing it the right way or so I'm told
I still can't figure out what to do with that error so I gave up on it, and then
I found a better way to do the debugging. Using remote targets in gdb.
Turns out you can spin up a gdb server then connect to it as a client and then run
your code, it's a whole thing.

After looking around the internet for sometime, I found the correct way to do it
with `qemu`, qemu rules you know. I can setup a qemu gdb server thingy, connect
to it with  `gdb-multiarch` and then I can finally debug our risc-v code, as
i've been wanting to do for the past 4 hours :(

Let's look into how they are saying I should do it. First I am supposed start
the qemu user emulator for the architecture I want to run in this case `risc-v64`
on a port
```
joe@debian:../intro $ gdb-multiarch qemu-risc-v64 -g 2022

```

There is no output, it should be okay. I think it is running.
Next we start gdb and connect to the port that the server is running at
```
joe@debian:../intro $ gdb-multiarch
... ( bunch of text I'm not interested in )
(gdb)
```

Cool cool cool. Okay lemme connect to the server
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

Okay this output from gdb just confuses me. It said I did not specified any
executables, now I have and you are asking me if I want to change the file.
I don't want to change any files gdb, all I want to do is the debugging.

Let me try running it this time and see if it works. I'll set a breakpoint at
`main` and then continue from there.
```
(gdb) break main
Breakpoint 1 at 0x10622: file main.c, line 5.
(gdb) continue
Continuing.

Breakpoint 1, main () at main.c:5
5               printf("Hello, risc-v on x86!\n");
(gdb)
```
Wuuhhhh! it works. Let me finish it off
```
(gdb) continue
Continuing.
[Inferior 1 (process 1) exited normally]
(gdb)
```

Nothing more to show here. Let me disassemble main, see if it works
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
Ohhh fuck yeah! It works fine now. I can concentrate on learning how to actually
do the debugging.

This is fun and all that but I am learning and I don't want to be manually doing
all this stuff all the time. I only want to write code, compile, whip out my
debugger and start the debugging. I made a makefile that takes care of all the
server and connecting part.
```make
cc = riscv64-linux-gnu-gcc
cflags = -static -ggdb -g -O0

exe: $(patsubst %.c, %.o, $(wildcard *.c))
        $(cc) $(cflags) -o $@ $^

%.o: %.c
        $(cc) $(cflags) -c -o $@ $^

# calling `make debug` from the shell will compile you c code, start the server
# connect to it automatically making the debugging breezy...
debug: exe
        @qemu-riscv64 -g 1234 $^ &
        @gdb-multiarch $< -iex "target remote :1234"

clean:
        rm *.o exe
```

All I have to do now is create a new c source file and then run make debug to
get the program in the debugger and start the debugging
```c
void main(void)
{
        int arr[] = {1, 2, 3};
        return 0;
}
```

Let's compile the file and start the debugging. Let's also set a breakpoint in
the main function
```
joe@debian:../intro $ make debug
... ( bunch of stuff we dont care about )

0x0000000000010538 in ?? ()
Reading symbols from exe...
(gdb) break main
Breakpoint 1 at 0x10620: file main.c, line 3.
(gdb)
```

Now lets continue running the program until after the array `arr` is declared
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
Ayyyyyyyy!!!!!!!!. Yeah this is what I love to see...

Finally I am done setting things up, I really did not see myself spending the better part of my
day doing this. But it was fun. Now if you want to learn the debugging tooj, which
I myself I'm yet to start or risc-v or c. The following resource might help
- [Learning C with gdb](https://recurse.com/blog/5-learning-c-with-gdb)
- [GWU OS Resources](https://github.com/gwu-cs-os/resources)
- [Makefile: 95% of what you need to know](https://www.youtube.com/watch?v=DtGrdB8wQ_8&t=366s)
- [RISC-V](htttps://riscv.org/)
