# 环境搭建

## 软件准备

### 0.安装编译器

step1.访问sifive官网，下载riscv gcc toolchain

[https://www.sifive.com/software](https://www.sifive.com/software)&#x20;

step2.找到Prebuilt RISC-V GCC Toolchain

如果你是在linux环境下开发，就下载Ubantu版本

step3.将下载好的tar.gz压缩包解压

```
tar -zxvf $your_tar_gz
```

step4.配置环境变量 解压完成得到文件夹，进入文件夹里的bin目录，打开terminal，输入pwd获得当前路径。

copy获得的路径。

```
vim /etc/profile
把刚刚copy的路径加入系统的PATH环境变量

export RISCV=你的路径

export PATH=$PATH:$RISCV
```

step5.加载环境变量文件

```
source /etc/profile
```

step6.验证 在终端输入`riscv64-unknown-elf-gcc -v`查看安装的gcc版本, 如果输出一大堆东西且最后一行有`gcc version 某个数字.某个数字.某个数字`，说明gcc配置成功，否则需要检查一下哪里做错了，比如环境变量**PATH**配置是否正确。一般需要把一个形如`..../bin`的目录加到**PATH**里。

step7.配置terminal启动脚本

```
vim ~/.bashrc 
```

加入一行

```
source /etc/profile
```

保存退出，以后每次打开终端都不用手动加载环境变量文件了

### 1.代码准备

step1.下载uCoreRV64代码库

```
git clone https://github.com/NKU-EmbeddedSystem/riscv64-ucore.git
```

step2.使用git branch -a 查看本地和远程分支

```
PS D:\毕设\code\riscv64-ucore> git branch -a                                                                            * k210-lab0
  k210-lab5
  master
  remotes/origin/HEAD -> origin/master
  remotes/origin/k210-lab0
  remotes/origin/lab0
  remotes/origin/lab1
  remotes/origin/master
  remotes/origin/tutorial
```

step3.切换分支 这里要根据需求切换，如果是需要跑k210的代码，就切换k210前缀的分支就好。

如果是在qemu上跑，就lab0-lab8就好。

如果本地没有lab0分支：

```
PS D:\毕设\code\riscv64-ucore> git checkout -b lab0 origin/lab0
Switched to a new branch 'lab0'
Branch 'lab0' set up to track remote branch 'lab0' from 'origin'.
PS D:\毕设\code\riscv64-ucore> git branch
  k210-lab0
  k210-lab5
* lab0
  master
```

如果本地有lab0分支 直接git checkout lab0就好：

```
git checkout  lab0 
```

### K210篇

step1.安装python3

step2.安装pip3

step3.使用数据线连接电脑USB和K210

```
查看设备 观察是否连接成功（一般出现/dev/ttyUSB0，就是成功）
```

lumin@lumin:\~$ ls /dev/ttyUSB\* /dev/ttyUSB0

```
```

step4.我们想要与k210通信，除了驱动，还需要一个类似终端的东西来进行输入/输出，也就是串口终端工具miniterm

```
```

sudo apt install python3-serial

```
然后就可以连接设备，用miniterm与之通信了
```

step5.进入代码文件夹，打开terminal，输入make k210，os内核就可以被编译、打包、烧录，然后运行在k210上。

