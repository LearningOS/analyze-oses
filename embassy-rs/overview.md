# embassy-analyze

Embassy 是下一代的嵌入式应用框架。使用 Rust 编程语言、其异步设施和 Embassy 库，可以更快地编写安全、正确和节能的嵌入式代码。

Embassy 项目由几个 crates 组成，你可以一起使用，也可以独立使用。

没有 async 的软件可能会在I/O操作上阻塞。在 std 环境下，如 PC，软件可以通过使用线程或非阻塞操作来处理这个问题。

有了线程，一个线程在 I/O 操作上阻塞，另一个线程就能取代它的位置。然而，即使在 PC 上，线程也是相对较重的，因此，一些编程语言，如Go，已经实现了一个叫做 coroutines 或 goroutines 的概念，它比线程要轻得多，而且不那么密集。

处理阻塞式 I/O 操作的另一种方法是支持轮询底层外设的状态，以检查它是否可以执行所请求的操作。在没有内置异步支持的编程语言中，这需要建立一个复杂的循环来检查事件。

在 Rust 中，非阻塞操作可以用 async-await 来实现，使嵌入式系统中的多任务处理变得前所未有的简单和高效。Async-await 工作方式是将每个异步函数转化为一个叫做 future 的对象。当一个 future 在 I/O 上阻塞时，future 就会产生，而被称为执行者的调度器可以选择一个不同的 future 来执行。与 RTOS 等替代方案相比，async 可以产生更好的性能和更低的功耗，因为执行者不需要猜测一个未来何时可以执行。然而，程序的大小可能比其他替代方案要高，这对某些空间受限、内存非常小的设备来说可能是个问题。在 Embassy 支持的设备上，比如 stm32 和 nrf，内存一般都足够大，可以容纳适度增加的程序大小。

* Hardware Abstraction Layers

硬件抽象 - HAL 实现了安全、惯性的 Rust API 来使用硬件能力，因此不需要对原始寄存器进行操作。Embassy 项目为特定的硬件维护 HAL，但是你仍然可以用 Embassy 来使用其他项目的 HAL。

* embassy-stm32

用于所有 STM32 微控制器系列。

* embassy-nrf

用于北欧半导体 nRF52, nRF53, nRF91 系列。

* Time that Just Works

embassy_time 提供全局可用的即时、持续时间和定时器类型，并且永不溢出。

* Real-time ready

实时--同一异步执行器上的任务合作运行，但你可以创建多个具有不同优先级的执行器，这样高优先级的任务就会优先于低优先级的。

* Low-power ready

低功耗--轻松构建具有多年电池寿命的设备。当没有工作要做时，异步执行器会自动使内核进入睡眠状态。任务被中断唤醒，等待时没有忙环轮询。

* Networking

网络 - embassy-net 网络栈实现了广泛的网络功能，包括以太网、IP、TCP、UDP、ICMP 和 DHCP。Async 极大地简化了管理超时和同时为多个连接提供服务。

* Bluetooth

蓝牙 - nrf-softdevice crate 为 nRF52 微控制器提供蓝牙低能耗 4.x 和 5.x 支持

* LoRa

embassy-lora 支持 STM32WL 无线微控制器和 Semtech SX126x 和 SX127x 收发器的 LoRa 网络。

* USB

embassy-usb 实现了一个设备端 USB 堆栈。常见的类如 USB 串口（CDC ACM）和 USB HID 的实现是可用的，丰富的构建器 API 允许建立你自己的。

* Bootloader and DFU

embassy-boot 是一个轻量级的 Bootloader，支持以断电安全的方式进行固件应用升级，可试运行和回滚。
