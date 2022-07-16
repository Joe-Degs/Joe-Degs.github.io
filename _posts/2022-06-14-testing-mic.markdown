---
layout: post
title: "Hello World!"
categories: misc
tags:
---

![vim text editor in a terminal](/assets/terminal-vim.png)
*satisfying picture of vim in my centos vm*

## Introduction
*Welcome to the blog, your presence is much appreciated. Now have your seat and
lets get started. Time is far spent.*

This blog is about a lot of things, things I'm hoping to
learn. There's going to be a little bit of everything, lowlevel and embedded 
programming for linux systems, network and socket programming, web development 
hopefully and many other things I can't think of right now.

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
```asm
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
to look into something you've been scared of or in extreme end help you 
understand some weird concept.This blog is all about doing the hard scary stuff 
and sweating through it. Yep that's it.

~ Joe
