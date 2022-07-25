---
layout: post
title: "RISC-ing it all: How data is stored in executables"
categories: systems
tags: risc-v assembly systems
---

---
## the beginning
If you are like me, you have written a program, compiled it and run the
executable binary file at some point in your (probably miserable) life.
And then you did it couple of times, then you started wondering, how
does it all work especially the compiling and execution part. 
It bothered you enough to want to learn it (or atleast read about it),
but somebody told you were wasting your time, you would be better off
learning to build web apps because that's where all the money is. And you knew
they were not lying so you
listened to them. You tried to learn
how to write beautiful web apps but realised you didn't care about building beautiful
things as much as you liked learning about how beautiful things get built.

And so you went on the internet and you found blogs
and even big books (very big ones) dedicated to how programs get compiled (or get executed) 
and you tried to read them
but they made absolutely no sense to you. You gave up on the idea, but not
before you learned that programs at the end of the day get executed by
something called the [CPU](https://en.wikipedia.org/wiki/Central_processing_unit) 
and that the CPU performs actions based on something called an
[instruction](https://pclt.sites.yale.edu/cpu-instructions) which is a [bit](https://en.wikipedia.org/wiki/Bit)
pattern and depending on the pattern of the bits it chooses the appropriate course of action. 

You didn't even know what that meant whatsoever
but it felt like a good thing to know so you held onto that piece of knowledge.
And then at another time you read somewhere that all the programs that we
cannot now live without are actually all just bunch of bit patterns that the
CPU thing can execute (bunch of instructions and **data** essentially).
Your mind was blown and your socks were knocked off but you put them back on and
continued reading. You found out that the compiler is responsible for taking
the programs you write and turning them into bit patterns that the CPU
understands and can perform actions based on.

And so it all started for you, you knew then that there was no turning back,
you knew too much to go back now so you went further and discovered
that the type of bit patterns the CPU understands is dictated by something
called the [Instruction Set Architecture](https://en.wikipedia.org/wiki/Instruction_set_architecture)
Which is even harder to understand than everything else because the manuals
explaing what they are and what they do are
big (alot bigger than the others) and you didn't know where to start. But then
you found the precious little ISA ([RISC-V](https://en.wikipedia.org/wiki/RISC-V)),
beautifully crafted in the open with much much smaller manuals, 
so you read, and read and watched videos and continued to read some more. And
everything made sense.
All the stars aligned and the universe made sense, finally the glimmer of light
in the darkness you were hoping for all along.

_Joe, enough! cut the crap and talk about what we are doing here today_

Okay, let's get into why we are here again. We know that when we write code and
compile it, it gets translated into instructions and data that the CPU can understand
and execute. These instructions get stored on disk as an [executable](https://en.wikipedia.org/wiki/Executable)
file. We have a vague idea of the tools and the processes involved in making all
of these things happen and they are:
1. A **text editor** is used to create **source code** in a high level
   language ([*C*](https://en.wikipedia.org/wiki/C_(programming_language)))
   or a low-level language ([*assembly*](https://en.wikipedia.org/wiki/Assembly_language))
   which are just symbolic representations of instructions.
2. The **compiler** or **assembler** takes the source code and turns them into
   [*object*](https://en.wikipedia.org/wiki/Object_file) files or 
   **executable** files.
3. A [*linker*](https://en.wikipedia.org/wiki/Linker_(computing)) takes 
   the object files and (surprise!) links them into one big (or small)
   executable file we can give to the CPU to run.

On our perpetual journey of trying to understand how things work, today we want
find out how data and instructions are stored in our compiled programs. We are
going to be experimenting with alot of RISC-V assembly code, we want to understand how
everything fits together and today we pick data. Without much ado... Welcome to
my party fellas!

## getting object files from our source code
We have establised some common ground in the beginning. Now let's move to
figuring out how we can get instructions and data from out compiled code. And 
possibly inspect them to see what we can find. The program we are using for this
particular task is just a simple C program that does absolutely nothing but
contains data -- data is very important here today.
```c
/* main.c */
int num            = 0;
int number         = 0x1234;
const int constant = 0x4567;

int main(int argc, char *argv[])
{
        return 0;
}
```
Fancy right! Looking sharp doing nothing.

We got some tools to make the work alot easier, and since we are going to be
looking at a lot of RISC-V assembly code, we will want to compile the code
with a RISC-V compiler. The one I have installed is the `riscv64-linux-gnu-gcc`
compiler that can spit out executables and objects files at the speed of
light (or maybe not). So let's get some object files out of the code above.
```
joe@debian:../$ riscv64-linux-gnu-gcc -c -g main.c
joe@debian:../$ ls
main.c  main.o
```
Well what do you know?. We've got ourselves a nice little object file `main.o`.
It looks beautiful doesn't it. But what next? we have out object file we
inspect it to see what we can find inside its nooks and cranies

### inspecting the object file
Alright we have an object file on our hands. Let's check what file format the
object file is in.
```
joe@debian:../$ file main.o
main.o: ELF 64-bit LSB relocatable, UCB RISC-V, version 1 (SYSV), with debug_info, not stripped
```
Well it turns out our object file is an [ELF](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format) 
file. If you use linux you probably
have heard of it before. It is the file format in which executable files, object
files and shared libraries are put in. 
*But why a file format, can't we just stuff all the instrutions and data 
into one big file and forget about having a format for the file?* 
Well, we will come back to this later.

First lets try to make sense out of the output of the `file` command up there.
- **ELF**: this specifies the file format, in this case an ELF file. There are
  many other file formats out there for executable file and what not. And they
  specification documents if you want to murk around with them.
- **LSB**: specifies the [endianness](https://en.wikipedia.org/wiki/Endianness) of the file.
  There are two types for executable files that I care and know about: little endian or
  Least Significant Byte (LSB) and big-endian or Most Significant Byte (MSB). 
  It essentially indicates how bytes are stored in memory, LSB means the least
  significant byte of a word is stored at the lower memory address and the most significant
  byte at the higher memory address and MSB means the most significant byte is 
  stored at the lower memory address and the least significant byte at a higher
  memory address.
- **relocatable**: Since this is an object file, it is not yet executable even
  though it contains instructions and data. It is a *relocatable* file because
  it needs to be linked (by the linker) with other object files to create an executable.
- **UCB RISC-V**: specifies the ISA of the instructions in the file. In this
  case the precious little RISC-V. UCB (UC Berkeley) is where the ISA was developed.
- **version 1 (SYSV)**: This specifies the OS and [ABI](https://en.wikipedia.org/wiki/Application_binary_interface) 
  of the file.
- **with debug_info, not stripped**: We compiled the code with the `-g` which
  embeds snippets of code and names of variables and functions in the resulting
  file, this snippets and symbols are called the `debug_info` and they are not
  stripped from the object file.

## mapping standard data widths in c to risc-v
the basic data types and their respective sizes in c are
- int    -> 2 bytes on 32 bit machines and 4 bytes on 64 bit machines
- char   -> 1 byte
- short  -> 2 bytes
- long   -> 4 bytes on 32 bit machines and 8 bytes on 64 bit machines
- float  -> 4 bytes
- double -> 8 bytes
these data types are the primitive types in c and for portable programs we are
generally advised to use uint8_t, uint16_t, uint32_t, uint64_t, __int128_t from
the `stdint.h` becuase these types unlike the other ones stay the same accross
different architectures. And because the compiler is very smart it figures out
most of this stuff so you don't have to figure them out yourself.
But you know me, I'm hard of hearing and like I'm used to doing, I am going to
peer into the nookies of compiled code to see how this data types map to 
underlying assembly and machine code. the machine architecture i'm using for
this is the sweet sweet RISC-V.

## different sections of compiled code
the different sections of compiled code are
- data (.data, .rodata) -> the data sections contains initialized variables or
  constants (in the case of .rodata section)
- text (.text) -> contains executable instructions as bytes that the cpu can read,
  decode and execute.
- .bss (.bss) -> contains static and global variables that are not initialized.
  this section is just a bunch of zeros when loaded into memory.

- at this stage let's get some code dig into the assembly of it and see how
  c types get mapped to risv assembly.

```c
#include <stdint.h>

/* TODO */
int main(int argc, char argv[]) {
    return 0
}
```

```
$ gcc -c -g main.c
```
over here what we really want to look at is the just the assembly of the code
we have written, so we compile the file into an object file `.o`  which is
just the binary representation of the code in the architecture we are compiling
to and nothing more. and this object file is in the elf file format.
Usually what the compiler does is it turns c code to object files, then it
invokes a linker which takes the object files and combines them into one big
executable binary file (except in the case of dynamic libraries).
I think its called a linker becuase when we write
code we write it in multiple files and most of the time each file cross
references each other.

- Now that the we have an object file lets take a look at the section thing we
  hinted up above. to see the different sections we use the `readelf` tool

```
joe@debian:../$ readelf --sections main.o
There are 20 section headers, starting at offset 0xbd8:

Section Headers:
  [Nr] Name              Type             Address           Offset
       Size              EntSize          Flags  Link  Info  Align
  [ 0]                   NULL             0000000000000000  00000000
       0000000000000000  0000000000000000           0     0     0
  [ 1] .text             PROGBITS         0000000000000000  00000040
       000000000000001a  0000000000000000  AX       0     0     2
  [ 2] .data             PROGBITS         0000000000000000  0000005c
       0000000000000004  0000000000000000  WA       0     0     4
  [ 3] .bss              NOBITS           0000000000000000  00000060
       0000000000000004  0000000000000000  WA       0     0     4
  [ 4] .rodata           PROGBITS         0000000000000000  00000060
       0000000000000004  0000000000000000   A       0     0     4
  [ 5] .debug_info       PROGBITS         0000000000000000  00000064
       00000000000000d0  0000000000000000           0     0     1
  [ 6] .rela.debug_info  RELA             0000000000000000  000007c8
       00000000000001f8  0000000000000018   I      17     5     8
  [ 7] .debug_abbrev     PROGBITS         0000000000000000  00000134

... (bunch of stuff we don't care about)

Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings), I (info),
  L (link order), O (extra OS processing required), G (group), T (TLS),
  C (compressed), x (unknown), o (OS specific), E (exclude),
  p (processor specific)
```
Nice!. Well it turns out there are alot more of sections out there and they all
are important in their own ways. We only care about `.text, .data, .bss,
.rodata`.

- using the readelf tool lets take a look at the relevant sections and see if
  we can understand anything that's going on, we might not and i've made my
  peace with it.
    $ readelf -x .bss main.o
```

```
    $ readelf -x .data main.o
```

```
    $ readelf -x .rodata main.o
```

```
    $ readelf -x .text main.o
```

```
everything from the outputs are just bunch of bytes. even the .text section
which contains executable instructions for the cpu. wild right!

- lets take a look at the disassembly of the data sections (.bss, .rodata,
  .data).
    $ riscv64-linux-gnu-objdump -z -S -D -j .data -j .bss -j .rodata main.o | less
<point out some things>

- and now let's take a look at the dissassembly of of the code (.text) section
    $ riscv64-linux-gnu-objdump -z -S -D -j .text main.o | less
well we compiled a function that did nothing and got back alot of assembly code
equally doing fuck all. but it will be interesting to figure out what all this
stuff mean anyway so let's take a few moments to dig in.
<the stuff going on is alot, probably draw some diagrams and try to make it
visually appealing for people to understand>


## rv-c-data
integers variables in c programming language.
overhere they are being compiled to object files and disassembled to look at
the resulting assembly code that is generated. it is pretty nifty. but we are
taking a look at the data section (.data, .bss) section of the object file.
riscv is little endian and so the low order byte is stored before the higher
order bit.

since this is data sections we are looking at it is safe to ignore the
instructions generated by objdump ( they are not correct )
this kind of makes a point, to the computer everything is data, whether its
code or actual data. And that is why the demarcation in virtual address space
segregate the various types of data present in executables into different
sections.

all the data stored is either 4 byte or 8 byte aligned.

if we take a look at the readelf .data section output, we can see that the
data is literally stored as little endian. the lowest byte (little end) of the
data appears first. `1200` is actually `0012`. fancy stuff right? If we take
a good look at the rest of the data we can see the same pattern over and over
again.

this code is compiled to run atop the riscv64 architecture. meaning integers
are 64bits wide. but we just declared `\__int128`. how do we get an 128 bit
integer on a 64 bit architecture. well the compiler coupled with the c library
is very smart and figures out how to do stuff like this. My guess is
`__int128` is just a struct of a two 64 bit integers representing low and high
4 bits respectively.

## rv-pointer
pointers in risc-v are dependent on the cpu architecture. on a 32 bit machine
pointer is 32 bits and vice versa.
from the code above `*p1` is a pointer with the value of `0x1234`
and `*p2` points to `a`.
if we take a look at the data section of our compiled code. we can see that
*p contains a value which is a memory address essentially.
*p2 on the other hand contains `0`. why is this the case? `i` is in the bss
section which is the place for unitialized data and zero values. The address 
of variables in .bss section is mostly zero (fact check me on this). And so
*p2 essentially hold an invalid memory address that will segfault if we try to
use it.
