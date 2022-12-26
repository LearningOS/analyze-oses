# Actandy OS

### 系统代码分析

内核文件的链接脚本：`src/arch/riscv64/boot/linker.ld`
内核入口函数：`src/arch/riscv64/boot/entry.rs::_start`
内核编译时，会通过调用构建脚本`build.rs`来自动生成`src/link_app.S`，生成的该文件用于定义一些常量和内核ELF的`.data`数据段等，以包含用户态应用程序到内核ELF文件中。

RISCV64内核初始化部分的代码，与zCore的初始化过程类似。
<br>先初始化一个主核，其中通过调用SBI的`HART_START`的方式来启动其他的核心；主核进行大部分的初始化工作，包括初始化内核打印功能，构建堆和虚拟内存，初始化中断，通过解析DTB设备树的方式调用众多的硬件外设驱动，加载用户态应用程序，文件系统初始化，进程执行环境的初始化`task::init`;

<br>这里主要分析线程与协程有关的代码逻辑。

`ROOT_PROC`是根进程，最早初始化，其他进程都属于它的子进程；进程会载入用户态ELF可执行程序，进程结构体会保存线程结构体信息;

`Thread` 一个线程属于一个CPU执行单元。故负责初始化`TrapFrame`的执行上下文；

`spawn()`
`run_until_idle()`
`future/tests.rs`

`future::init` --> `future/executor.rs::init`
<br>Future执行器的初始化：
* `Executor`结构体通过`VecDeque`来保存`Task`;
* `CPUS`全局静态数组中保存所有CPU核的信息，包括运行状态，当前线程，执行器结构体`Executor`；
* `Executor::spawn`  对future轮询并执行；
  

<br>
更多文档及代码注释等请见：<br>
https://github.com/ActandyOS/actandy/commits/comment-zyr-dev

---

### 快速运行

1. 编译用户态程序
<br>用户态程序包括`C`和`Rust`两种语言；
<br>之后会生成`rootfs`用于根文件系统的文件夹
```
cd actandy/user
make
```

2. 创建文件系统
<br>文件系统基于`Actandy FS`;
<br>之后会生产根文件系统镜像`rootfs/riscv64/fs.img`
```
cd actandy/os
make initfs
```

3. 启动系统并运行内核测试用例
<br>线程与协程的测试
```
cd actandy/os
make run KTESTS=1 DEBUG=1

# 包括的内核测试用例输出如下：

[INFO  0 0 0] basic tests starts
test kernel task: tid = 10, arg = 0xdead
test kernel task: tid = 11, arg = 0xbeef
[INFO  0 0 0] basic tests over
[INFO  0 0 0] mutex tests starts
[INFO  0 0 0] mutex tests over
[INFO  0 0 0] park mutex tests starts
[INFO  0 0 0] Kernel Procs States: [
[INFO  0 0 0] 	proc[13] name: "park test 0" state: "Normal"
[INFO  0 0 0] 		thread[13] state: "Running"
[INFO  0 0 0] 	proc[14] name: "park test 1" state: "Normal"
[INFO  0 0 0] 		thread[14] state: "Ready"
[INFO  0 0 0] ]
```

   
