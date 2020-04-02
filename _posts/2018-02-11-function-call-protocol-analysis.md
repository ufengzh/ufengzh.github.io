---
layout: post
title:  "函数调用协议及栈分析"
date:   2018-02-11 14:32:00 +0800
categories: C
---
## 前言

堆、栈、函数调用、协程是C语言的一些基本知识，网上已经一大堆文章介绍和分析，偶尔还能听到同事们讨论，前段时间刚好听到这样两个问题：a）如何判断指针所指向的一块数据是在堆、还是栈上？b）为什么函数调用需要预先分配栈空间而不是动态分配？突然间想起很久以前的某一次面试沟通中被问到协程原理，觉得当时在口头表述上特别不让自己满意，于是就把这篇以前写好压箱底的科普文章再翻了出来。

> 本文讨论的是x86 32位平台，64位请自行查阅相关文档。
> 文内涉及的汇编代码均采用*AT&T*语法，*gcc*编译、*gdb/objdump*反汇编时默认使用*AT&T*语法，同时提供*Intel*语法支持。
> 1. *gcc -masm=intel foo.c*
> 2. *(gdb) set disassembly-flavor intel*
> 3. *objdump -M intel a.out*

## 寄存器

<center>![@图1：x86寄存器 | | 400x0](/assets/img/x86-registers.png)</center>

> 这张图原始链接来自[x86 Assembly Guide](http://www.cs.virginia.edu/~evans/cs216/guides/x86-registers.png)。

x86寄存器用途介绍：

- *EAX*：函数返回值。
- *ECX*：循环计数器。
- *EAX、ECX、EDX*：调用方保存寄存器（caller-saved），其他寄存器都是被调方保存寄存器（callee-saved）。 
- *ESP*：栈指针寄存器。
- *EBP*：栈基址寄存器，在被调子程序体执行时不会发生变化，因此通过`EBP + 偏移`能够快速定位传入参数、本地变量。

## 调用协议

各个开发者为了在不同程序中共享相同的代码或开发库，简化子程序*Subroutine*的使用，*x86*平台C语言函数调用要求调用者、被调者遵守通用的调用协议*Calling Convention*。

<center>![@图2：子程序调用栈](/assets/img/subroutine-call-stack.png)</center>

### 调用者规则

调用者规则*Caller Rules*：

1. 保存寄存器：调用方保存寄存器（*caller-saved*）包含*EAX、ECX、EDX*，这些寄存器在被调子程序中可能会被修改，而且在子程序结束后调用方依赖这些寄存器的时候，调用方就会提前把他们压入栈中以便保护子程序调用前这些寄存器存储数值（比如子程序返回值可能通过*EAX*寄存器返回）。
2. 准备调用参数：调用者将传递给子程序的参数压入栈中。
3. 调用子程序：使用*call*指令调用子程序。

> *call*、*jmp*指令区别：
> 1. *jmp*：跳转至另一个地址执行。
> 2. *call*：将当前*call*指令的下一条指令（*EIP*）压入栈中，然后跳转至指定址执行。
> 
> *call 804840b*相当于：
> 1. *push %eip*
> 2. *jmp 804840b*

当子程序结束返回后，调用者还应当：

1. 移除栈上的调用参数。
2. 恢复调用方保存的寄存器（*EAX、ECX、EDX*）。

### 被调者规则

被调者规则*Callee Rules*：

1. 栈基址准备：保存旧的*EBP*（调用者栈空间的栈基址），复制*ESP*至*EBP*寄存器。
	*push %ebp*
	*mov %esp, %ebp*
2. 分配栈空间：栈指针一般从高地址往地址方向滑动，使用*sub*指令减小*ESP*分配栈空间，用于存储子程序本地变量。
3. 保存寄存器：被调方保存寄存器（*callee-saved*）包含*EBX、EDI、ESI*，当这些寄存器在当前子程序使用的时候，就会把这些寄存器压入栈中。

当子程序体执行结束后，被调者还应该当：

1. 把返回值存储至*EAX*。
2. 恢复被调方保存的寄存器（*EBX、EDI、ESI*）。
3. 释放本地变量存储空间：将栈指针*ESP*指向栈基址*EBP*，可选择用*mov*指针实现（*mov %esp, %ebp*）。
4. 恢复旧的*EBP*。
5. 使用*ret*指令把控制权将由调用者，执行*call*指令的下一条指令。可以理解成，把调用子程序前由调用方压入栈中保存的*call*指令下一条指令弹出至*EIP*执行。

调用协议中规则比较多，但并不是所有规则都是必须的，简而言之，一次函数调用一般至少包含这几个步骤：

1. 调用者准备调用参数。
2. 调用子程序。
3. 子程序分配栈空间：被调者设置栈基址、分配栈空间存储本地变量。
	...
	被调者执行子程序体。
	...
4. 子程序释放栈空间：被调者设置返回值，释放栈空间。
5. 被调者执行*ret*返回。
6. 调用者移除调用参数。

## 源代码

函数调用源码示例*foo.c*，在Linux平台用*gcc*编译成目标代码：*gcc -m32 foo.c -o foo*。

```cpp
#include <stdio.h>

int foo(int a, int b)
{
    int sum = 0;

    sum = a + b;
    return sum;
}

int main(int argc, char *argv[])
{
    int ret = 0;

    ret = foo(2, 3);
    printf("ret = %d\n", ret);

    return 0;
}
```

## 反汇编

用*objdump*把编译后的目标代码转换为汇编代码（或者使用*gdb disass*命令）。

```plain
ufeng@ubuntu ~/p/c> objdump -d foo
0804840b <foo>:
 804840b:       55                      push   %ebp
 804840c:       89 e5                   mov    %esp,%ebp
 804840e:       83 ec 10                sub    $0x10,%esp
 8048411:       c7 45 fc 00 00 00 00    movl   $0x0,-0x4(%ebp)
 8048418:       8b 55 08                mov    0x8(%ebp),%edx
 804841b:       8b 45 0c                mov    0xc(%ebp),%eax
 804841e:       01 d0                   add    %edx,%eax
 8048420:       89 45 fc                mov    %eax,-0x4(%ebp)
 8048423:       8b 45 fc                mov    -0x4(%ebp),%eax
 8048426:       c9                      leave  
 8048427:       c3                      ret    

08048428 <main>:
 8048428:       8d 4c 24 04             lea    0x4(%esp),%ecx
 804842c:       83 e4 f0                and    $0xfffffff0,%esp
 804842f:       ff 71 fc                pushl  -0x4(%ecx)
 8048432:       55                      push   %ebp
 8048433:       89 e5                   mov    %esp,%ebp
 8048435:       51                      push   %ecx
 8048436:       83 ec 14                sub    $0x14,%esp
 8048439:       c7 45 f4 00 00 00 00    movl   $0x0,-0xc(%ebp)
 8048440:       6a 03                   push   $0x3
 8048442:       6a 02                   push   $0x2
 8048444:       e8 c2 ff ff ff          call   804840b <foo>
 8048449:       83 c4 08                add    $0x8,%esp
 804844c:       89 45 f4                mov    %eax,-0xc(%ebp)
 804844f:       83 ec 08                sub    $0x8,%esp
 8048452:       ff 75 f4                pushl  -0xc(%ebp)
 8048455:       68 f0 84 04 08          push   $0x80484f0
 804845a:       e8 81 fe ff ff          call   80482e0 <printf@plt>
 804845f:       83 c4 10                add    $0x10,%esp
 8048462:       b8 00 00 00 00          mov    $0x0,%eax
 8048467:       8b 4d fc                mov    -0x4(%ebp),%ecx
 804846a:       c9                      leave  
 804846b:       8d 61 fc                lea    -0x4(%ecx),%esp
 804846e:       c3                      ret    
 804846f:       90                      nop
```

### Step 1：准备参数

Step 1：调用者把函数参数从右到左依次压入栈中。当前示例就是立即数3、2。

```plain
 ret = foo(2, 3);
 
 8048440:       6a 03                   push   $0x3
 8048442:       6a 02                   push   $0x2
```

<center>![@图3：准备参数栈变化](/assets/img/prepare-stack-args.png)</center>

### Step 2：调用子程序

Step 2：调用者使用*call*指令调用子程序。首先把*call*指令的下一条指令压入栈中，然后跳转至子程序开始执行。

```plain
 8048444:       e8 c2 ff ff ff          call   804840b <foo>
 8048449:       83 c4 08                add    $0x8,%esp
```

<center>![@图4：调用子程序栈变化](/assets/img/invoke-subroutine-stack.png)</center>

### Step 3：分配栈空间

Step 3：子程序分配栈空间，被调者设置栈基址、分配栈空间存储本地变量。

1. 保存旧*EBP*值：*push %ebp*。
2. 分栈栈空间：

```plain
0804840b <foo>:
 804840b:       55                      push   %ebp        ; 保存旧EBP值
 804840c:       89 e5                   mov    %esp,%ebp   ; 设置子程序栈基址
 804840e:       83 ec 10                sub    $0x10,%esp  ; 分配16字节栈空间
```

被调者执行子程序体。

```plain
 int sum = 0;
 sum = a + b;

 8048411:       c7 45 fc 00 00 00 00    movl   $0x0,-0x4(%ebp)  ; 把0拷贝至[ebp - 4]
 8048418:       8b 55 08                mov    0x8(%ebp),%edx   ; 把[ebp + 8]拷贝至edx
 804841b:       8b 45 0c                mov    0xc(%ebp),%eax   ; 把[ebp + 12]拷贝至eax
 804841e:       01 d0                   add    %edx,%eax        ; 执行edx + eax并把结果存储至eax
 8048420:       89 45 fc                mov    %eax,-0x4(%ebp)  ; 把eax拷贝至[ebp - 4]
```

<center>![@图5：分配栈空间&执行子程序体栈变化](/assets/img/alloc-stack-execute.png)</center>

### Step 4：子程序释放栈空间

Step 4：子程序释放栈空间。被调者首先设置返回值，然后释放栈空间。

```plain
 8048423:       8b 45 fc                mov    -0x4(%ebp),%eax  ; 把[ebp - 4]拷贝至eax，设置返回值
 8048426:       c9                      leave                   ; 把ebp指针拷贝至esp，释放栈空间
                                                                ; 然后从栈上弹出旧ebp值
```

释放栈空间一般使用*add*指令实现（*add $0x10, %esp*），我猜测这里可能是编译优化，省略了*add*指令直接用，毕竟高级指令*leave*相当于包含了这个功能。

> [*leave*](http://www.felixcloutier.com/x86/LEAVE.html)指令：
> 1. 拷贝栈基址ebp至栈指针寄存器esp（释放之前分配的栈空间）。
> 2. 从栈中弹出旧栈基址值至ebp寄存器（恢复调用者栈基址）。

<center>![@图6：子程序释放栈空间栈变化](/assets/img/release-stack.png)</center>

### Step 5：被调者执行*ret*返回

Step 5：被调者执行[*ret*](http://www.felixcloutier.com/x86/RET.html)返回。被调者从栈中弹出返回地址至*EIP*，然后继续执行。

```plain
 8048427:       c3                      ret    
```

<center>![@图7：被调者执行返回栈变化](/assets/img/caller-return-code.png)</center>

### Step 6：调用者移除调用参数

Step 6：调用者移除调用参数。调用者先从栈上移除调用参数，然后从eax获取返回值。

```plain
 ret = foo(2, 3);
 
 8048449:       83 c4 08                add    $0x8,%esp        ; 移除调用参数占用的8字节栈空间
 804844c:       89 45 f4                mov    %eax,-0xc(%ebp)  ; 从eax获取返回值拷贝给本地局部变量
```

<center>![@图8：调用者移除调用参数栈变化](/assets/img/caller-remove-args.png)</center>

至此，函数*foo*的调用过程执行完毕，整体而言对栈空间的使用是比较简单的。

再回过头来看之前听到、或遇到的问题：

1. 如何判断指针所指向的一块数据是在堆、还是栈上？

> 在不进行思维发散的前提下，堆的地址要小于当前栈空间局部变量地址，基于这个原理就能写出验证代码。

```cpp
struct buffer
{
	char data[1024];
};

// `p`是函数局部变量，指向一块大小为1K的内存。
// void *p = new buffer();
if (p < &p) {
	printf("堆地址\n");
} else {
	printf("栈地址\n");
}
```

2. 为什么函数调用需要预先分配栈空间而不是动态分配？

> 其实这个问题也可以理解成，为什么需要栈，内存不能都由堆来管理吗？某种程序上讲并非不可能，只是实现上比较复杂。因为堆空间在当前进程上下文共享，对于多线程环境而言是全局的，这就要求对它的访问必须是线程安全的；除此之外还有内存碎片化的管理等复杂问题。针对堆、栈的分配及速度等更多讨论，在*Stack Overflow*上面*Jeff Hill*给出了非常好的答案（请戳[这里](https://stackoverflow.com/questions/79923/what-and-where-are-the-stack-and-heap/80113#80113)）。

3. 协程原理？

> 对于C/C++这类传统语言而言协议切换一般在线程内发生（而非线程/进程间），而且操作系统是感知不到的（没有专门用于切换协程的系统调用，没有用户态、内核态切换，只发生在用户态），它就是一次简单的函数调用，只不过栈空间可能被预先分配在固定大小的堆内存上面，而且在栈空间还会记录各协程的寄存器上下文（比如[libco](https://github.com/Tencent/libco)的实现）。

参考：

- [x86 Wiki](https://en.wikipedia.org/wiki/X86)
- [x86 Calling Conventions](https://en.wikipedia.org/wiki/X86_calling_conventions)
- [x86 Assembly Guide](http://www.cs.virginia.edu/~evans/cs216/guides/x86.html)
- [x86 Instruction Set Reference](http://www.felixcloutier.com/x86/)
- [Using Assembly Language in Linux](http://asm.sourceforge.net/articles/linasm.html)
