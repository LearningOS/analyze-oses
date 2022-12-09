# Aero BackTrace 分析

> Aero BackTrace

```shell
unwind.rs:180 (tid=4, pid=4) error cpu '0' panicked at 'called `Result::unwrap()` on an `Err` value: EntryNotFound'
unwind.rs:184 (tid=4, pid=4) error aero_kernel/src/main.rs:163:21
unwind.rs:188 (tid=4, pid=4) error 
unwind.rs:113 (tid=4, pid=4) trace ---------------------------------- BACKTRACE -----------------------------------
unwind.rs:150 (tid=4, pid=4) trace  0: 0xffffffff80078922 - rust_begin_unwind
unwind.rs:150 (tid=4, pid=4) trace  1: 0xffffffff800aaf73 - core::panicking::panic_fmt::ha223941c3a41d72c
unwind.rs:150 (tid=4, pid=4) trace  2: 0xffffffff800ad825 - core::result::unwrap_failed::h308206dbe673654c
unwind.rs:150 (tid=4, pid=4) trace  3: 0xffffffff80079016 - core::result::Result<T,E>::unwrap::h4b1489cc891de0a8
unwind.rs:150 (tid=4, pid=4) trace  4: 0xffffffff8007a3d1 - aero_kernel::kernel_main_thread::h43f94442fe2b1ebc
```

前两行为 `panic` 信息，在`unwind` 文件中的 `rust_begin_unwind` 函数中，`BACKTRACE` 在 `unwind_stack_trace`文件中。

## X86 常用寄存器功能

RAX特殊用途：存放运算结果，执行乘法及除法指令时，会自动使用

RBX特殊用途：基底寻址法的基底寄存器

RCX特殊用途：做循环的计数器

RDX特殊用途：数据寄存器

堆叠指位器RSP：指向Stack的顶端（Point to the top of stack）

基底指位器RBP：指向Stack的底部（Point to the bottom of stack）

索引指位器RSI、RDI

程序指位器RIP：主掌CPU动向的寄存器（放下一个指令地址）

## X86 调用栈



> 来源：https://textbook.cs161.org/memory-safety/x86.html#:~:text=x86%20function%20calls,at%20lower%20addresses%20in%20memory.

![下一个堆栈图，之前分配的 8 个字节现在已用于局部变量](https://textbook.cs161.org/assets/images/memory-safety/x86/stack7.png)

## panic Message

第一行为 `panic` ，下方给出一个相似的例子：

```rust
#[panic_handler]
fn panic_handler(info: &PanicInfo) -> ! {
    // panic message
    if let Some(message) = info.message() {
        println!("\x1b[1;31mpanic: '{}'\x1b[0m", message);
    }

    // panic location
    if let Some(location) = info.location() {
        println!("\x1b[1;31mpanic location: '{}'\x1b[0m", location);
    }
    
    println!("!TEST FINISH!");
    shutdown()
}
```

利用 `info.message` 获取错误信息，对应 `Aero BackTrace` 第一行，利用 `info.location` 获取错误位置，对应 `Aero Backtrace` 第二行。

## unwind_stack_trace

> Aero 的 BACKTRACE 使用了一些 X86 寄存器的一些特性  参考上述 X86 寄存器和 调用栈

```rust
pub fn unwind_stack_trace() {
    let _guard = IrqGuard::new();

    let mut address_space = AddressSpace::this();
    let offset_table = address_space.offset_page_table();

    let kernel_elf = &UNWIND_INFO.get().unwrap().kernel_elf;
    let mut symbol_table = None;

    // Get Symbol Table
    for section in kernel_elf.section_iter() {
        if section.get_type() == Ok(ShType::SymTab) {
            let section_data = section
                .get_data(&kernel_elf)
                .expect("Failed to get kernel section data information");

            if let SectionData::SymbolTable64(symtab) = section_data {
                symbol_table = Some(symtab);
            }
        }
    }

    let symbol_table = symbol_table.unwrap();
    let mut rbp: usize;

    unsafe {
        asm!("mov {}, rbp", out(reg) rbp);
    }

    // Make sure the RBP is not NULL. If it is then we cannot do the stack unwinding/tracing
    // as no frame pointers were emmited in this build. This should only occur if you
    // set the field `eliminate-frame-pointer` in the target file to true or manually resetting
    // the RBP to prevent backtrace to avoid address leaks (for example when jumping to userland).
    // If thats the case we return (resumes the parent function).
    if rbp == 0x00 {
        log::trace!("<empty backtrace>");
        return;
    }

    log::trace!("{:-^80}", " BACKTRACE ");

    for depth in 0../*64*/16 {
        if let Some(rip_rbp) = rbp.checked_add(core::mem::size_of::<usize>()) {
            if offset_table
                .translate_addr(VirtAddr::new(rip_rbp as u64))
                .is_none()
            {
                log::trace!("{:>2}: <guard page>", depth);
                break;
            }

            let rip = unsafe { *(rip_rbp as *const usize) };

            if rip == 0 {
                break;
            }

            unsafe {
                rbp = *(rbp as *const usize);
            }

            let mut name = None;

            for data in symbol_table {
                let st_value = data.value() as usize;
                let st_size = data.size() as usize;

                if rip >= st_value && rip < (st_value + st_size) {
                    let mangled_name = data.get_name(&kernel_elf).unwrap_or("<unknown>");
                    let demangled_name = rustc_demangle::demangle(mangled_name);

                    name = Some(demangled_name);
                }
            }

            if let Some(name) = name {
                log::trace!("{:>2}: 0x{:016x} - {}", depth, rip, name);
            } else {
                log::trace!("{:>2}: 0x{:016x} - <unknown>", depth, rip);
            }
        } else {
            // RBP has been overflowed...
            break;
        }
    }
}
```

获取符号表：

```rust
    let _guard = IrqGuard::new();

    let mut address_space = AddressSpace::this();
    let offset_table = address_space.offset_page_table();

    let kernel_elf = &UNWIND_INFO.get().unwrap().kernel_elf;
    let mut symbol_table = None;

    // Get Symbol Table
    for section in kernel_elf.section_iter() {
        if section.get_type() == Ok(ShType::SymTab) {
            let section_data = section
                .get_data(&kernel_elf)
                .expect("Failed to get kernel section data information");

            if let SectionData::SymbolTable64(symtab) = section_data {
                symbol_table = Some(symtab);
            }
        }
    }

    let symbol_table = symbol_table.unwrap();
```

获取栈顶（根据 X86 调用栈此时 rbp 指向调用函数的 rbp，rbp + 1 指向 rip， 调用函数的执行位置）：

```rust
	let mut rbp: usize;

    unsafe {
        asm!("mov {}, rbp", out(reg) rbp);
    }

    // Make sure the RBP is not NULL. If it is then we cannot do the stack unwinding/tracing
    // as no frame pointers were emmited in this build. This should only occur if you
    // set the field `eliminate-frame-pointer` in the target file to true or manually resetting
    // the RBP to prevent backtrace to avoid address leaks (for example when jumping to userland).
    // If thats the case we return (resumes the parent function).
    if rbp == 0x00 {
        log::trace!("<empty backtrace>");
        return;
    }
```

输出 BACKTRACE 字符串

```rust
    log::trace!("{:-^80}", " BACKTRACE ");
```

输出调用栈

```rust
	// 处理调用栈 深度
	for depth in 0../*64*/16 {
        // 获取 rip 指针, rbp + 1
        if let Some(rip_rbp) = rbp.checked_add(core::mem::size_of::<usize>()) {
            // 翻译地址
            if offset_table
                .translate_addr(VirtAddr::new(rip_rbp as u64))
                .is_none()
            {
                log::trace!("{:>2}: <guard page>", depth);
                break;
            }

            // 获取调用函数的 rip
            let rip = unsafe { *(rip_rbp as *const usize) };

            if rip == 0 {
                break;
            }

            // 获取调用函数的 rbp，下次迭代时 处理的是调用函数的调用栈
            unsafe {
                rbp = *(rbp as *const usize);
            }

            let mut name = None;

            // 根据调用函数的指针 获取函数的 name 后 demangle
            for data in symbol_table {
                let st_value = data.value() as usize;
                let st_size = data.size() as usize;

                if rip >= st_value && rip < (st_value + st_size) {
                    let mangled_name = data.get_name(&kernel_elf).unwrap_or("<unknown>");
                    let demangled_name = rustc_demangle::demangle(mangled_name);

                    name = Some(demangled_name);
                }
            }

            // 输出调用函数信息
            if let Some(name) = name {
                log::trace!("{:>2}: 0x{:016x} - {}", depth, rip, name);
            } else {
                log::trace!("{:>2}: 0x{:016x} - <unknown>", depth, rip);
            }
        } else {
            // RBP has been overflowed...
            break;
        }
    }
```

