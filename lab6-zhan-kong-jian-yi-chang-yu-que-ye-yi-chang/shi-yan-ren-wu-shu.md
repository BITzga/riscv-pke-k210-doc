# 实验任务书

## **给定应用**

* user/app\_sum\_sequence.c

```c
/*
 * The application of lab2_3.
 */

#include "user_lib.h"
#include "../util/types.h"

//
// compute the summation of an arithmetic sequence. for a given "n", compute
// result = n + (n-1) + (n-2) + ... + 0
// sum_sequence() calls itself recursively till 0. The recursive call, however,
// may consume more memory (from stack) than a physical 4KB page, leading to a page fault.
// PKE kernel needs to improved to handle such page fault by expanding the stack.
//
uint64 sum_sequence(uint64 n) {
  if (n == 0)
    return 0;
  else
    return sum_sequence( n-1 ) + n;
}

int main(void) {
  // we need a large enough "n" to trigger pagefaults in the user stack
  uint64 n = 1000;

  printu("Summation of an arithmetic sequence from 0 to %ld is: %ld \n", n, sum_sequence(n) );
  exit(0);
}

```



给定一个递增的等差数列：`0, 1, 2, ..., n`，如何求该数列的和？以上的应用给出了它的递归（recursive）解法。通过定义一个函数sum\_sequence(n)，将求和问题转换为sum\_sequence(n-1) + n的问题。问题中n依次递减，直至为0时令sum\_sequence(0)=0。

* （先提交lab2\_2的答案，然后）切换到lab2\_3、继承lab2\_2及以前所做修改，并make后的直接运行结果：

```
In m_start, hartid:0
HTIF is available!
(Emulated) memory size: 2048 MB
Enter supervisor mode...
PKE kernel start 0x0000000080000000, PKE kernel end: 0x000000008000e000, PKE kernel size: 0x000000000000e000 .
free physical memory address: [0x000000008000e000, 0x0000000087ffffff]
kernel memory manager is initializing ...
KERN_BASE 0x0000000080000000
physical address of _etext is: 0x0000000080004000
kernel page table is on
User application is loading.
user frame 0x0000000087fbc000, user stack 0x000000007ffff000, user kstack 0x0000000087fbb000
Application: ./obj/app_sum_sequence
Application program entry point (virtual address): 0x0000000000010096
Switching to user mode...
handle_page_fault: 000000007fffdff8
You need to implement the operations that actually handle the page fault in lab2_3.

```

以上执行结果，为什么会出现handle\_page\_fault呢？这就跟我们给出的应用程序（递归求解等差数列的和）有关了。

递归解法的特点是，函数调用的路径会被完整地保存在栈（stack）中，也就是说函数的下一次调用会将上次一调用的现场（包括参数）压栈，直到n=0时依次返回到最开始给定的n值，从而得到最终的计算结果。显然，在以上计算等差数列的和的程序中，n值给得越大，就会导致越深的栈，而栈越深需要的内存空间也就越多。

通过4.1.3节中对用户进程逻辑地址空间的讨论，以及图4.5的图示，我们知道应用程序最开始被载入（并装配为用户进程）时，它的用户态栈空间（栈底在0x7ffff000，即USER\_STACK\_TOP）仅有1个4KB的页面。显然，只要以上的程序给出的n值“足够”大，就一定会“压爆”用户态栈。而以上运行结果中，出问题的地方（即handle\_page\_fault后出现的地址，0x7fffdff8）也恰恰在用户态栈所对应的空间。

以上分析表明，之所以运行./obj/app\_sum\_sequence会出现错误（handle\_page\_fault），是因为给sum\_sequence()函数的n值太大，把用户态栈“压爆”了。

## **实验内容**

在PKE操作系统内核中完善用户态栈空间的管理，使得它能够正确处理用户进程的“压栈”请求。

实验完成后的运行结果：

```
    .______       __    __      _______.___________.  _______..______   __
    |   _  \     |  |  |  |    /       |           | /       ||   _  \ |  |
    |  |_)  |    |  |  |  |   |   (----`---|  |----`|   (----`|  |_)  ||  |
    |      /     |  |  |  |    \   \       |  |      \   \    |   _  < |  |
    |  |\  \----.|  `--'  |.----)   |      |  |  .----)   |   |  |_)  ||  |
    | _| `._____| \______/ |_______/       |__|  |_______/    |______/ |__|
    [rustsbi] Implementation: RustSBI-K210 Version 0.0.2
    [rustsbi] misa: RV64ACDFIMSU
    [rustsbi] mideleg: ssoft, stimer (0x22)
    [rustsbi] medeleg: ima, bkpt, uecall (0x109)
    [rustsbi] enter supervisor 0x80020000
    Enter supervisor mode...
    PKE kernel start 0x0000000080020000, PKE kernel end: 0x0000000080028000, PKE kernel size: 0x0000000000008000 .
    free physical memory address: [0x0000000080028000, 0x000000008011ffff]
    kernel memory manager is initializing ...
    KERN_BASE 0x0000000080020000
    physical address of _etext is: 0x0000000080025000
    kernel page table is on
    User application is loading.
    user frame 0x000000008011b000, user stack 0x000000007ffff000, user kstack 0x000000008011a000
    Application program entry point (virtual address): 0x0000000080020fc2
    Switch to user mode...
    handle_page_fault: 000000007fffdff8
    handle_page_fault: 000000007fffcff8
    handle_page_fault: 000000007fffbff8
    Summation of an arithmetic sequence from 0 to 1000 is: 500500
    User exit with code:0.
    System is shutting down with exit code 0.
    [rustsbi] reset triggered! todo: shutdown all harts on k210; program halt. Type: 0, reason: 0    
```
