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

`__embassy_main` 又是 `#[::embassy_executor::task()]` 修饰的，这个过程宏定义在 `embassy-macro/src/macros/task.rs`:

```rust
let result = quote! {
    // This is the user's task function, renamed.
    // We put it outside the #task_ident fn below, because otherwise
    // the items defined there (such as POOL) would be in scope
    // in the user's code.
    #task_inner

    #visibility fn #task_ident(#fargs) -> ::embassy_executor::SpawnToken<impl Sized> {
        type Fut = impl ::core::future::Future + 'static;
        static POOL: ::embassy_executor::raw::TaskPool<Fut, #pool_size> = ::embassy_executor::raw::TaskPool::new();
        unsafe { POOL._spawn_async_fn(move || #task_inner_ident(#(#arg_names,)*)) }
    }
};

Ok(result)
```

主要就是为顶层协程提供一个 `TaskPool`，并将任务传入 Pool 产生一个 `SpawnToken`。

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

### Higher level executor

`Executor` 是特定于架构的简化版执行器，以 cortex_m 为例（`embassy-executor/src/arch/cortex_m.rs`）：

```rust
use core::arch::asm;
use core::marker::PhantomData;
use core::ptr;

use super::{raw, Spawner};

/// Thread mode executor, using WFE/SEV.
///
/// This is the simplest and most common kind of executor. It runs on
/// thread mode (at the lowest priority level), and uses the `WFE` ARM instruction
/// to sleep when it has no more work to do. When a task is woken, a `SEV` instruction
/// is executed, to make the `WFE` exit from sleep and poll the task.
///
/// This executor allows for ultra low power consumption for chips where `WFE`
/// triggers low-power sleep without extra steps. If your chip requires extra steps,
/// you may use [`raw::Executor`] directly to program custom behavior.
pub struct Executor {
    inner: raw::Executor,
    not_send: PhantomData<*mut ()>,
}

impl Executor {
    /// Create a new Executor.
    pub fn new() -> Self {
        Self {
            inner: raw::Executor::new(|_| unsafe { asm!("sev") }, ptr::null_mut()),
            not_send: PhantomData,
        }
    }

    /// Run the executor.
    ///
    /// The `init` closure is called with a [`Spawner`] that spawns tasks on
    /// this executor. Use it to spawn the initial task(s). After `init` returns,
    /// the executor starts running the tasks.
    ///
    /// To spawn more tasks later, you may keep copies of the [`Spawner`] (it is `Copy`),
    /// for example by passing it as an argument to the initial tasks.
    ///
    /// This function requires `&'static mut self`. This means you have to store the
    /// Executor instance in a place where it'll live forever and grants you mutable
    /// access. There's a few ways to do this:
    ///
    /// - a [StaticCell](https://docs.rs/static_cell/latest/static_cell/) (safe)
    /// - a `static mut` (unsafe)
    /// - a local variable in a function you know never returns (like `fn main() -> !`), upgrading its lifetime with `transmute`. (unsafe)
    ///
    /// This function never returns.
    pub fn run(&'static mut self, init: impl FnOnce(Spawner)) -> ! {
        init(self.inner.spawner());

        loop {
            unsafe {
                self.inner.poll();
                asm!("wfe");
            };
        }
    }
}
```

如文档所述，`Executor` 使用 cortex-M 的 sev 和 wfe 指令实现 async wake。这两个指令成对出现，wfe = wfi+e，会在发生陷入或任何核执行 sev 时退出。sev 则专用于唤醒 wfe。

> 但是这个操作对于单核来说似乎没用。

### Raw executor

程序启动之后首先初始化了一个 `Executor`，也就是初始化了一个 `raw::Executor`，`raw::Executor` 的定义和初始化如下：

> 隐去了 `integrated-timers` feature 控制的部分。

```rust
/// Raw executor.
///
/// This is the core of the Embassy executor. It is low-level, requiring manual
/// handling of wakeups and task polling. If you can, prefer using one of the
/// [higher level executors](crate::Executor).
///
/// The raw executor leaves it up to you to handle wakeups and scheduling:
///
/// - To get the executor to do work, call `poll()`. This will poll all queued tasks (all tasks
///   that "want to run").
/// - You must supply a `signal_fn`. The executor will call it to notify you it has work
///   to do. You must arrange for `poll()` to be called as soon as possible.
///
/// `signal_fn` can be called from *any* context: any thread, any interrupt priority
/// level, etc. It may be called synchronously from any `Executor` method call as well.
/// You must deal with this correctly.
///
/// In particular, you must NOT call `poll` directly from `signal_fn`, as this violates
/// the requirement for `poll` to not be called reentrantly.
pub struct Executor {
    run_queue: RunQueue,
    signal_fn: fn(*mut ()),
    signal_ctx: *mut (),
}

impl Executor {
    /// Create a new executor.
    ///
    /// When the executor has work to do, it will call `signal_fn` with
    /// `signal_ctx` as argument.
    ///
    /// See [`Executor`] docs for details on `signal_fn`.
    pub fn new(signal_fn: fn(*mut ()), signal_ctx: *mut ()) -> Self {
        Self {
            run_queue: RunQueue::new(),
            signal_fn,
            signal_ctx,
        }
    }

...

}
```

即直接把 sev 当作 `signal_fn` 保存起来。同时 `signal_ctx` 是一个空指针。

`init(self.inner.spawner());` 对于默认的过程宏，展开成：

```rust
let spawner = self.inner.spawner();
spawner.must_spawn(__embassy_main(spawner));
```

把主程序也代入的话，展开成：

> 省略了 TaskPool 相关的部分。

```rust
self.inner.spawner().must_spawn(async {
    let mut config = Config::default();
    config.rcc.sys_ck = Some(Hertz(36_000_000));
    let _p = embassy_stm32::init(config);

    loop {
        info!("Hello World!");
        Timer::after(Duration::from_secs(1)).await;
    }
})
```

### Spawner

`Spawner` 只是 `raw::Executor` 的包装，代表 `!Send` 约束：

```rust
/// Handle to spawn tasks into an executor.
///
/// This Spawner can spawn any task (Send and non-Send ones), but it can
/// only be used in the executor thread (it is not Send itself).
///
/// If you want to spawn tasks from another thread, use [SendSpawner].
#[derive(Copy, Clone)]
pub struct Spawner {
    executor: &'static raw::Executor,
    not_send: PhantomData<*mut ()>,
}

impl Spawner {
    pub(crate) fn new(executor: &'static raw::Executor) -> Self {
        Self {
            executor,
            not_send: PhantomData,
        }
    }

    /// Get a Spawner for the current executor.
    ///
    /// This function is `async` just to get access to the current async
    /// context. It returns instantly, it does not block/yield.
    ///
    /// # Panics
    ///
    /// Panics if the current executor is not an Embassy executor.
    pub async fn for_current_executor() -> Self {
        poll_fn(|cx| unsafe {
            let task = raw::task_from_waker(cx.waker());
            let executor = (*task.as_ptr()).executor.get();
            Poll::Ready(Self::new(&*executor))
        })
        .await
    }

    /// Spawn a task into an executor.
    ///
    /// You obtain the `token` by calling a task function (i.e. one marked with `#[embassy_executor::task]`).
    pub fn spawn<S>(&self, token: SpawnToken<S>) -> Result<(), SpawnError> {
        let task = token.raw_task;
        mem::forget(token);

        match task {
            Some(task) => {
                unsafe { self.executor.spawn(task) };
                Ok(())
            }
            None => Err(SpawnError::Busy),
        }
    }

    // Used by the `embassy_macros::main!` macro to throw an error when spawn
    // fails. This is here to allow conditional use of `defmt::unwrap!`
    // without introducing a `defmt` feature in the `embassy_macros` package,
    // which would require use of `-Z namespaced-features`.
    /// Spawn a task into an executor, panicking on failure.
    ///
    /// # Panics
    ///
    /// Panics if the spawning fails.
    pub fn must_spawn<S>(&self, token: SpawnToken<S>) {
        unwrap!(self.spawn(token));
    }

    /// Convert this Spawner to a SendSpawner. This allows you to send the
    /// spawner to other threads, but the spawner loses the ability to spawn
    /// non-Send tasks.
    pub fn make_send(&self) -> SendSpawner {
        SendSpawner {
            executor: self.executor,
        }
    }
}
```

所以 `Spawner::must_spawn` 完全就是 `raw::Executor::spawn`，但 task 变成了一个指针。

### spawn

`raw::Executor::spawn` 就是把 task 放入 `run_queue`，并立即调用一次 `signal_fn`：

> `RunQueue` 是侵入式单链表。

> 省略了 `rtos-trace` feature 控制的部分。

```rust
/// Enqueue a task in the task queue
///
/// # Safety
/// - `task` must be a valid pointer to a spawned task.
/// - `task` must be set up to run in this executor.
/// - `task` must NOT be already enqueued (in this executor or another one).
#[inline(always)]
unsafe fn enqueue(&self, cs: CriticalSection, task: NonNull<TaskHeader>) {
    if self.run_queue.enqueue(cs, task) {
        (self.signal_fn)(self.signal_ctx)
    }
}

/// Spawn a task in this executor.
///
/// # Safety
///
/// `task` must be a valid pointer to an initialized but not-already-spawned task.
///
/// It is OK to use `unsafe` to call this from a thread that's not the executor thread.
/// In this case, the task's Future must be Send. This is because this is effectively
/// sending the task to the executor thread.
pub(super) unsafe fn spawn(&'static self, task: NonNull<TaskHeader>) {
    task.as_ref().executor.set(self);
    critical_section::with(|cs| {
        self.enqueue(cs, task);
    })
}
```

### poll

最后关注 `poll`：

> 省略了所有 feature 控制的部分：

```rust
/// Poll all queued tasks in this executor.
///
/// This loops over all tasks that are queued to be polled (i.e. they're
/// freshly spawned or they've been woken). Other tasks are not polled.
///
/// You must call `poll` after receiving a call to `signal_fn`. It is OK
/// to call `poll` even when not requested by `signal_fn`, but it wastes
/// energy.
///
/// # Safety
///
/// You must NOT call `poll` reentrantly on the same executor.
///
/// In particular, note that `poll` may call `signal_fn` synchronously. Therefore, you
/// must NOT directly call `poll()` from your `signal_fn`. Instead, `signal_fn` has to
/// somehow schedule for `poll()` to be called later, at a time you know for sure there's
/// no `poll()` already running.
pub unsafe fn poll(&'static self) {
    loop {
        self.run_queue.dequeue_all(|p| {
            let task = p.as_ref();

            let state = task.state.fetch_and(!STATE_RUN_QUEUED, Ordering::AcqRel);
            if state & STATE_SPAWNED == 0 {
                // If task is not running, ignore it. This can happen in the following scenario:
                //   - Task gets dequeued, poll starts
                //   - While task is being polled, it gets woken. It gets placed in the queue.
                //   - Task poll finishes, returning done=true
                //   - RUNNING bit is cleared, but the task is already in the queue.
                return;
            }

            // Run the task
            task.poll_fn.read()(p as _);
        });
    }
}
```
