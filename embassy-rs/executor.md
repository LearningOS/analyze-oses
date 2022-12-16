# 从 example 开始探索 embassy executor

embassy-rs 项目的特色在于其高度异步化——即正常使用情况下可以直接定义异步的入口和任务。

例如，从一个简单的示例 `examples/stm32f1/src/bin/hello.rs` 开始：

> 隐去了项目属性和导入。

```rust
#[embassy_executor::main]
async fn main(_spawner: Spawner) -> ! {
    let mut config = Config::default();
    config.rcc.sys_ck = Some(Hertz(36_000_000));
    let _p = embassy_stm32::init(config);

    loop {
        info!("Hello World!");
        Timer::after(Duration::from_secs(1)).await;
    }
}
```

只需要定义这些代码就能启动一个每隔 1 秒打印一个 “Hello World!” 的协程。

## 过程宏

除了相关的 `cortex_m_rt` crate 提供的布局和启动向量表之外，embassy 主要依靠 `#[embassy_executor::main]` 这个过程宏提供服务。

`embassy_executor::main` 的定义位于 `embassy-macro/src/macros/main.rs`，在相关的 `run` 里，原本的 `async fn main` 被包装成 `__embassy_main`：

```rust
let result = quote! {
    #[::embassy_executor::task()]
    async fn __embassy_main(#fargs) {
        #f_body
    }

    unsafe fn __make_static<T>(t: &mut T) -> &'static mut T {
        ::core::mem::transmute(t)
    }

    #main
};

Ok(result)
```

随后被传入一个默认的 `Executor`，整体传递给 rt 库定义的入口：

```rust
pub fn cortex_m() -> TokenStream {
    quote! {
        #[cortex_m_rt::entry]
        fn main() -> ! {
            let mut executor = ::embassy_executor::Executor::new();
            let executor = unsafe { __make_static(&mut executor) };
            executor.run(|spawner| {
                spawner.must_spawn(__embassy_main(spawner));
            })
        }
    }
}
```

## `Executor`
