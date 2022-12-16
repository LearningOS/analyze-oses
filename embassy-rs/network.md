# embassy network

embassy 网络定义在 embassy-net 这个 crate 里，底层使用 smoltcp 协议栈，支持 tcp 和 udp。
目前在项目中 examples/nrf 中 usb_ethernet.rs 中引用。

以 tcp 为例：

构建了三种错误类型

``` rust
pub enum Error {
    ConnectionReset,
}

pub enum ConnectError {
    /// The socket is already connected or listening.
    InvalidState,
    /// The remote host rejected the connection with a RST packet.
    ConnectionReset,
    /// Connect timed out.
    TimedOut,
    /// No route to host.
    NoRoute,
}

pub enum AcceptError {
    /// The socket is already connected or listening.
    InvalidState,
    /// Invalid listen port
    InvalidPort,
    /// The remote host rejected the connection with a RST packet.
    ConnectionReset,
}

```

socket 连接时错误使用  ConnectError， 接受 socket 时错误使用 AcceptError， 其他错误使用 Error。

定义了三种数据结构

```rust
pub struct TcpSocket<'a> {
    io: TcpIo<'a>,
}

pub struct TcpReader<'a> {
    io: TcpIo<'a>,
}

pub struct TcpWriter<'a> {
    io: TcpIo<'a>,
}
```

这三种结构有一个共同的字段 TcpIo

```rust
struct TcpIo<'a> {
    stack: &'a RefCell<SocketStack>,
    handle: SocketHandle,
}

pub(crate) struct SocketStack {
    pub(crate) sockets: SocketSet<'static>,
    pub(crate) iface: Interface<'static>,
    pub(crate) waker: WakerRegistration,
    next_local_port: u16,
}

pub struct SocketHandle(usize);
```

TcpReader 结构实现了 read 方法， TcpWriter 实现了 write 方法，TcpSocket 方法比较多，new 用来做构造函数，使用 Stack， rx_buffer, tx_buffer 三个参数。
rx_buffer, tx_buffer 是一个可变 u8 切片。

```rust
 pub fn new<D: Device>(stack: &'a Stack<D>, rx_buffer: &'a mut [u8], tx_buffer: &'a mut [u8]) -> Self
 ```

Stack 结构如下，封装了 SocketStack 和 Inner 结构，其底层基本上都是引用 smoltcp 中结构。

```rust
pub struct Stack<D: Device> {
    pub(crate) socket: RefCell<SocketStack>,
    inner: RefCell<Inner<D>>,
}

pub(crate) struct SocketStack {
    pub(crate) sockets: SocketSet<'static>,
    pub(crate) iface: Interface<'static>,
    pub(crate) waker: WakerRegistration,
    next_local_port: u16,
}

struct Inner<D: Device> {
    device: D,
    link_up: bool,
    config: Option<Config>,
    #[cfg(feature = "dhcpv4")]
    dhcp_socket: Option<SocketHandle>,
}
```

connect 方法用来连接 socket，需要对端的 endpoint，这是一个泛型参数，用可以转化到 IpEndpoint 做限定。

首先使用内部结构的方法 get_local_port() 获取本地端口号，然后结合传入的参数 remote_endpoint， 然后调用 smoltcp crate 中的 connect 方法进行连接。

然后进行轮询，直到状态变成 Poll::Ready。

```rust
    pub async fn connect<T>(&mut self, remote_endpoint: T) -> Result<(), ConnectError>
    where
        T: Into<IpEndpoint>,
    {
        let local_port = self.io.stack.borrow_mut().get_local_port();

        match {
            self.io
                .with_mut(|s, i| s.connect(i.context(), remote_endpoint, local_port))
        } {
            Ok(()) => {}
            Err(tcp::ConnectError::InvalidState) => return Err(ConnectError::InvalidState),
            Err(tcp::ConnectError::Unaddressable) => return Err(ConnectError::NoRoute),
        }

        poll_fn(|cx| {
            self.io.with_mut(|s, _| match s.state() {
                tcp::State::Closed | tcp::State::TimeWait => Poll::Ready(Err(ConnectError::ConnectionReset)),
                tcp::State::Listen => unreachable!(),
                tcp::State::SynSent | tcp::State::SynReceived => {
                    s.register_send_waker(cx.waker());
                    Poll::Pending
                }
                _ => Poll::Ready(Ok(())),
            })
        })
        .await
    }
```

accept 方法用来接受 socket 连接，需要本地的 local_endpoint，这是一个泛型参数，用可以转化到 IpListenEndpoint 做限定。

同样是调用底层 smoltcp 的方法 listen()，然后进行轮询，直到状态变成 Poll::Ready。

```rust
pub async fn accept<T>(&mut self, local_endpoint: T) -> Result<(), AcceptError>
    where
        T: Into<IpListenEndpoint>,
    {
        match self.io.with_mut(|s, _| s.listen(local_endpoint)) {
            Ok(()) => {}
            Err(tcp::ListenError::InvalidState) => return Err(AcceptError::InvalidState),
            Err(tcp::ListenError::Unaddressable) => return Err(AcceptError::InvalidPort),
        }

        poll_fn(|cx| {
            self.io.with_mut(|s, _| match s.state() {
                tcp::State::Listen | tcp::State::SynSent | tcp::State::SynReceived => {
                    s.register_send_waker(cx.waker());
                    Poll::Pending
                }
                _ => Poll::Ready(Ok(())),
            })
        })
        .await
    }
```

TcpSocket 的 read 和 write 也是对内部结构 TcpIo 方法的简单包装。其他一些辅助函数和方法类似。

TcpIo 的 read 和 write 其底层是 smoltcp 结构方法，实现异步操作。

从 socket 中读写数据，直到操作成功。

```rust
    pub async fn read(&mut self, buf: &mut [u8]) -> Result<usize, Error> {
        self.io.read(buf).await
    }

    pub async fn write(&mut self, buf: &[u8]) -> Result<usize, Error> {
        self.io.write(buf).await
    }

   async fn read(&mut self, buf: &mut [u8]) -> Result<usize, Error> {
        poll_fn(move |cx| {
            // CAUTION: smoltcp semantics around EOF are different to what you'd expect
            // from posix-like IO, so we have to tweak things here.
            self.with_mut(|s, _| match s.recv_slice(buf) {
                // No data ready
                Ok(0) => {
                    s.register_recv_waker(cx.waker());
                    Poll::Pending
                }
                // Data ready!
                Ok(n) => Poll::Ready(Ok(n)),
                // EOF
                Err(tcp::RecvError::Finished) => Poll::Ready(Ok(0)),
                // Connection reset. TODO: this can also be timeouts etc, investigate.
                Err(tcp::RecvError::InvalidState) => Poll::Ready(Err(Error::ConnectionReset)),
            })
        })
        .await
    }

    async fn write(&mut self, buf: &[u8]) -> Result<usize, Error> {
        poll_fn(move |cx| {
            self.with_mut(|s, _| match s.send_slice(buf) {
                // Not ready to send (no space in the tx buffer)
                Ok(0) => {
                    s.register_send_waker(cx.waker());
                    Poll::Pending
                }
                // Some data sent
                Ok(n) => Poll::Ready(Ok(n)),
                // Connection reset. TODO: this can also be timeouts etc, investigate.
                Err(tcp::SendError::InvalidState) => Poll::Ready(Err(Error::ConnectionReset)),
            })
        })
        .await
    }
```

下面看看在 usb_ethernet.rs 中如何使用的。

```rust
    // Init network stack
    let stack = &*singleton!(Stack::new(
        device,
        config,
        singleton!(StackResources::<1, 2, 8>::new()),
        seed
    ));

    unwrap!(spawner.spawn(net_task(stack)));

    // And now we can use it!

    let mut rx_buffer = [0; 4096];
    let mut tx_buffer = [0; 4096];
    let mut buf = [0; 4096];

    loop {
        let mut socket = TcpSocket::new(stack, &mut rx_buffer, &mut tx_buffer);
        socket.set_timeout(Some(embassy_net::SmolDuration::from_secs(10)));

        info!("Listening on TCP:1234...");
        if let Err(e) = socket.accept(1234).await {
            warn!("accept error: {:?}", e);
            continue;
        }

        info!("Received connection from {:?}", socket.remote_endpoint());

        loop {
            let n = match socket.read(&mut buf).await {
                Ok(0) => {
                    warn!("read EOF");
                    break;
                }
                Ok(n) => n,
                Err(e) => {
                    warn!("read error: {:?}", e);
                    break;
                }
            };

            info!("rxd {:02x}", &buf[..n]);

            match socket.write_all(&buf[..n]).await {
                Ok(()) => {}
                Err(e) => {
                    warn!("write error: {:?}", e);
                    break;
                }
            };
        }
    }
}
```
