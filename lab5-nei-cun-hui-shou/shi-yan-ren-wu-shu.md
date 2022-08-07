# 实验任务书

## **给定应用**

* user/app\_naive\_malloc.c

<pre class="language-c"><code class="lang-c"><strong>/*
</strong> * Below is the given application for lab2_2.
 */

#include "user_lib.h"
#include "../util/types.h"

struct my_structure {
  char c;
  int n;
};

int main(void) {
  struct my_structure* s = (struct my_structure*)naive_malloc();
  s->c = 'a';
  s->n = 1;

  printu("s: %lx, {%c %d}\n", s, s->c, s->n);

  naive_free(s);
  exit(0);
}
</code></pre>

该应用的逻辑非常简单：首先分配一个空间（内存页面）来存放my\_structure结构，往my\_structure结构的实例中存储信息，打印信息，并最终将之前所分配的空间释放掉。这里，新定义了两个用户态函数naive\_malloc()和naive\_free()，它们最终会转换成系统调用，完成内存的分配和回收操作。

从输出结果来看，`s: 0000000000400000, {a 1}`的输出说明分配内存已经做好（也就是说naive\_malloc函数及其内核功能的实现已完成），且打印出了我们预期的结果。但是，naive\_free对应的功能并未完全做好。

## **实验内容**

如输出提示所表明的那样，需要完成naive\_free对应的功能，并获得以下预期的结果输出：

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
    PKE kernel start 0x0000000080020000, PKE kernel end: 0x0000000080028000, PKE kernel size: 0x0000000000008000 .
    free physical memory address: [0x0000000080028000, 0x000000008011ffff]
    kernel memory manager is initializing ...
    KERN_BASE 0x0000000080020000
    physical address of _etext is: 0x0000000080025000
    kernel page table is on
    User application is loading.
    user frame 0x000000008011b000, user stack 0x000000007ffff000, user kstack 0x000000008011a000
    Application program entry point (virtual address): 0x0000000080020f28
    Switch to user mode...
    s: 0000000000400000, {a 1}
    User exit with code:0.
    System is shutting down with exit code 0.
    [rustsbi] todo: shutdown all harts on k210; program halt  
```
