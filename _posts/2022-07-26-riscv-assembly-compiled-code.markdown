---
layout: post
title: "RISC-ing it all: How binary data is stored"
categories: systems
tags: risc-v assembly systems
---

---
## the beginning
If you are like me, you have written a program, compiled it and run the
executable binary file at some point in your life.
And you've done it a couple of times, then you started wondering, how
does it all work especially the compiling and execution part. 
It bothered you enough to want to learn it (or atleast read about it),
but somebody told you were wasting your time, that you would be better off
learning to build web apps. And you knew they were not lying. 
You tried to learn how to write beautiful web apps but realised you didn't care about building beautiful
things as much as you liked learning about how beautiful things get built.

And so you went on the internet and you found blogs
and even big books (very big ones) dedicated to teaching how programs get 
compiled (or get executed) and you tried to read them
but they made absolutely no sense to you. You gave up on the idea, but not
before you learned that programs at the end of the day get executed by
something called the [CPU](https://en.wikipedia.org/wiki/Central_processing_unit) 
and that the CPU performs actions based on something called an
[instruction](https://pclt.sites.yale.edu/cpu-instructions) and that instructions are [bit](https://en.wikipedia.org/wiki/Bit)
patterns and depending on the pattern of the bits, the CPU would choose the appropriate
course of action.

You didn't even know what that meant
but it felt like a good thing to know so you held onto that piece of knowledge.
At another time you read somewhere that all the programs that we
cannot now live without are actually all just bunch of bit patterns that the
CPU thing can execute (bunch of instructions and **data** essentially).
Your mind was blown and your socks were knocked off but you put them back on and
continued reading. You found out that a [compiler](https://en.wikipedia.org/wiki/Compiler) 
is responsible for taking the programs you write and turning them into bit 
patterns that the CPU understands and can execute.

And so it all started for you, you knew there and then that there was no turning back,
you knew too much to go back now so you went further and discovered
that the type of bit patterns the CPU understands is dictated by something
called the [Instruction Set Architecture](https://en.wikipedia.org/wiki/Instruction_set_architecture),
which is even harder to understand than everything else because the manuals
explaing what they are and what they do are
big (alot bigger than the compilers') and you didn't know where to start. But then
you found the precious little ISA ([RISC-V](https://en.wikipedia.org/wiki/RISC-V)),
beautifully crafted in the open with much much smaller manuals, specs and
a loving community 
so you read, and read and watched videos and continued to read some more. And
everything made sense.
All the stars aligned and the universe made sense, finally the glimmer of light
in the darkness you were hoping for all along.

_Joe, enough! cut the crap and talk about what we are doing here today_

Okay, let's get into why we are here again. We know that when we write code and
compile it, it gets translated into instructions and data that the CPU can understand
and execute. These instructions get stored on disk in an [executable](https://en.wikipedia.org/wiki/Executable)
file. The tools and processes involved from source code to the executable
file look vaguely like this:
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

As part of the perpetual journey of trying to figure out how things work, we are
going to figure out how data and instructions are stored in our compiled programs. 
We are going to be experimenting with alot of RISC-V assembly code and binary data. 
Lets get into it!

## getting object files from our source code
We have establised some common ground in the beginning. Now let's move to
figuring out how we can get instructions and data from our compiled code, and 
inspecting them to see what we can find. The program we are using for this
particular task is just a simple C program that does absolutely nothing but
contains data -- data is very important for things to make sense.
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

I have the the RISC-V GNU Compiler [toolchain](https://github.com/riscv-collab/riscv-gnu-toolchain) 
installed on my computer. And I am going to be using these tools to compile,
disassemble and dump binaries all over the place. Lets start by compiling our
C code into an object file.

```
joe@debian:../$ riscv64-linux-gnu-gcc -c -g main.c
joe@debian:../$ ls
main.c  main.o
```
We've got ourselves a nice little object file `main.o`.
It looks beautiful doesn't it. We now have our object file (a binary file) so we can
inspect it to see what we can find inside its nooks and cranies

### inspecting the object file
Lets start by running `file` on the file to see what type of stuff we are
dealing with.

```
joe@debian:../$ file main.o
main.o: ELF 64-bit LSB relocatable, UCB RISC-V, version 1 (SYSV), with debug_info, not stripped
```
Well it turns out our object file is an [ELF](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format) 
file. If you use linux you've used it before and probably know what it is.
It is the file format of executable files, object files and shared libraries.

Lets try to make sense out of the output of the `file` command up there.
- **ELF**: this specifies the file format, in this case an ELF file. There are
  many other file formats out there for executables and object files and their
  specification documents if you want to play around with them.
- **LSB**: specifies the [endianness](https://en.wikipedia.org/wiki/Endianness) of the file.
  There are two types for executable files that I care and know about: little endian or
  Least Significant Byte (LSB) and big-endian or Most Significant Byte (MSB). 
  It essentially indicates how bytes are stored in memory, LSB means the least
  significant byte of a [word](https://en.wikipedia.org/wiki/Word_(computer_architecture))
  is stored at the lower memory address and the most significant
  byte at the higher memory address and MSB means the most significant byte is 
  stored at the lower memory address and the least significant byte at a higher
  memory address.
- **relocatable**: Since this is an object file, it is not yet executable even
  though it contains instructions and data. It is a *relocatable* file because
  it needs to be linked (by the linker) with other object files to create an executable.
- **UCB RISC-V**: specifies the ISA of the instructions in the file. In this
  case the precious little RISC-V. UCB (UC Berkeley) is where the ISA was developed.
- **(SYSV)**: This specifies the OS and [ABI](https://en.wikipedia.org/wiki/Application_binary_interface) 
  of the file.
- **with debug_info, not stripped**: We compiled the code with the `-g` option to preserve
  snippets of code, names of variables and functions (called symbols) in the file.
  These snippets and symbols are referred to as debug symbols and they are not
  stripped from the object file. As the name suggests these symbols are valuable
  for debugging programs.

Now that we are over with that, lets take a closer look at the general structure of
an ELF file.

### closer look at the elf object file
Lets take a look at the ELF file header (using `readelf`).

```
joe@debian:../$ readelf -h main.o
ELF Header:
  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00
  Class:                             ELF64
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              REL (Relocatable file)
  Machine:                           RISC-V
  Version:                           0x1
  Entry point address:               0x0
  Start of program headers:          0 (bytes into file)
  Start of section headers:          2992 (bytes into file)
  Flags:                             0x5, RVC, double-float ABI
  Size of this header:               64 (bytes)
  Size of program headers:           0 (bytes)
  Number of program headers:         0
  Size of section headers:           64 (bytes)
  Number of section headers:         20
  Section header string table index: 19
```
Well well well! we have most of the same information we got from `file` here and
soo much more, Nice. We have `section header` and `program header` information. 
Section and Program headers are the main parts of the ELF file format, 
together they contain most of the things we care about -- instructions and data.

Let's take a look and see what they look like. We start with the program headers
also called `segments`.
```
joe@debian:../$ readelf --segments main.o

There are no program headers in this file.
```
What! why? Well, program headers are only needed by the operating system because they
provide information to it about where and how to load the sections into memory
and since object files are not yet *linked* (by the linker) into executables,
the compiler saw no need to include them into the object file. So we do not
have program headers on object files. Lets move on.

What about the sections?
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

... (bunch of stuff we don't care about)

  [17] .symtab           SYMTAB           0000000000000000  00000330
       00000000000003d8  0000000000000018          18    37     8
  [18] .strtab           STRTAB           0000000000000000  00000708
       00000000000000ae  0000000000000000           0     0     1
  [19] .shstrtab         STRTAB           0000000000000000  00000b00
       00000000000000ae  0000000000000000           0     0     1
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings), I (info),
  L (link order), O (extra OS processing required), G (group), T (TLS),
  C (compressed), x (unknown), o (OS specific), E (exclude),
  p (processor specific)
```
Yep we got them!, and they are numerous, more than I was expecting. Remember the *with
debug_sections, not stripped*? we have alot of debug stuff in the compiled code that are helpful for
finding bugs but take up space.

### peeking at the relevant sections
Now we try to find out what all the different sections mean and what they do.
I mean there has to be a purpose to every single one of them right? But we will
only look at the those that contain data and instructions. Lets get to looking
at the different sections and the data each one of them contains. We will be
looking at some hexdumps 😬. The relevant sections are: `.text`, `.data`,
`.rodata`, `.bss` and `.symtab`.

#### .text section
this section contains the instructions we love and cherish. It
contains the bit patterns that tell the CPU what to do. Essentially our code
but in binary format.

```
joe@debian:../$ readelf -x .text main.o

Hex dump of section '.text':
0x00000000 011122ec 0010aa87 2330b4fe 2326f4fe ..".....#0..#&..
0x00000010 81473e85 62640561 8280              .G>.bd.a..
```

Well since instructions are just bit patterns it makes sense that it gets stored
as bunch of bytes. We can take a look at the dissambly of the above
code and see what those bit patterns actually mean.

```
main.o:     file format elf64-littleriscv


Disassembly of section .text:

0000000000000000 <main>:
int num;
int number       = 0x1234;
const int constant = 0x4567;

int main(int argc, char *argv[])
{
   0:   1101                    addi    sp,sp,-32
   2:   ec22                    sd      s0,24(sp)
   4:   1000                    addi    s0,sp,32
   6:   87aa                    mv      a5,a0
   8:   feb43023                sd      a1,-32(s0)
   c:   fef42623                sw      a5,-20(s0)
        return 0;
  10:   4781                    li      a5,0
}
  12:   853e                    mv      a0,a5
  14:   6462                    ld      s0,24(sp)
  16:   6105                    addi    sp,sp,32
  18:   8082                    ret
```
That is a lot of assembly code for a function that does absolutely nothing.
The debug symbols are making it easier to understand what the assembly code above
In the C code above we have a main function with signature
`int main(int argc, char *argv[])` that does nothing and returns zero. 

According to the RISC-V calling conventions, arguments are passed to functions through the
`a0-a7` registers and the return value of functions are placed in `a0-a1`
registers. `s0` doubles as a frame pointer and `sp` is the stack pointer.

The stack grows downwards towards the low memory addresses. The way I think
about this is, if the stack grows downwards towards the lower addresses, to
allocate space will require a subtraction from the stack pointer and to 
deallocate space will require an addition to the stack pointer.

Because we compiled our code to to run on a 64 bit machine, memory addresses (pointers) are
64 bits (8 bytes). Memory is byte addressable and adjacent memory locations are
32/64 bits apart(I am not really sure on the exact value), either way we are able to load and
store 64 bits at a time with the `ld` and `sd` instructions respectively.

Now let's dissect the assembly and understand what every part of it means.
We start with the first three instructions

```asm
addi    sp,sp,-32
sd      s0,24(sp)
addi    s0,sp,32
```
Over here it looks like the stack frame is been setup for the function. 32 bytes
of space is allocated on the stack, and then the frame pointer `s0` is saved off at
the first 8 bytes of the stack. `addi s0, sp, 32` is setting the frame pointer `s0`
to point to the start of the allocated stack space for the function. So a stack
frame is a range of memory from the frame pointer to the stack pointer.

```asm
mv      a5,a0
sd      a1,-32(s0)
sw      a5,-20(s0)
```
Over here, the first argument `int argc`'s value in `a0` gets copied to to the `a5` register, 
and `a1` the second argument register holding `char *argv[]`'s value is stored off on the
stack (note that it is a pointer). The integer value in the `a5` also gets
stored on the stack. This snippet of assembly code stores the arguments of the
function on the stack.

```asm
li      a5,0

mv      a0,a5
ld      s0,24(sp)
addi    sp,sp,32
ret
```
In the first line of the above snippet, the immediate value `0` is copied into the `a5` register and then
the value is copied to the `a0` register. The frame pointer stored that was
stored on the stack initially is copied back into the `s0` register. 
`addi sp, sp, 32` deallocates the stack, essentially moving it back to its state before
the `main` function. Note that the 0 moved into a0 is the return value of the function.

#### .data section
contains the all the global and static variable of the compiled code.

```
joe@debian:../$ readelf -x .data main.o

Hex dump of section '.data':
0x00000000 34120000                            4...
```

Lets take a look the snippet of our code that might correspond to this section
(global and static variables).

```c
/* ... */
int number = 0x1234;
```

Okay, we have the `1234` with alot of zeroes after it in the hex dump up above but it looks like its been reversed.
Remember the LSB and MSB? We see its effect over here, we see that the `34`
comes before `12` in the hexdump because it is storing the data in the
little-endian (stores the little end first) format. But how do we know which one is the
little end and which is the big end. The byte with the lowest place value
is the little end (least significant byte) and the byte with the highest place value is the big end (most
significant byte)

#### .bss
this one is just like the *.data* section but only keeps those
variables that do not have an initial value.

```
joe@debian:../$ readelf -x .bss main.o
Section '.bss' has no data to dump.
```

Ehmm what is going on here? We declared a global variable and did not
initialize it.

```c
/* ... */

int num;

/* ... */
```

Because the variables in the *.bss* section contain no data, the compiler leaves
the section empty in the file to save some space. It is the responsibility of
the executable loader to fill in the zeroes when the file is loaded into memory
for execution. Let's look at the dissambly of the section with `objdump` to test this logic.

```
joe@debian:../$ riscv64-linux-gnu-objdump -z -S -D -j .bss main.o

main.o:     file format elf64-littleriscv


Disassembly of section .bss:

0000000000000000 <num>:
int num;
   0:   0000                    unimp
   2:   0000                    unimp
```

Yeah well, we see the `num` variable name with a bunch of zeroes in that
sections. Everything looks good.

#### .rodata
this section contains constant data. If you have an integer
constant or string constant, they get stored in this section. If you pass
a literal string to a function, it gets stored here.

```
joe@debian:../$ readelf -x .rodata main.o
Hex dump of section '.rodata':
  0x00000000 67450000                            gE..
```

What does the corresponding C code to this section look like

```c
/* ... */

const int constant = 0x4567;

/* ... */
```

Well we see the same pattern. The data is stored in the little endian format
with a bunch of zeroes after it.

#### .symtab
this section contains information about all the symbols in the
object file: variables, constants, functions etc.
Since this section contains the symbols of the program, lets take a look at the
actual symbol table and not the hex dump of it.

```
joe@debian:../$ readelf --symbols main.o

Symbol table '.symtab' contains 41 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND
     1: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS main.c
     2: 0000000000000000     0 SECTION LOCAL  DEFAULT    1
     3: 0000000000000000     0 SECTION LOCAL  DEFAULT    2
     4: 0000000000000000     0 SECTION LOCAL  DEFAULT    3
     5: 0000000000000000     0 SECTION LOCAL  DEFAULT    4

... (bunch of stuff we don't care about) ...

    34: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT    5 .Ldebug_info0
    35: 0000000000000000     0 SECTION LOCAL  DEFAULT   13
    36: 0000000000000000     0 SECTION LOCAL  DEFAULT   15
    37: 0000000000000000     4 OBJECT  GLOBAL DEFAULT    3 num
    38: 0000000000000000     4 OBJECT  GLOBAL DEFAULT    2 number
    39: 0000000000000000     4 OBJECT  GLOBAL DEFAULT    4 constant
    40: 0000000000000000    26 FUNC    GLOBAL DEFAULT    1 main
```
Wow cool stuff indeed. We can see the variable names, the `main` function
and even the filename in the symbol table. That's wild!

### final words
This post spiralled out of control quickly and took a life of its own, it was
fun and I learnt a ton writing this one. I might continue and write more stuff
on this topic of how data and code get stored in binary files. I would like to continue to
figure out how especially arrays, structs and pointers are stored.. Ohh and how data
is aligned in memory and padding and all the other stuff.
And maybe look at control flow and program logic at some other time.

#### resources
