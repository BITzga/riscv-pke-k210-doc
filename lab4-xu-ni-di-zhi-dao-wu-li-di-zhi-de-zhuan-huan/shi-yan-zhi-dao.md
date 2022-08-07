# 实验指导

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
