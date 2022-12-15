# Overview

## Cell

一个比较核心的概念，从实现和功能两个角度看分别是：

- 一个 crate，经 rustc 编译为一个 object file，装入时再进行链接。
- 一个 OS 组件，组件之间有明确定义的边界，包括装入之后内存中可执行文件中的各 section，因此一个 crate 中的 modules 不能提供这个意义下的隔离。

### nano_core

上述架构就要求启动时至少有虚拟内存、装入/链接对象文件的能力，[kernel/nano_core](https://github.com/theseus-os/Theseus/tree/theseus_main/kernel/nano_core "GitHub") 被构建为静态链接完成的一组 cell 来提供这一需求，待启动将要结束时再把 nano_core 通过 cell swapping 替换为其组成部分，让所有 cell 都是动态装入、链接的，保证组成 nano_core 的这些 cell 之后也能单独拆装。

### SAS & SPL

为了后文说的 intralingual 设计，以及统一的 cell swapping（==TODO：为什么不这样就不能 cell swapping==），我们需要保证所有 cell：

- 使用同一地址空间（single address space），这样资源使用是 rustc 可感知的，进而能够检查安全性。
  - 注：因此还要使用统一的全局分配器。
- 处于同一特权模式（single privilege level），这样所有控制流都是 rustc 可追踪的，进而可以在出错时通过 unwinding 正确释放占用的资源。

==TODO：所以**用户态**的 task 是以什么方式存在的？==



---



## Intralingual & State Spill-Free Design

这两者某种意义上是相辅相成的。前者在使用现代类型安全的语言开发的项目中应该是基本原则之一，Theseus OS 中例如内存这样的资源均由持有者保存对应的对象（这体现了 SAS 的必要性），在使用结束时通过 RAII 机制自动释放，共享资源则使用 `alloc::sync::Arc` 这样的类型表示，这一点同 rCore 实验框架类似。此外，这还让 cell 之间的交互变成了类似网站开发中前后端 RESTful 架构的风格，例如内存管理模块不再需要记录哪些内存被分配出去了，而是使用对应内存的 cell 持有代表所有权的对象，这就减少了内存管理模块的 state spill，如果需要动态更换新的内存管理模块，或者内存管理模块出错需要进行恢复，依赖它的 cell 会几乎不受影响，因为 state 都保存在它们自己手里。

注：当然，语言和硬件之间必然存在着 gap，因此 unsafe 是必然存在的，但都可以被限定到最底层贴近硬件的部分。同时，尤其是硬件已然规定好的交互部分是 clientless 的，因此必然存在一些需要 spill 的特殊 states，它们被交由 [kernel/state_store](https://github.com/theseus-os/Theseus/tree/theseus_main/kernel/state_store "GitHub") 统一管理，这样其余的 cell 就可以认为自己是 state spill-free 的了。（==TODO：这是否会导致过分依赖 state_store 的鲁棒性？==）
