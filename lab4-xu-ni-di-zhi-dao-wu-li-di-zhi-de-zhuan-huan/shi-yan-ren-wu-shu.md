# 实验任务书

**给定应用**

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

**实验内容**

实现user\_va\_to\_pa()函数，完成给定逻辑地址到物理地址的转换，并获得以下预期结果：

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

**实验指导**

读者可以参考lab1\_1的内容，重走从应用的printu到S态的系统调用的完整路径，最终来到kernel/syscall.c文件的sys\_user\_print()函数：

```c
 21 ssize_t sys_user_print(const char* buf, size_t n) {
 22   //buf is an address in user space on user stack,
 23   //so we have to transfer it into phisical address (kernel is running in direct mapping).
 24   assert( current );
 25   char* pa = (char*)user_va_to_pa((pagetable_t)(current->pagetable), (void*)buf);
 26   sprint(pa);
 27   return 0;
 28 }
```

\
该函数最终在第26行通过调用sprint将结果输出，但是在输出前，需要将buf地址转换为物理地址传递给sprint，这一转换是通过user\_va\_to\_pa()函数完成的。而user\_va\_to\_pa()函数的定义在kernel/vmm.c文件中定义：

```c
150 void *user_va_to_pa(pagetable_t page_dir, void *va) {
151   // TODO (lab2_1): implement user_va_to_pa to convert a given user virtual address "va"
152   // to its corresponding physical address, i.e., "pa". To do it, we need to walk
153   // through the page table, starting from its directory "page_dir", to locate the PTE
154   // that maps "va". If found, returns the "pa" by using:
155   // pa = PYHS_ADDR(PTE) + (va - va & (1<<PGSHIFT -1))
156   // Here, PYHS_ADDR() means retrieving the starting address (4KB aligned), and
157   // (va - va & (1<<PGSHIFT -1)) means computing the offset of "va" in its page.
158   // Also, it is possible that "va" is not mapped at all. in such case, we can find
159   // invalid PTE, and should return NULL.
160   panic( "You have to implement user_va_to_pa (convert user va to pa) to print messages in lab2_1.\n" );
161
162 }
```

如注释中的提示，为了在page\_dir所指向的页表中查找逻辑地址va，就必须通过调用页表操作相关函数找到包含va的页表项（PTE），通过该PTE的内容得知va所在的物理页面的首地址，最后再通过计算va在页内的位移得到va最终对应的物理地址。
