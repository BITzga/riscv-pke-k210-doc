# 实验指导

一般来说，应用程序执行过程中的动态内存分配和回收，是操作系统中的堆（Heap）管理的内容。在本实验中，我们实际上是为PKE操作系统内核实现一个简单到不能再简单的“堆”。为实现naive\_free()的内存回收过程，我们需要了解其对偶过程，即内存是如何“分配”给应用程序，并供后者使用的。为此，我们先阅读kernel/syscall.c文件中的naive\_malloc()函数的底层实现，sys\_user\_allocate\_page()：

<pre class="language-c"><code class="lang-c"><strong> 43 uint64 sys_user_allocate_page() {
</strong> 44   void* pa = alloc_page();
 45   uint64 va = g_ufree_page;
 46   g_ufree_page += PGSIZE;
 47   user_vm_map((pagetable_t)current->pagetable, va, PGSIZE, (uint64)pa,
 48          prot_to_type(PROT_WRITE | PROT_READ, 1));
 49
 50   return va;
 51 }</code></pre>

这个函数在44行分配了一个首地址为pa的物理页面，这个物理页面要以何种方式映射给应用进程使用呢？第45行给出了pa对应的逻辑地址va = g\_ufree\_page，并在46行对g\_ufree\_page进行了递增操作。最后在47--48行，将pa映射给了va地址。这个过程中，g\_ufree\_page是如何定义的呢？我们可以找到它在kernel/process.c文件中的定义：

```c
 27 // start virtual address of our simple heap.
 28 uint64 g_ufree_page = USER_FREE_ADDRESS_START;
```

而USER\_FREE\_ADDRESS\_START的定义在kernel/memlayout.h文件：

```c
 17 // simple heap bottom, virtual address starts from 4MB
 18 #define USER_FREE_ADDRESS_START 0x00000000 + PGSIZE * 1024
```

* 找到一个给定va所对应的页表项PTE（查找4.1.4节，看哪个函数能满足此需求）；
* 如果找到（过滤找不到的情形），通过该PTE的内容得知va所对应物理页的首地址pa；
* 回收pa对应的物理页，并将PTE中的Valid位置为0。

以上了解了内存的分配过程后，我们就能够大概了解其反过程的回收应该怎么做了，大概分为以下步骤：

可以看到，在我们的PKE操作系统内核中，应用程序执行过程中所动态分配（类似malloc）的内存是被映射到USER\_FREE\_ADDRESS\_START（4MB）开始的地址的。\
