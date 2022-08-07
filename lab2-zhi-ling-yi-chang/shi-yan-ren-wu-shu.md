# 实验任务书

## 背景

### **给定应用**

* user/app\_illegal\_instruction.c

```c
/*
 * Below is the given application for lab1_2.
 * This app attempts to issue M-mode instruction in U-mode, and consequently raises an exception.
 */

#include "user_lib.h"
#include "../util/types.h"

int main(void) {
  printu("Going to hack the system by running privilege instructions.\n");
  // we are now in U(user)-mode, but the "csrw" instruction requires M-mode privilege.
  // Attempting to execute such instruction will raise illegal instruction exception.
  asm volatile("csrw sscratch, 0");
  exit(0);
}

```

（在用户U模式下执行的）应用企图执行RISC-V的特权指令`csrw sscratch, 0`。该指令会修改S模式的栈指针，如果允许该指令的执行，执行的结果可能会导致系统崩溃。

显然，这种企图破坏了RISC-V机器以及操作系统的设计原则，对于机器而言该事件并不是它期望（它期望在用户模式下执行的都是用户模式的指令）发生的“异常事件”，需要介入和破坏该应用程序的执行。查找RISC-V体系结构的相关文档，我们知道，这类异常属于非法指令异常，即CAUSE\_ILLEGAL\_INSTRUCTION，它对应的异常码是02（见kernel/riscv.h中的定义）。那么，当RISC-V机器截获了这类异常后，该将它交付给谁来处理呢？

我们知道PKE操作系统内核在启动时会将部分异常和中断“代理”给S模式处理，但是它是否将CAUSE\_ILLEGAL\_INSTRUCTION这类异常也进行了代理呢？这就要研究M层程序的实现了。

由于在一开始我们引入了RustSBI，并使其成为我们的M态程序。我们需要阅读RustSBI源码，判断其是否将该异常代理给我们S态的内核。

## 实验要求

### 1.阅读RustSBI-K210源码，梳理RustSBI代理的异常与中断类型。

[﻿https://github.com/rustsbi/rustsbi-k210](https://github.com/rustsbi/rustsbi-k210)

### 2.切换到k210/lab1\_2\_exception分支，使用`make k210`将内核代码编译、烧录到k210开发板上。观察实验效果，并验证你的梳理是否与RustSBI的异常/中断代理一致。

\
