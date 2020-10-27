---
title: "Making C Programs Small: The Beauty of C"
subtitle: ""
date: 2020-10-15T10:00:00+03:00
draft: false
description: "We write a program that can work without C standard library and is very lightweight."

tags: []
categories: []

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
---
```shell
$ gcc -static -O2 -o hello hello.c
$ du -h --apparent-size hello
843K	hello
```

Eight hundred forty-three kilobytes for such a simple program! What a waste of space in 2020, yeah?
Nevertheless, in this post, we'll try to make our programs smaller (maybe a lot).

## Getting rid of libc

The fact is that by default, C compilers automatically link our code with [GNU C Library][glibc] and most of the binary size is taken by it, so our initial goal is going to be to make our program independent from that. To make it possible, we need to write an entry point to our program and wrappers for system calls in Assembler. Let's do that (I'm using x86-64 instructions set):

```asm
# start.s
.section .text
.globl _start

_start:
    xorl %ebp, %ebp # mark the end of a stack frame
    movl   (%rsp), %edi        # argc
    leaq  8(%rsp), %rsi        # argv
	leaq 16(%rsp,%rdi,8), %rdx # envp
    xorl %eax, %eax
    call main
    movl %eax, %edi # exit code
    movl  $60, %eax # exit syscall num
    syscall
```
---
```c
// sys.h
#ifndef SYS_H_
#define SYS_H_
typedef long int ssize_t;
ssize_t strlen(const char*);
ssize_t read(ssize_t, char*);
ssize_t write(const char*);
#endif //SYS_H_
```

```asm
# sys.s
.section .text
.globl strlen
.globl read
.globl write

# const char* (%rdi)
strlen:
    xorl %eax, %eax
    cmpb $0, (%rdi)
    je .strlen_ret
.strlen_loop:
    addq $1, %rax
    cmpb $0, (%rdi,%rax)
    jne .strlen_loop
.strlen_ret:
    ret

# int64 (%rdi)
# char* (%rsi)
read:
    movq %rdi, %rdx
    xorq %rax, %rax
    xorq %rdi, %rdi
    syscall
    ret

# const char* (%rdi)
write:
    call strlen
    movq %rdi, %rsi
    movq %rax, %rdx # number of bytes to output
    movq   $1, %rax # write syscall num
    movq   $1, %rdi # stdout file descriptor
    syscall
    ret
```

### Dumb calculator

For the purpose of showing that one can do something meaningful without exploiting C library, I shall implement a CLI program that takes +|-|*|/ as an operation from an argument and reduces the list from the stdin with it.

```c
// io.h
#ifndef IO_H_
#define IO_H_
#define EOF (-1)
int getc();
int is_digit(char);
int read_int(int*);
int digits_num(int);
void write_int(int);
#endif //IO_H_
```
```c
// io.c
#include "./io.h"
#include "./sys.h"

int getc() {
    char c;
    return read(1, &c) == 1 ? c : EOF;
}

int is_digit(char c) {
    return c >= '0' && c <= '9';
}

int read_int(int* err) {
    char c;
    while ((c = getc()) != EOF && c != '-' && c != '+' && !is_digit(c));
    int is_negative = 0;
    while (c == '-' || c == '+') {
        if (c == '-') {
            is_negative = !is_negative;
        }
        c = getc();
    }
    int x = 0;
    if (err && !is_digit(c)) {
        *err = 1;
    }
    while (is_digit(c)) {
        x = x * 10 + (c - '0');
        c = getc();
    }
    if (is_negative) {
        x = -x;
    }
    return x;
}

int digits_num(int x) {
    int num = 0;
    do {
        num++;
        x /= 10;
    } while (x);
    return num;
}

void write_int(int x) {
    char buf[12];
    int chars_num = digits_num(x);
    if (x < 0) {
        chars_num++;
        buf[0] = '-';
        x = -x;
    }
    buf[chars_num] = '\0';
    do {
        buf[--chars_num] = '0' + x % 10;
        x /= 10;
    } while (x);
    write(buf);
}
```
---
```c
// ops.h
#ifndef OPS_H_
#define OPS_H_
#include "./sys.h"
typedef int (*binary_op)(int, int);
int accumulate(ssize_t, int[], binary_op);
binary_op parse_op(char);
#endif //OPS_H_
```
```c
// ops.c
#include "./ops.h"
#include "./sys.h"

static int sum(int x, int y) {
    return x + y;
}
static int sub(int x, int y) {
    return x - y;
}
static int mult(int x, int y) {
    return x * y;
}
static int div(int x, int y) {
    return x / y;
}

int accumulate(ssize_t n, int xs[], binary_op op) {
    int result = xs[0];
    for (ssize_t i = 1; i < n; i++) {
        result = op(result, xs[i]);
    }
    return result;
}

binary_op parse_op(char op) {
    switch (op) {
    case '+': return sum;
    case '-': return sub;
    case '*': return mult;
    case '/': return div;
    }
}
```
---
```c
// main.c
#include "./sys.h"
#include "./io.h"
#include "./ops.h"

void print_usage() {
    write(
        "Usage:\n"
        "  ./calc [+|-|*|/]\n"
    );
}

void print_empty_list_error() {
    write(
        "Error: empty list\n"
    );
}

int main(int argc, char* argv[]) {
    if (argc != 2) {
        print_usage();
        return 1;
    }
    if (strlen(argv[1]) != 1) {
        print_usage();
        return 1;
    }
    char op = argv[1][0];
    switch (op) {
    case '+': break;
    case '-': break;
    case '*': break;
    case '/': break;
    default: print_usage();
             return 1;
    }

    const int MAXN = 1000;
    int xs[MAXN];
    int err = 0;
    int result = read_int(&err);
    if (err) {
        print_empty_list_error();
        return 2;
    }
    binary_op bin_op = parse_op(op);
    while (!err) {
        xs[0] = result;
        int n;
        for (n = 1; n < MAXN; n++) {
            xs[n] = read_int(&err);
            if (err) break;
        }
        result = accumulate(n, xs, bin_op);
    }
    write_int(result);
    write("\n");
    return 0;
}
```

Write a Makefile:

```makefile
CFLAGS = -Wall -Wextra -Werror -nostdlib -static -O2
ASFLAGS = -64
LDFLAGS = -m elf_x86_64 -nodefaultlibs

all: calc

calc: start.o sys.o io.o ops.o main.o
	$(LD) $(LDFLAGS) $^ -o $@

.phony clean:
	rm -rf *.o calc
```

Now let's build and test it:

```shell
$ make
as -64  -o start.o start.s
as -64  -o sys.o sys.s
cc -Wall -Wextra -Werror -nostdlib -static -O2   -c -o io.o io.c
cc -Wall -Wextra -Werror -nostdlib -static -O2   -c -o ops.o ops.c
cc -Wall -Wextra -Werror -nostdlib -static -O2   -c -o main.o main.c
ld -m elf_x86_64 -nodefaultlibs start.o sys.o io.o ops.o main.o -o calc
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
3.5K	calc
```

The size equals 3.5KB, which is approximately $99.6\\%$ smaller than the initial hello-world program. Sounds fascinating :tada:. Of course, we become seriously limited by the inability to use functions from the standard library. However, the critical point is that we **could** get independent from runtime libraries and thus have almost zero initialization runtime, making C the only good choice for implementing system kernels and developing for cheap microcontrollers. 

Source code of the example is available [here](https://gist.github.com/magnickolas/bafd9caee039e0cfe27329645d9dadbb).

[glibc]: https://www.gnu.org/software/libc/
