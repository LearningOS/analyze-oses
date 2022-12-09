# Aero 文件系统分析

## fs 对 kernel 的依赖

通过整理 use 和路径，可以得到 fs 对 kernel 其他部分的依赖：

```rust
mod internal {
    pub(super) use crate::{
        arch::tls,
        cmdline, logger,
        mem::paging::{
            align_down, FrameAllocator, PageSize, PhysAddr, PhysFrame, Size4KiB, VirtAddr,
            FRAME_ALLOCATOR,
        },
        rendy::{get_rendy_info, RendyInfo, DEBUG_RENDY},
        socket::{unix::UnixSocket, SocketAddr},
        userland::scheduler,
        utils::{
            buffer::Buffer,
            sync::{BlockQueue, Mutex},
            CeilDiv,
        },
        PHYSICAL_MEMORY_OFFSET,
    };
}
```

## cache

梳理内部结构，可以看到 cache 对外部依赖是比较少的：

> 除了 `core`、`alloc` 和 `spin` 之外只有这么多：

```rust
use super::{
    inode::{DirEntry, INodeInterface},
    internal::Mutex,
    FileSystem,
};
```

缓存系统的层次是 `Cache` <- `CacheIndex` <- `CacheItem` <- `Cacheable`  <- `CacheKey`。

- `CacheKey` 就是 `Hash + Ord + Borrow<Self> + Debug` 的 trait 别名。

  ```rust
  pub trait CacheKey: Hash + Ord + Borrow<Self> + Debug {}

  impl<T> CacheKey for T where T: Hash + Ord + Borrow<Self> + Debug {}
  ```

- `Cacheable` 就是能产生 `CacheKey` 的结构体：

  ```rust
  pub trait Cacheable<K: CacheKey>: Sized {
      fn cache_key(&self) -> K;
  }
  ```

- `CacheItem` 是一个满足 `Cacheable` 的值，以及一个保存它是否占用的原子变量：

  ```rust
  pub struct CacheItem<K: CacheKey, V: Cacheable<K>> {
      cache: Weak<Cache<K, V>>,
      value: V,
      /// Whether the cache item has active strong references associated
      /// with it.
      used: AtomicBool,
  }

  impl<K: CacheKey, V: Cacheable<K>> CacheItem<K, V> {
      pub fn new(cache: &Weak<Cache<K, V>>, value: V) -> CacheArc<Self> {
          CacheArc::new(Self {
              cache: cache.clone(),
              value,
              used: AtomicBool::new(false),
          })
      }

      pub fn is_used(&self) -> bool {
          self.used.load(Ordering::SeqCst)
      }

      pub fn set_used(&self, yes: bool) {
          self.used.store(yes, Ordering::SeqCst);
      }
  }
  ```

- `CacheIndex` 用 `HashMap` 保存正在使用的对象，用 512 格的 lru 保存未使用的对象：

  ```rust
  struct CacheIndex<K: CacheKey, V: Cacheable<K>> {
      used: hashbrown::HashMap<K, Weak<CacheItem<K, V>>>,
      /// Cache items that are longer have any active strong references associated
      /// with them. These are stored in the cache index so, if the item is
      /// accessed again, we can re-use it; reducing required memory allocation
      /// and I/O (if applicable).
      unused: lru::LruCache<K, Arc<CacheItem<K, V>>>,
  }
  ```

  重点是两个交换操作，`get` 把对象从 `unused` 移动到 `used`，`mark_item_unused` 从 `used` 移动到 `unused`

  ```rust
  pub fn get(&self, key: K) -> Option<CacheArc<CacheItem<K, V>>> {
      let mut index = self.index.lock();

      if let Some(entry) = index.used.get(&key) {
          let entry = entry.upgrade()?;
          Some(CacheArc::from(entry))
      } else if let Some(entry) = index.unused.pop(&key) {
          entry.set_used(true);
          index.used.insert(key, Arc::downgrade(&entry));

          Some(entry.into())
      } else {
          None
      }
  }

  fn mark_item_unused(&self, item: CacheArc<CacheItem<K, V>>) {
      item.set_used(false);

      let mut index = self.index.lock_irq();
      let key = item.cache_key();

      assert!(index.used.remove(&key).is_some());
      index.unused.put(key, item.0.clone());
  }
  ```

  `mark_item_unused` 只会在 `CacheItem` 释放时调用。

  ```rust
  impl<K: CacheKey, T: Cacheable<K>> CacheDropper for CacheItem<K, T> {
      fn drop_this(&self, this: Arc<Self>) {
          if let Some(cache) = self.cache.upgrade() {
              if self.is_used() {
                  cache.mark_item_unused(this.into());
              }
          }
      }
  }
  ```
