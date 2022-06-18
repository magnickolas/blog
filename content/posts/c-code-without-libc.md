---
title: "Making C Programs Small: The Beauty of C"
subtitle: ""
date: 2020-10-15T10:00:00+03:00
draft: false
description: "We write a program that can work without C standard library and is very lightweight."

tags: [c, asm]
categories: [exploration]

hiddenFromHomePage: false
hiddenFromSearch: false

featuredImage: ""
featuredImagePreview: ""

toc:
  enable: true
math:
  enable: true
lightgallery: false
license: ""
---

## Introduction

Let's write a simple hello-world program in C language and compile a static binary with GCC:<!--more-->

```c
#include <stdio.h>

int main() {
    printf("Hello, reader!\n");
    return 0;
}
```

First, let's just compile it and check the size of the stripped binary file:
```shell
$ gcc -static -O2 -o hello hello.c
$ strip hello
$ du -h --apparent-size hello
15K     hello
```
It's 15K, but this is not a static binary and uses system shared libraries:
```shell
$ ldd hello
        linux-vdso.so.1 ...
        libc.so.6 => ...
        /lib64/ld-linux-x86-64.so.2 => ... 
```
Let's compile a static binary and check its size:
```shell
$ gcc -static -O2 -o hello hello.c
$ strip hello
$ du -h --apparent-size hello
681K    hello
```

Six hundred eighty-one kilobytes for such a simple program! What a waste of space in 2020, huh? 🙈
Nevertheless, in this post, we'll try to make our programs smaller (maybe a lot).

## Getting rid of C standard library

The fact is that by default, C compilers automatically link our code with [GNU C Library][glibc] and most of the binary size is taken by it, so our initial goal is going to be making our program independent from that. To make it possible, we need to write an entry point to our program and wrappers for system calls in Assembler. Let's do that (I'm using x86-64 instructions set):

{{< gist magnickolas bafd9caee039e0cfe27329645d9dadbb "start.s" >}}
{{< gist magnickolas bafd9caee039e0cfe27329645d9dadbb "sys.h" >}}
{{< gist magnickolas bafd9caee039e0cfe27329645d9dadbb "sys.s" >}}

### Dumb calculator

For the purpose of showing that one can do something meaningful without exploiting C library, I shall implement a CLI program that takes +|-|*|/ as an operation from an argument and reduces the list from the stdin with it.

{{< gist magnickolas bafd9caee039e0cfe27329645d9dadbb "io.h" >}}
{{< gist magnickolas bafd9caee039e0cfe27329645d9dadbb "io.c" >}}
{{< gist magnickolas bafd9caee039e0cfe27329645d9dadbb "ops.h" >}}
{{< gist magnickolas bafd9caee039e0cfe27329645d9dadbb "ops.c" >}}
{{< gist magnickolas bafd9caee039e0cfe27329645d9dadbb "main.c" >}}

Write a Makefile:

{{< gist magnickolas bafd9caee039e0cfe27329645d9dadbb "Makefile" >}}

Now let's build and test it:

```shell
$ make
as -64  -o start.o start.s
as -64  -o sys.o sys.s
cc -Wall -Wextra -Werror -nostdlib -static -O2 -fno-stack-protector   -c -o io.o io.c
cc -Wall -Wextra -Werror -nostdlib -static -O2 -fno-stack-protector   -c -o ops.o ops.c
cc -Wall -Wextra -Werror -nostdlib -static -O2 -fno-stack-protector   -c -o main.o main.c
ld -m elf_x86_64 start.o sys.o io.o ops.o main.o -o calc
strip calc
```

```shell
$ ./calc some unappropriate arguments
Usage:
  ./calc [+|-|*|/]
$ echo | ./calc +
Error: empty list
$ echo "12 34 -56" | ./calc +
-10
$ echo "1 2 3 4 5" | ./calc '*'
120
```

### How small is it?

```shell
$ du -h --apparent-size calc
9.0K    calc
```

The size equals 9KB, which is approximately $98.7\\%$ smaller than the initial hello-world program. Sounds fascinating :tada:. Of course, we become seriously limited by the inability to use functions from the standard library. However, the critical point is that we **could** get independent from runtime libraries and thus have almost zero initialization runtime, making C the only good choice for implementing system kernels and developing for cheap microcontrollers. 

Source code of the example is available [here](https://gist.github.com/magnickolas/bafd9caee039e0cfe27329645d9dadbb).

### P.S.

To just produce a smaller binary for simple programs such as hello-world example, we can use [musl libc][musl], which is a less-monolithic implementation of the C standard library compared to [glibc][glibc].
```shell
$ musl-gcc -static -O2 -o hello hello.c
$ strip hello
$ du -h --apparent-size hello
14K     hello
```
And... we've got a binary that weights less than the initial dynamically-linked version.

[glibc]: https://www.gnu.org/software/libc/
[musl]: https://musl.libc.org/
