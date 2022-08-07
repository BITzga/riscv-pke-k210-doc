# 实验任务书

## 背景

### **给定应用**

* user/app\_long\_loop.c

```c
/*
 * Below is the given application for lab1_3.
 * This app performs a long loop, during which, timers are
 * generated and pop messages to our screen.
 */

#include "user_lib.h"
#include "util/types.h"

int main(void) {
  printu("Hello world!\n");
  int i;
  for (i = 0; i < 100000000; ++i) {
    if (i % 5000000 == 0) printu("wait %d\n", i);
  }

  exit(0);

  return 0;
}

```

## **实验内容**

### 完成PKE操作系统内核未完成的时钟中断处理过程，使得它能够完整地处理时钟中断。

### 具体要求：

实验操作者需要初始化时钟中断， 设置定时器中断的时间间隔， 并且编写时钟中断处理程序。 最终让用户程序执行循环的同时， 定时打印出自增的次数。

### 实验完成后的运行结果：

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
    
    ++ setup timer interrupts
    
    Application program entry point (virtual address): 0x000000008002091a
    
    Switch to user mode...
    
    wait 0
    wait 5000000
    wait 10000000
    wait 15000000
    wait 20000000
    wait 25000000
    wait 30000000
    Ticks 1
    wait 35000000
    wait 40000000
    wait 45000000
    wait 50000000
    wait 55000000
    wait 60000000
    Ticks 2
    .....
    Ticks 3
    wait 90000000
    wait 95000000
    
    User exit with code:0.
    System is shutting down with exit code 0.
    [rustsbi] todo: shutdown all harts on k210; program halt
       
```



