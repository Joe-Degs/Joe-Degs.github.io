---
layout: post
title: "Learning to Cross Compile with the Zig Toolchain"
categories: systems
tags: zig risc-v systems
---

---
Lots of things have happened since the last time I wrote something here.
One of those things was learning [Zig](https://ziglang.org).
The good thing about learning Zig, aside from it being an awesome programming
language is that Zig is a toolchain for cross-compiling Zig and C code for
almost any architecture you can dream of. Did I mention it is [interoperable](https://ziglang.org/documentation/master/#C) with C?

Previously, cross-compiling C code meant downloading a toolchain,
for every target architecture I wanted to compile for. With zig, I can cross-compile without
ever having to download a new compiler toolchain, like ever. How it does this is something I am still struggling to understand. Right now, I just know that it works and that I can use it.

## some background
Sometime last year, I started taking the [MIT 6.S081 Course](https://pdos.csail.mit.edu/6.828/2021/overview.html) to learn the design and implementation of operating systems. I didn't exactly stick with it till the end, but
I learned a lot on my very short journey. Even trying my hands on a few [lab assignments](https://pdos.csail.mit.edu/6.828/2020/labs/) as I went along. The course uses a toy operating system [xv6-riscv](https://github.com/mit-pdos/xv6-riscv) created for the RISC-V architecture to illustrate the intricate details of operating system design and implementation.


I intended to save all the work I did on my own fork of the repository at [github.com/Joe-Degs/xv6-riscv](https://github.com/Joe-Degs/xv6-riscv), but since I ended up not doing much work on it, there is not much to see over there.

## so.. what's up
Yes, my fork had been lying dormant until a few days ago. I got bored and didn't have much to do, so I looked around 
for something fun to while away the time. Then the idea came to me, since Zig is so good at cross-compiling 
C code, why not try porting over the userspace of the xv6-riscv OS's userspace to zig? And my thinking was that maybe, just maybe, if xv6-riscv is in a language that does not feel like a chore to program, then I would be excited 
enough to actually do all the lab assignments (and yes, I know I might be wrong, but let a young boy dream).

## what I wanted to do
I started the project with no idea of how to actually compile C code with Zig, except for the knowledge of having seen somebody do it on YouTube. So I went back and watched it, and then I looked around the internet for blog posts on doing something similar.


The first challenge was figuring out which target I wanted to compile the code for. This was supposed to be
be the easiest challenge because the Makefile provided as part of the course's lab materials contained
some pieces of that information. I knew I wanted to compile for a RISC-V target, but I needed to supply
the target in the format that the Zig toolchain understands, and that was where I struggled.


The next challenge was figuring out how to bundle both ZIG and C source files to create an executable. And
since Zig was interoperable with C, I wanted to write some xv6-riscv userspace code in Zig while also trying
to port over the ones in C (I had big plans).

All the other challenges I faced were the result of trying to solve the first two challenges.

## what I did
I initialized a new zig project in the root of my xv6-riscv fork and then set out to make the world
a better place with a little bit of zig magic. The first place I got to work was the `build.zig` file for the project.


After setting up, I had to figure out which target to build binaries for; it was a little tricky because the operating system we are compiling for is xv6-riscv, and zig has no idea what that is.
I wanted a target for Zig that would give me a similar output as the C compiler's target.
```
➜  xv6-riscv git:(zig-port) ✗ file kernel/kernel
kernel/kernel: ELF 64-bit LSB executable, UCB RISC-V, RVC, double-float ABI, version 1 (SYSV), statically linked, with debug_info, not stripped
```

After looking around and reading some standard library code, I stumbled upon the target that made the stars align.
The resulting target in the zig build file looked something like this.
```ts
// file: build.zig

const target = CrossTarget{
    // the cpu architecture I intend to produce binaries for
    .cpu_arch = .riscv64,

    // the operating system to compile the code for, we specify
    // `freestanding` over here because this doesn't really run
    // on any operating system zig is aware of.
    .os_tag = .freestanding,

    // I still don't really understand this.
    .abi = .gnueabihf,

    // the cpu model includes the features of the cpu architecture
    // the hardware manufacturers added to their product.
    .cpu_model = .{ .explicit = &std.Target.riscv.cpu.sifive_s54 },
};
```

Now I needed some zig code that imported some of the already existing C code (remember interoperability?)
and could be compiled successfully. Also, we need to get a similar output as above when we run the `file` command on the resulting binary. I decided that a Hello World test program would do, and I ended up with this.
```ts
// file: src/test.zig

// import xv6-riscv libraries needed
const c = @cImport({
    @cInclude("types.h");
    @cInclude("stat.h");
    @cInclude("user.h");
});

// The main function of the xv6-riscv userspace program
// this is the zig equivalent of `int main(int argc, char* argv[])`
// we use `export` keyword because we need to link against some
// other xv6-riscv programs if we want this program to execute.
// We have callconv(.C) because of the same reasons, we want our
// code to be compatible with the C ABI
export fn main(_: c_int, _: [*c][*c]u8) callconv(.C) c_int {
    const hello = "Hello world!\n"

    // xv6-riscv is unix-like and has the similar syscalls.
    _ = c.write(1, @ptrCast(*const anyopaque, hello), hello.len);

    return 0;
}
```
It took me an awefully long time to get the code compiling, and I'm still not sure it is correct, but at least it compiles.


The next challenge was getting this to compile so that we could get a binary fit to be executed.
But since we are including
header files, we need to include the path to those header files so that the Zig toolchain can find them and make
use of them. I also mentioned that a few C programs need to be linked with the Zig program to produce the final
binary. Luckily, the link script for doing the linking is already provided, so we don't have to worry about that.
The main challenge was getting the includes right and successfully adding the C dependencies to produce a suitable
binary. In the end, I was able to cobble something together in the `build.zig` file that looked something like this:
```ts
// file: build.zig

pub fn build(b: *std.Build) void {
    // ...

    // add the executable and the zig source code that produces it 
    const exe = b.addExecutable(.{
        .name = "_" ++ source,
        .root_source_file = .{ .path = "src/test.zig" },
        .target = target,
        .optimize = optimize,
    });

    // add the location of the included header files
    exe.addIncludePath("kernel");
    exe.addIncludePath("user");

    // add list of all C dependencies and the compiler flags for compiling them
    exe.addCSourceFiles(c_deps, c_flags);

    // also any assemble sources that are depended on
    exe.addAssemblyFileSource(file_source);

    // add the linker script for bundling everything together into one binary
    exe.setLinkerScriptPath(.{ .path = "user/user.ld" });

    // now install the motherfucker
    b.installArtifact(exe);

    // ...
}
```
Compiling and running `file` on the resulting binary produced the same output as its C equivalent.
Except for the fact that we are stripping this one of all its debug information.
```
➜  xv6-riscv git:(zig-port) ✗ file user/_test
user/_test: ELF 64-bit LSB executable, UCB RISC-V, RVC, double-float ABI, version 1 (SYSV), statically linked, stripped
```
This made me happy, very happy.

I then moved on to execute it in the xv6-riscv operating system but got some very sad news :-(
```
xv6 kernel is booting

hart 1 starting
hart 2 starting
init: starting sh
$ test
exec test failed
$
```
I still haven't been able to figure out why it fails to execute at the time of this writing. One of my challenges is that I lack the debugging skills needed to debug a problem such as this, but I am learning, and I hope to find
answers oneday.


All the work I have done so far on this project can be found [here](https://github.com/Joe-Degs/xv6-riscv/tree/zig-port).

## one more thing before I go
While doing this, I discovered some really cool things about the zig build system that made me fall more in love with it.
I understand that you get to write more code to compile, but for some reason I actually prefer that to writing
shell scripts or Makefiles.


One of the fun little things I learned from this is how build steps work in the Zig build system.
One of the processes involved in compiling the userspace code involved generating some assembly code with a Perl script.
This is how it was done in the Makefile.
```
user/usys.S : user/usys.pl
	perl user/usys.pl > user/usys.S
```

Zig provides a build step for generating files and then caching those files so they can be reused
in subsequent steps that rely on that file being available; an example is the compile step. It is a lot more
verbose than the Makefile or shell equivalent but fits nicely into the Zig build system.
```ts
// file: build.zig

pub fn build(b: *std.Build) void {
    // ...

    // create a step to generate and cache the usys.S file
    const usys_source = blk: {
        var code: u8 = undefined;
        const usys_contents = b.execAllowFail(
            &[_][]const u8{
                "perl",
                b.pathFromRoot("user/usys.pl"),
            },
            &code,
            .Ignore,
        ) catch |err| @panic(b.fmt("failed to create usys.S: {}", .{err}));

        break :blk std.build.Step.WriteFile.create(b).add("usys.S", usys_contents);
    };

    // ...
}
```

## what's next?
The next step is figuring out how to get the compiled code to execute, or how to get it to compile correctly if that is the problem.

## resources
- [xv6-riscv book](https://pdos.csail.mit.edu/6.828/2021/xv6/book-riscv-rev2.pdf)
- [Course videos](https://youtube.com/playlist?list=PLVW70f0xtTUxHXRtZhGEJAiBDFx-ofc_G)