# udp-over-tcp

用于通过 TCP 流传输 UDP 数据报的库（和二进制文件）。

某些程序/协议只能通过 UDP 运行。有些网络只允许 TCP。这就是 `udp-over-tcp`派上用场的地方。该库分为两部分：

* `udp2tcp`- 通过 TCP 流转发传入的 UDP 数据报。返回流被转换回数据报并再次通过 UDP 发送回。这部分可以很容易地用作库和二进制文件。因此它可以独立运行，但也可以轻松包含在其他 Rust 程序中。UDP 套接字连接到第一个传入数据报的对等地址。因此，一个 [`Udp2Tcp`] 实例只能处理来自单个对等方的流量。
* `tcp2udp`- 接受 TCP 上的连接，并将传入流作为 UDP 数据报转换+转发到设置期间/命令行上指定的目的地。主要设计为在服务器上运行的独立可执行文件。但也可以作为 Rust 库使用。 `tcp2udp`继续接受新传入的 TCP 连接，并为每个连接创建一个新的 UDP 套接字。因此，单个`tcp2udp`服务器可以用于为多个`udp2tcp`客户端提供服务。

## 协议

TCP 流内的数据格式非常简单。每个数据报前面都有一个按大端字节顺序排列的 16 位无符号整数，指定数据报的长度。

## tcp2udp 服务器示例

让服务器侦听 TCP 连接，然后将其转发到本地 UDP 服务。这将监听`10.0.0.1:5001/TCP`并转发任何进入的内容`127.0.0.1:51820/UDP`：
```bash
user@server $ RUST_LOG=debug tcp2udp \
    --tcp-listen 10.0.0.0:5001 \
    --udp-forward 127.0.0.1:51820
```

`RUST_LOG`可用于设置日志记录级别。有关信息，请参阅文档[`env_logger`]。必须使用`env_logger`该功能构建板条箱才能使其处于活动状态。

`REDACT_LOGS=1`可以设置为使用日志中的服务编辑对等方的 IP。允许打开日志记录，但不将潜在的用户敏感数据存储到磁盘。

[`env_logger`]: https://crates.io/crates/env_logger

## UDP2TCP示例

这是集成`udp2tcp`到 Rust 程序中的一种方法。这会将 TCP socket连接到`1.2.3.4:9000`环回接口上的随机端口，并将 UDP socket绑定到该随机端口。然后，它将 UDP socket连接到第一个传入数据报的socket地址，并开始将所有流量转发到（或来自）TCP socket。

```rust

let udp_listen_addr = "127.0.0.1:0".parse().unwrap();
let tcp_forward_addr = "1.2.3.4:9000".parse().unwrap();

// 创建 UDP -> TCP 转发器, 这将连接 TCP socket
// 到 `tcp_forward_addr`
let udp2tcp = udp_over_tcp::Udp2Tcp::new(
    udp_listen_addr,
    tcp_forward_addr,
    udp_over_tcp::TcpOptions::default(),
)
.await?;

// 读出UDP实际绑定到哪个地址。如果您指定了端口，则很有用
// 零从操作系统获取随机端口
let local_udp_addr = udp2tcp.local_udp_addr()?;

spin_up_some_udp_thing(local_udp_addr);

// 运行转发器，直到 TCP socket断开连接或发生错误。
udp2tcp.run().await?;
```

