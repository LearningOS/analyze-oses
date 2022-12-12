1. syscall: socket

pub fn socket(domain: usize, socket_type: usize, protocol: usize) -> Result<usize, SyscallError> 
从当前线程分配获取一个fd返回

用来建立 socket 连接
从 userland 过来一个 socket 请求，会携带 domain，socket_type, protocol 三个参数，
分别代表 socket 域，socket 类型和协议类型。

```
常见的 domain： 

AF_UNIX = 1  // Local communication
AF_INET = 2  // IPv4 Internet protocols
AF_INET6 = 10 // IPv6 Internet protocols
AF_NETLINK = 16 // Kernel user interface device
AF_PACKET = 17 // Low-level packet interface
```

```
常见的 socket_type:

SOCK_STREAM = 1 // An out-of-band data transmission mechanism may be supported
SOCK_DGRAM = 2 // Supports datagrams (connectionless, unreliable messages of a fixed maximum length)
SOCK_RAW = 3 // Provides raw network protocol access
SOCK_RDM = 4 // Provides a reliable datagram layer that does not guarantee ordering
SOCK_SEQPACKET = 5 // Provides a sequenced, reliable, two-way connection-based data
SOCK_DCCP = 6 // Datagram Congestion Control Protocol socket
SOCK_PACKET = 10 // Obsolete and should not be used in new programs
SOCK_NONBLOCK = 0x800 // Set O_NONBLOCK flag on the open fd
SOCK_CLOEXEC = 0x80000 // Set FD_CLOEXEC flag on the new fd
```

```
常见的 protocol：

IPPROTO_IP = 0 // Dummy protocol for TCP
IPPROTO_ICMP = 1 // Internet Control Message Protocol
IPPROTO_TCP = 6 // Transmission Control Protocol
IPPROTO_UDP = 17 // User Datagram Protocol
IPPROTO_IPV6 = 41 // IPv6-in-IPv4 tunnelling
IPPROTO_ICMPV6 = 58 //  ICMPv6
```

目前只支持 domain：AF_UNIX,其他域直接打印 log，并返回错误。
如果传递过来的 domain 是 AF_UNIX ，则会创建一个 UnixSocket 结构体， 这个结构及其方法
定义在一个单独的 socket 模块。

```
pub struct UnixSocket {
    inner: Mutex<UnixSocketInner>,
    buffer: Mutex<MessageQueue>,
    wq: BlockQueue,
    weak: Weak<UnixSocket>,
    handle: Once<Arc<FileHandle>>,
}
```

从传入的 socket_type 去创建 socket fd 的 OpenFlag 参数。
因为 UnixSocket 实现了 INodeInterface trait，所以可以从 socket 生产出一个
DirCacheItem。
然后获取当前调度线程的文件列表，从中分配一个 fd 返回。

2. syscall: bind

pub fn bind(fd: usize, address: usize, length: usize) -> Result<usize, SyscallError>
给 socket 绑定地址

从 userland 传递的 address 绑定到传递的 fd 上，这个 fd 就是 socket 创建返回的那个。

首先使用 socket_addr_from_addr 函数生产一个 SocketAddr
然后从当前调度线程获取 fd 的处理句柄，如果句柄不为空且是 socket 类型，就绑定地址，
具体的绑定操作在 socket 模块里定义，以 INodeInterface trait 的实现形式出现，
INodeInterface 都有默认实现，即直接返回错误，它定义了很多 socket 相关的操作方法，
这样对于没有实现必要接口成员方法的结构也可以使用这个 trait，而不用重新定义个trait，
然后做 trait 间的转换。

对于 UnixSocket 类型来说，只要从传入的参数 address 转型到 SocketAddrUnix，
使用这个类型转型到 path 类型，判断这个路径在系统中是否已经存在，没有则创建并绑定，
并更新自身的 address 地址。


3. syscall: listen

pub fn listen(fd: usize, backlog: usize) -> Result<usize, SyscallError>
开始监听是否有连接

开始会先关中断并获得锁，然后判断当前 UnixSocket 实例结构中的 state，
如果是 Disconnected 且 address 存在，则执行监听，并把 state 置成 Listening
如果是 Listening，则直接设置监听的数目。


4. syscall: connect

pub fn connect(fd: usize, address: usize, length: usize) -> Result<usize, SyscallError>
开始连接

从当前调度线程获取 fd 的处理句柄，然后根据 address 找到对应的 DirCacheItem
从 DirCacheItem 转型到 UnixSocket，如果处于 Listening 状态，就把自己插入到
当前 AcceptQueue 队列里，并通知操作完成。
阻塞操作直到状态变成 Connected。


5. syscall: accept

pub fn accept(fd: usize, address: usize, length: usize) -> Result<usize, SyscallError>
接收连接

阻塞直到 AcceptQueue 不为空，然后从队列中 pop 出一个 UnixSocket 结构处理
先把本端和对端状态改成 Connected，然后修改对端的地址和长度。
最后返回当前处理句柄。
