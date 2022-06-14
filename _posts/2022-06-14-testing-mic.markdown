---
layout: post
title: "Testing Mic cho lo lo..."
categories: misc
---

![vim text editor in a terminal](/assets/terminal-vim.png)
*satisfying picture of vim in my centos vm*

## Introduction
Welcome to the blog. This blog is about a lot of things most of which I do not
know about yet. There's going to doing a little bit of everything, a big bit of
bit of everything now that I think about it. 
There will definitely be some code, a lot of code actually.
Most of what I write about will be me trying to figure out how something that
already exists works and maybe trying to implement the thing myself.

There will be alot of golang code on this blog. Why?? I love the language
dammit
```go
package main

import "fmt"

func main() {
    fmt.Println("Hello Beautiful People ☺")
}
```

There will be some c too. Why?? Cos I love to suffer?
```c
#include <stdio.h>

int main(void)
{
        println("%s\n", "Hello Chololo");
        return 0;
}
```

Python might pop its head sometimes. Why?? I just can't seem to run away from it
```python
print("Hmmm!")
```

Some Javascript might come up once in a while. Why?? We don't know what the
future holds, let's keep all options open
```javascript
console.log("Tadaa... I'm here!")
```

Probably some assembly 😬. Assembly! are you sure?? 
You don't have to worry, [RISC-V](https://en.wikipedia.org/wiki/RISC-V) is relatively easy
```nasm
.globl main


.data
msg: .string "Hello...\n"
.equ msg_len, 9

.equ STDOUT, 1
.equ WRITE_SYSCALL_NO, 64
    
.text
main:
    li a0, STDOUT
    la a1, msg
    li a2, msg_len
    li a7, WRITE_SYSCALL_NO
    ecall
    li a0, 0
```

We might need some shell scripts to tie things together sometimes.
```sh
#!/bin/bash

echo "Heyy peeps!"
```

I hope the things I write actually help somebody solve some problem, ginger you 
to look into something they've been scared of in the past or on the
extreme end help you understand some weird concept. This blog is
all about doing the hard scary stuff and suffering through it. Yep that's it.

~ Joe
