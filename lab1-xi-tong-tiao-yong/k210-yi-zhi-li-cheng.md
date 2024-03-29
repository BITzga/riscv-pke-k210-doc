# K210移植历程

## 0 引入RustSBI

### 背景

移植K210时，PKE需要依赖SBI（Supervisor Binary Interface）提供BOOTLOADER和RUNTIME功能，所以烧录内核时需要带上SBI固件。通过调研发现，OpenSBI与RustSBI（用Rust语言实现的SBI）均按照SBI标准实现。这两种也是业内使用最多的开源SBI。qemu就是用了OpenSBI为RISC-V提供了环境支持。此外，RustSBI还对K210做了特殊支持。所以目前暂定使用RustSBI当作SBI固件，为PKE提供BOOTLOADER功能和RUNTIME运行时服务。移植过程中我们不需要关心[RustSBI的具体实现](https://github.com/rustsbi/rustsbi)，只需要根据SBI标准调用其接口即可。

RustSBI在K210兼容了高版本的指令。K210实现的RISC-V指令集是1.9.1标准的。目前最新的特权级标准已经达到1.11。如果我们的内核代码里有用到更高级的RISC-V汇编指令，可能会在K210上无法运行。这种情况下，就要改动内核的代码，会带来许多工作量。因此，使用RustSBI可以使我们免去处理RISC-V汇编版本的麻烦。

除此之外，引入SBI，可以便于内核在其他板子上运行。当更换板子时，不需要改变内核代码，只需要更改RustSBI实现代码即可。这也简化了后续内核在其他芯片的移植工作。

## 1 接口移植

### 1.1 串口实现&格式化输出实现

串口实现: 引入了RustSBI，我们很容易实现串口输出。只需要调用SBI提供的服务即可。

```c
uint64 SBI_CONSOLE_PUTCHAR = 1;
uint64 sbi_call(uint64 sbi_type, uint64 arg0, uint64 arg1, uint64 arg2) {
    uint64 ret_val;
    __asm__ volatile (
    "mv x17, %[sbi_type]\n"
    "mv x10, %[arg0]\n"
    "mv x11, %[arg1]\n"
    "mv x12, %[arg2]\n"
    "ecall\n"
    "mv %[ret_val], x10"
    : [ret_val] "=r"(ret_val)
    : [sbi_type] "r"(sbi_type), [arg0] "r"(arg0), [arg1] "r"(arg1), [arg2] "r"(arg2)
    : "memory"
    );
    return ret_val;
}
void sbi_console_putchar(unsigned char ch) {
    sbi_call(SBI_CONSOLE_PUTCHAR, ch, 0, 0);
}
```

格式化输出实现：有了串口输出的方法，接下来我们需要格式化输出sprint。sprint定义在spike\_interface/spike\_utils.c 接下来我们看看sprint的代码。

```c
void sprint(const char *s, ...) {
    va_list vl;
    va_start(vl, s);
    vprintk(s, vl);
    va_end(vl);
}
```

sprint函数的第一个参数对应了一个字符串的起始地址，第二个参数...代表可变参数。接下来我们点开vprintk的实现：

```c
void vprintk(const char* s, va_list vl) {
  char out[256];
  int res = vsnprintf(out, sizeof(out), s, vl);
  //you need spike_file_init before this call
  spike_file_write(stderr, out, res < sizeof(out) ? res : sizeof(out));
}
```

通过阅读代码发现，vsnprintf并没有将字符串真正输出到控制台。而是根据原先的字符串和参数做字符串格式化，将最终结果保存在out数组中。 真正将字符串打印的函数调用是spike\_file\_write。这个函数是调用了spike的接口，通过spike去调用Linux的字符串打印API。所以我们需要在K210上实现串口输出，方案已经很明显，就是将spike\_file\_write函数替换成sbi\_console\_putchar实现的打印函数cputs。

```c
void vprintk(const char *s, va_list vl) {
    char out[256];
    int res = vsnprintf(out, sizeof(out), s, vl);
    cputs(out);
}
/* *
 * cputs- writes the string pointed by @str to stdout and
 * appends a newline character.
 * */
int cputs(const char *str) {
    int cnt = 0;
    char c;
    while ((c = *str++) != '\0') {
        cputch(c, &cnt);
    }
    cputch('\n', &cnt);
    return cnt;
}
```

至此，我们完成了串口实现和格式化输出。通过在K210上验证，我们的sprint可以通过串口，在控制台上输出格式化的字符串。

### 1.2 shutdown\&poweroff函数实现

内核在编码调试过程中，需要借助一些方法来判断变量值是否符合预期，如assert方法，如果不符合预期需要打印错误信息，并且让内核panic。那么panic在pke上是如何实现的呢？我们可以阅读pke实现panic的代码。

```c
void do_panic(const char *s, ...) {
    va_list vl;
    va_start(vl, s);
    sprint(s, vl);
    shutdown(-1);
    va_end(vl);
}

void shutdown(int code) {
  sprint("System is shutting down with exit code %d.\n", code);
  frontend_syscall(HTIFSYS_exit, code, 0, 0, 0, 0, 0, 0);
  while (1)
    ;
}
```

通过观察我们可以发现，panic的调用链路是：do\_panic->shutdown->frontend\_syscall 最终panic是通过frontend\_syscall调用spike提供的接口实现的。既然是需要调用spike，与spike具有依赖关系，那么我们移植的时候就需要去除相关依赖，自行实现do\_panic函数。

通过观察panic和poweroff功能，很容易发现，他们都是打印报错信息，然后终止了硬件线程（hart）。那么我们可以通过调用SBI的shutdown接口来实现这两个接口，打印报错以后以后，终止所有的hart。

## 2 编译流程改造

### 2.1 内存布局改造

由于我们引入了RustSBI，RustSBI需要占用0x80000000-0x8001FFFF的物理内存空间。所以，内核的程序入口点由此发生了变化。我们需要修改内核lds文件，更改了程序的入口点以保证内核可以正常运行。

首先，我们现在mentry.S中加入以下两行，用以确保\_mentry是内核的程序入口点。

```nasm
.globl _mentry
.section .text.prologue, "ax"
_mentry:
```

确定了内核的程序入口点，还需要把程序入口点的地址设置为0x80020000，这需要我们对内存布局进行改造。修改BASE\_ADDRESS，赋值为0x80020000.并设置代码段的起始地址为BASE\_ADDRESS，自此，内存布局就修改完成了。

```nasm
OUTPUT_ARCH(riscv)
ENTRY(_mentry)
BASE_ADDRESS = 0x80020000;
SECTIONS
{
  /*--------------------------------------------------------------------*/
  /* Code and read-only segment                                         */
  /*--------------------------------------------------------------------*/

  /* Begining of code and text segment, starts from DRAM_BASE to be effective before enabling paging */
  . = BASE_ADDRESS;
  
  /* text: Program code section */
  .text : 
  {
    stext = .;
    *(.text.prologue);
    *(.text .stub .text.* .gnu.linkonce.t.*);
    . = ALIGN(0x1000);
    ......
   }
```

### 2.2 MakeFile改造

由于我们需要引入RustSBI，并需要将其打包进入内核。除此之外，还需要将内核烧录到K210，并与K210进行串口通讯。现有的Makefile并不支持这些工作。因此，我们需要修改MakeFile。

```makefile
k210: $(KERNEL_K210_TARGET)
   $(PYTHON) compile_tool/kflash.py -p $(PORT) -b 1500000 $(KERNEL_K210_TARGET)
   $(TERM) --eol LF --dtr 0 --rts 0 --filter direct $(PORT) 115200
```

整体的编译流程为： **1.打包内核**

```makefile
$(KERNEL_K210_TARGET): $(KERNEL_TEMP_TARGET) $(BOOTLOADER)
   $(COPY) $(BOOTLOADER) $@
   $(V)dd if=$(KERNEL_TEMP_TARGET) of=$@ bs=128K seek=1
```

以上步骤是为了把rust-sbi.bin和pke.img打包成kernel.img。bs=128k意味着输入/输出的block大小为128k，seek=1意味着跳过第零个block进行复制操作。也就是说，内核镜像里第零个block存放着rust-sbi.bin，第一个block才开始存放pke.img。128k对应着十六进制0x20000，也就是二进制的‭0010 0000 0000 0000 0000‬。 我们的kernel.img放置在0x80000000处，再加上以上原因，pke的地址自然就是0x8020000。所以，我们需要在链接脚本kernel.lds里指定内核起始地址为0x8020000，这也相当于告诉SBI这是内核的起始地址。当SBI在行使bootloader的功能时，会跳转到0x8020000，将控制权转接给内核。

**2.烧录**

用数据线将K210与上位机连接，再使用kflash，指定好相关参数即可完成烧录。

```makefile
$(PYTHON) compile_tool/kflash.py -p $(PORT) -b 1500000 $(KERNEL_K210_TARGET)
```

**3.运行**

```makefile
$(TERM) --eol LF --dtr 0 --rts 0 --filter direct $(PORT) 115200
```

打开miniterm,接收K210的串口打印输出。 **4.总结**

最小可执行内核在K210的运行流程是：

指定内核起始地址->打包完整内核镜像->

烧录到flash->引导程序加载和运行rust-sbi->

rust-sbi运行并跳转到指定地址->

控制权交接给内核

## 3 内核启动流程改造

### 3.1 加载用户程序

在原先pke的中，是通过调用spike接口，进而调用linux的文件系统接口来加载用户程序的。而在K210上，我们没有spike的环境支持，不能直接调用spike接口。因此，加载用户程序就需要自行实现文件系统，或者使用其他办法。

由于文件系统的实现较为繁琐，其工作量会阻塞这个移植进度。因此，我们暂时不实现文件系统，而是采用获取用户程序地址，再加载的办法来实现这个需求。

具体的做法是：

1. 将用户程序、内核和RustSBI一起编译打包到kernel.img
2. 使用objdump命令查找到用户程序main函数的地址
3. 得到地址以后，把地址的值赋值到内核加载用户程序处 通过这种技术方案，我们可以用较低的开发成本实现用户程序加载。

### 3.2 内核程序入口点修改

由于RustSBI已经运行在M态，并且为我们提供了许多运行时服务。有了RustSBI，在K210上，pke运行在M态会破坏RustSBI的设计，因此pke只需要运行在S态即可。

这样，我们就可以直接将pke的M态代码根据自身需求迁移到S态代码。

迁移完成后，需要更改内核程序入口点至S态入口

mentry.S

```nasm
call s_start
```

## 4 其他问题

### 指令异常原因\&RustSBI处理

烧录验证时发现指令异常（U-mode使用CSR相关的指令）被M层的RustSBI处理了。而改造实验的预期是在S层处理指令异常。通过查阅RISC-V手册发现，CSR相关指令只能在M层运行，在U层访问CSR会引起指令异常。而异常处理一般在M层，也可以通过修改medeleg寄存器，将异常处理托管给S层的pke。
