# 实验任务书

## 背景

### **给定应用**

* user/app\_helloworld\_no\_lds.c

```c
/*
 * Below is the given application for lab2_1.
 * This app runs in its own address space, in contrast with in direct mapping.
 */

#include "user_lib.h"
#include "util/types.h"

int main(void) {
 printu("Hello world!\n");
 exit(0);
}

```

该应用的代码跟lab1\_1是一样的。但是，不同的地方在于，它的编译和链接并未指定程序中符号的逻辑地址。

从以上运行结果来看，我们的应用app\_helloworld\_no\_lds并未如愿地打印出“Hello world!\n”，这是因为user/app\_helloworld\_no\_lds.c的第10行`printu("Hello world!\n");`中的“Hello world!\n”字符串本质上是存储在.rodata段，它被和代码段（.text）一起被装入内存。从逻辑地址结构来看，它的逻辑地址就应该位于图4.5中的“用户代码段”，显然低于0x80020000。

而printu是一个典型的系统调用（参考lab1\_1的内容），它的执行逻辑是通过ecall指令，陷入到内核（S模式）完成到屏幕的输出。然而，对于内核而言，显然不能继续使用“Hello world!\n”的逻辑地址对它进行访问，而必须将其转换成物理地址（因为如图4.4所示，操作系统内核已建立了到“实际空闲内存”的直映射）。而lab2\_1的代码，显然未实现这种转换。

## **实验内容**

### 实现user\_va\_to\_pa()函数，完成给定逻辑地址到物理地址的转换，并获得以下预期结果：

```
    .______       __    __      _______.___________.  _______..______   __
    |   _  \     |  |  |  |    /       |           | /       ||   _  \ |  |
    |  |_)  |    |  |  |  |   |   (----`---|  |----`|   (----`|  |_)  ||  |
    |      /     |  |  |  |    \   \       |  |      \   \    |   _  < |  |
    |  |\  \----.|  `--'  |.----)   |      |  |  .----)   |   |  |_)  ||  |
    | _| `._____| \______/ |_______/       |__|  |_______/    |______/ |__|
    
    [rustsbi] Platform: K210
    [rustsbi] misa: RV64ACDFIMSU
    [rustsbi] mideleg: 0x222
    [rustsbi] medeleg: 0x1ab
    [rustsbi] Kernel entry: 0x80020000
    Enter supervisor mode...
    PKE kernel start 0x0000000080020000, PKE kernel end: 0x0000000080027000, PKE kernel size: 0x0000000000007000 .
    free physical memory address: [0x0000000080027000, 0x000000008011ffff] 
    kernel memory manager is initializing ...
    KERN_BASE 0x0000000080020000
    physical address of _etext is: 0x0000000080024000
    kernel page table is on 
    User application is loading.
    user frame 0x000000008011b000, user stack 0x000000007ffff000, user kstack 0x000000008011a000 
    Application program entry point (virtual address): 0x0000000080020ec8
    Switch to user mode...
    Hello world!
    User exit with code:0.
    System is shutting down with exit code 0.
    [rustsbi] todo: shutdown all harts on k210; program halt  
```

****
