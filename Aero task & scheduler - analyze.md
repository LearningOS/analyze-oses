# Aero 代码阅读：进程 / 线程 / 调度

以下讨论均基于 `/src/aero_kernel/src` 目录

## 线程

见 `/userland/task.rs`。没有特别设计线程结构，但有一些线程特征：

在 `Task::clone_process` 方法中体现通常意义上的“创建线程”：

- 复制 fdtable / vm 上的 Arc，即新线程(实际是进程)和当前进程共享这些信息

- 复制 signals (信号)模块的值，即新线程(实际是进程)直接用这些信息初始化

但是又有如下不同：

- 没有独立的 tid，所有的 tid 都等于 pid（尽管它们可能共用地址空间）

- 没有tls。 `/syscall/process.rs` 中 `sys_clone` 只支持前两个参数，即

```rust
pub fn sys_clone(entry: usize, stack: usize) -> Result<usize, SyscallError> {
    let value = syscall2(prelude::SYS_CLONE, entry, stack);
    isize_as_syscall_result(value as _)
}
```

    事实上后面还应该有三个参数 `ptid` `tls` `ctid`

- 相关的线程 syscall ，如 `sys_set_tid_address` 等等自然也是没有的

#### comments

Aero 其实没有准备线程的东西，甚至没有“线程”这个结构。但它简单地复制了 vm 的指针和其他一些模块，绕过了很多处理：

如信号的标准实现应该是 "fork 继承，exec不继承，clone线程时共享处理函数"，它这里只复制了值，没有共享 Arc，居然也能成功运行。

但这些东西以及相关 `syscall` 的缺失意味着它一定无法通过比赛要求的 `libc-test`，也几乎无法运行使用现代 `libc` 的多线程程序

## 进程

#### pid

原子变量自增分配，不重用

```rust
#[derive(Debug, Copy, Clone, PartialEq, Eq, Hash, PartialOrd, Ord)]
#[repr(transparent)]
pub struct TaskId(usize);

impl TaskId {
    pub const fn new(pid: usize) -> Self {
        Self(pid)
    }

    /// Allocates a new task ID.
    fn allocate() -> Self {
        static NEXT_PID: AtomicUsize = AtomicUsize::new(1);

        Self::new(NEXT_PID.fetch_add(1, Ordering::AcqRel))
    }

    pub fn as_usize(&self) -> usize {
        self.0
    }
}
```

#### 进程结构体

大致结构如下，大体上和通常的os是一样的，特殊的下面再讲

```rust
pub struct Task {
    sref: Weak<Task>,

    arch_task: UnsafeCell<ArchTask>,
    state: AtomicU8,

    pid: TaskId,
    tid: TaskId,

    parent: Mutex<Option<Arc<Task>>>,
    children: Mutex<intrusive_collections::LinkedList<TaskAdapter>>,

    zombies: Zombies,

    sleep_duration: AtomicUsize,
    signals: Signals,

    executable: Mutex<Option<DirCacheItem>>,
    pending_io: AtomicBool,

    pub(super) link: intrusive_collections::LinkedListLink,
    pub(super) clink: intrusive_collections::LinkedListLink,

    pub vm: Arc<Vm>,
    pub file_table: Arc<FileTable>,

    pub message_queue: MessageQueue,

    cwd: RwLock<Option<Cwd>>,

    pub(super) exit_status: AtomicIsize,
}
```

有几个量可能需要说明：

- `arch_task`：本体在 `/arch/x86_64/task.rs` 下，管理一些上下文 / 中断信息。类似 `rCore-Tutorial` 的 `TaskContext`。实际的写法比较复杂，不是直接做寄存器保存，具体可看上面文件。

- `tid`：一定等于 `pid`

- `link` 和 `clink`：管理进程父子关系的链表，`link` 指向兄弟，`clink` 指向孩子

- `message_queue`：`ipc`相关

- `zombies`：这里的僵尸进程处理涉及一个完整的结构：

```rust
struct Zombies {
    list: Mutex<LinkedList<SchedTaskAdapter>>,
    block: BlockQueue,
}
```

这个结构有意思的一点在于，它接管了 `waitpid`。这个结构还包含了一个 `BlockQueue`，它本质上和 `loop {...}` 相同，但和下面提到的调度密切相关

## 调度

调度方式为 `round robin`，从代码来看是 FIFO，但还是与 `rCore-Tutorial` 区别较大。具体来说，调度核心依赖于 `userland/scheduler/round_robin.rs` 提到的结构 `TaskQueue`：

```rust
/// Scheduler queue containing a vector of all of the task of the enqueued
/// taskes.
struct TaskQueue {
    /// The kernel idle task is a special kind of task that is run when
    /// no taskes in the scheduler's queue are avaliable to execute. The idle task
    /// is to be created for each CPU.
    idle_task: Arc<Task>,
    preempt_task: Arc<Task>,
    current_task: Option<Arc<Task>>,

    runnable: LinkedList<SchedTaskAdapter>,
    dead: LinkedList<SchedTaskAdapter>,
    awaiting: LinkedList<SchedTaskAdapter>,
    deadline_awaiting: LinkedList<SchedTaskAdapter>,
}
```

这里有四个队列，用法和命名类似：

- `runnable`：可运行的进程，每次上下文切换后会从这里找一个进程运行

- `dead`：已结束进程

- `awaiting`：阻塞进程，不可运行，需要其他进程调用 `SchedulerInterface::wake_up` 才会被扔回 `runnable` 队列

- - （`trait SchedulerInterface` 是一个抽象接口，但目前只有一个实现，就是上面的 `TaskQueue`，所以只看上面就行）

- `deadline_awaiting`：睡眠进程，时间到了就会被扔回 `runnable` 队列

- - 检查时间的方式是每次上下文切换时查询这个队列里所有进程的等待时间

上面的三个 `idle_task` `current_task` `preempt_task` 都不在四个队列里：

- `idle_task`：空闲时进程

- `preempt_task`："调度进程"。在上下文切换时，首先会切换到这个进程，再切到下一个真正的进程。它只有两个用法，循环切换下一个进程或者处理僵尸进程：

```rust
fn sweeper() {
    let scheduler_ref = super::get_scheduler()
        .inner
        .downcast_arc::<RoundRobin>()
        .unwrap();

    loop {
        scheduler_ref.sweep_dead();
    }
}

fn preempter() {
    let scheduler_ref = super::get_scheduler()
        .inner
        .downcast_arc::<RoundRobin>()
        .unwrap();

    loop {
        scheduler_ref.schedule_next_task();
    }
}
```

    这个进程实际上代替了一部分 `rCore-Tutorial`中 `idle_task` 的功能。在`rCore-Tutorial`中，通常是在**切换任务的途中**切回 `idle_task`，然后再切到新进程。Aero 这个设计事实上区分了“在切换任务的过程中”和“现在空闲”这两种状态

## BlockQueue

在 `utils/sync.rs`里定义的结构。这是一个类似信号量或者`futex`的等待队列，目的是使一个进程等待在条件变量上。作者试图用 `future` 来命名它的一些功能，但实际效果并不是 `future`。

```rust
pub struct BlockQueue {
    queue: Mutex<Vec<Arc<Task>>>,
}
```

这个队列标记的是"等待在同一个条件变量上的进程"，另有方法 `BlockQueue::notify_complete()` 可以唤醒队列中所有进程。

它的主要方法 `block_on`的参数如下：

```rust
pub fn block_on<'a, T, F: FnMut(&mut MutexGuard<T>) -> bool>(
        &self,
        mutex: &'a Mutex<T>,
        mut f: F,
    ) -> SignalResult<MutexGuard<'future, T>> {
```

其中 `Mutex` 是一个**自定义的关中断的锁**，不是原版 `spin::Mutex`

它的内容具体来说分为以下部分(伪代码，变量名不完全相同)：

1. 运行 `f(mutex.lock())` ，如果为 `true` 则直接返回。这对应实际上不需要锁的情况。

2. 将进程加入自身队列

3. 然后循环：
   
   - 标记当前进程为 `sleep`，扔进调度一节说的`awaiting`队列。这意味着该进程切出后不会再被调度，除非被其他进程 `wake_up`
   
   - 运行 `f(mutex.lock())`，返回 `true` 则退出循环

4. 将进程从自身队列中删除

#### comments

作者用变量名 `future` 来描述这个结构的一些内容，但实际上这个结构的效果只是条件变量，没有用到 `async` 关键字和任何相关的东西。

这里的循环部分想表达类似 `poll` 的含义，但加了一个意义不太明确的 `Mutex` 参数。从用法来看（如 `/userland/task.rs` 的 `Zombies`），可能是想把所有"参数"藏在 `Mutex` 里，省去 `BlockQueue` 本需要的泛型参数。
