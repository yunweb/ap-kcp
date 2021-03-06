# AP-KCP

Armor-Piercing KCP (装甲穿透KCP/破甲KCP)

基于KCP修改和优化，用于穿透恶劣网络环境的高性能可靠传输协议(ARQ)。使用 Rust 实现。拥有基于 ring 的密码学支持，和基于 smol 的异步运行时。

下图为在校园网的繁忙时期，进行的简单带宽测试。

左为TCP(BBR拥塞控制)对照组，右为 AP-KCP(基于UDP传输)隧道。

![speedtest](speedtest.png)

**警告：AP-KCP可能会以侵占其他用户的带宽为代价换取使用者更好的网络吞吐效率，请勿进行大规模部署和使用，否则可能导致大范围的网络拥塞甚至瘫痪。**

## 概览

AP-KCP 与 KCP 一样，基于不可靠包传输建立可靠流式传输。它与下层协议实现无关，只需保证下层是基于封包的传输即可，即包不会被切分或合并。无需保证传输顺序和传输可靠性，甚至无需保证每个包的正确性和完整性（误码和篡改均会被纠正）。例如 UDP 和 ICMP 都可用作底层传输。

AP-KCP 与 KCP 有几点主要区别：

* AP-KCP 的目标是改善用户在恶劣网络环境下的网络吞吐，而非减少延迟。因此使用更为激进的拥塞控制策略以保证链路的传输瓶颈始终被 AP-KCP 包填充。

* AP-KCP 使用更短的首部和更小的 ACK 包，并将包粘连传送，以最大化有效载荷和传输速率。

* AP-KCP 添加了 AEAD 密码学支持，传输的封包均经过加密以消除特征，确保 ISP 无法进行有效检测和 QoS 限制，并保证包的完整性和传输数据安全。

AP-KCP 本身是一个异步库，可以被用于在任何不可靠的基于封包的底层传输上，建立高效的可靠流式传输。AP-KCP 的二进制发布版本是一个隧道程序，基于 UDP 建立了AP-KCP 隧道连接。

## 使用

### AP-KCP-TUN（作为可执行二进制程序使用）

AP-KCP 的二进制发行版本是一个基于 UDP 的加密隧道。

假设

* 你的服务器 IP 是 233.233.233.233

* 你想了一个很棒(lan)的密码 mypassword

* 你想要远端的 AP-KCP 服务端监听 4000 UDP 端口，本地 AP-KCP 客户端监听 3000 UDP 端口

* 你希望其他软件连接本地的 AP-KCP 监听的 3000 TCP 端口时，本地的 AP-KCP 将这个连接通过隧道传输，转换为 AP-KCP 流量发送到服务器的 4000 UDP 端口上，然后服务端的 AP-KCP 服务端将其重新转换为 TCP 连接，最终连接到 1.1.1.1 的 5000 TCP 端口。也就是说，连接本地的 3000 端口等同于连接 1.1.1.1 的 5000 端口。

客户端

```shell
ap-kcp-tun --client --password mypassword --local 127.0.0.1:3000 --remote 233.233.233.233:4000
```

服务端

```shell
ap-kcp-tun --server --password mypassword --local 0.0.0.0:4000 --remote 1.1.1.1:5000
```

你可以使用 --kcp-config 指定详细传输参数的配置文件（默认参数配置文件参见目录下 default-config.toml），使用 --algorithm 指定加密方式，支持下面三种 AEAD 加密方式：

* aes-256-gcm

* aes-128-gcm

* chacha20-poly1305

### AP-KCP（作为库使用）

AP-KCP 本身与底层协议实现无关。如果你需要在自己的协议上使用 AP-KCP，在 Cargo.toml 中添加依赖后，实现下面的 KcpIo trait 即可直接使用。

```rust
#[async_trait::async_trait]
pub trait KcpIo {
    async fn send_packet(&self, buf: &mut Vec<u8>) -> std::io::Result<()>;
    async fn recv_packet(&self, buf: &mut Vec<u8>) -> std::io::Result<()>;
}
```

`buf` 需要可变的原因，是使下层对包进行操作时实现零内存分配。AP-KCP 假定调用 `send_packet` 后 buf 内容无意义并立即清空，假定 `recv_packet` 得到的缓冲区大小即为接收到的包大小。

下面是一个示例，它为 `smol::net::UdpSocket` 实现了 `KcpIo` trait。

```rust
#[async_trait::async_trait]
impl KcpIo for smol::net::UdpSocket {
    async fn send_packet(&self, buf: &mut Vec<u8>) -> std::io::Result<()> {
        self.send(buf).await?;
        Ok(())
    }

    async fn recv_packet(&self, buf: &mut Vec<u8>) -> std::io::Result<()> {
        let size = self.recv(buf).await?;
        buf.truncate(size);
        Ok(())
    }
}
```

之后便可使用 UdpSocket 建立 AP-KCP 会话。

```rust
let udp = UdpSocket::bind("0.0.0.0:10000").await.unwrap();
udp.connect("233.233.233.233:20000").await.unwrap();
let kcp_handle = KcpHandle::new(udp, KcpConfig::default())?;
let mut stream1 = kcp_handle.connect().await.unwrap();
let mut stream2 = kcp_handle.connect().await.unwrap();
let mut stream3 = kcp_handle.accept().await.unwrap();
stream1.read(buf).await.unwrap();
stream2.write(buf).await.unwrap();
stream3.read(buf).await.unwrap();
```

同一个 `KcpHandle` 可以建立多个传输流，使用 `connect()` 向远端发起连接，或使用 `accept()` 从远端接受连接。`KcpStream` 实现了 `AsyncRead` 和 `AsyncWrite`, 可以与其他异步传输流一样使用 `read()` `write()` 读写。

`KcpHandle` 一旦 drop 将释放所有资源，并将其上建立的所有传输流全部强行关闭（不再收发任何包），请确保使用 `KcpStream` 时 `KcpHandle` 在作用域内。

对 `KcpStream` 的 drop 将使其优雅停机（四次挥手，等待超时），但仍然建议使用 `close()` 方法显式地关闭流。

## 编译

编译需要使用 rust 工具链。

```shell
git clone https://github.com/black-binary/ap-kcp.git
cd ap-kcp
make
```

如果你需要静态链接，可以换用 `x86_64-unknown-linux-musl` 或者其他 musl 工具链

```shell
make x86_64-unknown-linux-musl
```

然后去喝一杯咖啡。

## 细节

### 规格

AP-KCP 封包结构
| 2字节          | 1字节        | 2字节                             | 4字节             | 4字节          | 4 字节                     | 2字节          | len 字节  |
| -------------- | ------------ | --------------------------------- | ----------------- | -------------- | -------------------------- | -------------- | --------- |
| 流ID stream_id | 指令 command | 可用接收窗口大小 recv_window_size | 时间戳　timestamp | 包序号sequence | 接收窗口起始序号 recv_next | 包数据长度 len | 数据 data |

加密封包结构

|                        | n 字节           | 16 字节  | 12字节           |
| ---------------------- | ---------------- | -------- | ---------------- |
| 明文封包(n 字节)       | 明文(AP-KCP封包) | 无此字段 | 无此字段         |
| 密文封包(n+16+12 字节) | 密文             | 标签 tag | 一次性密钥 nonce |

无论 AP-KCP 包是否相同，每一次发送均重新生成一次 nonce 并重新加密。

加解密操作均为原地(in-place)，全程零内存分配。

### 特点

AP-KCP 与 KCP 一样，基于不可靠包传输建立可靠流式传输，保留了 KCP 的优化策略：

* 所有数据包都包含接受窗口信息

    每个数据包都描述了发送该包的主机的接受窗口起始和结束位置，帮助对方判断封包接受情况并快速移动发送窗口。

* 快速重传

    当某个发送窗口中的包，若该包未接受到相应ACK包，但其后超过一定数量的包均得到ACK回复，则无需等待超时直接重传。

同时做出了一些改进

* 优化的首部大小

    AP-KCP 通过减少`stream_id`，`len`等字段的长度的方式（因为现实中没有人会同时建立42亿个连接，或者发送单个长度为4GB的数据包），压缩原版的24字节首部至19字节。

* 改进的 ACK 包结构

    原版KCP版本的每个ACK包（包含时间戳和序号）需要单独发送，每次24字节，AP-KCP将其压缩在同一个ACK包中，每增加一个ACK段只需8字节。

* 封包粘连

    无论何时，发送的封包如果过短，则会尝试与其他封包粘连在同一个底层封包中发送（同一个 UDP 包，包含多个 KCP 包）。

* 快速应答

    若状态机中积累的 ACK 数量过多，则跳过当前心跳间隔直接发送响应。

* 简化的控制命令

    AP-KCP 移除了原版的两个窗口探查指令，简化为三种控制命令

    * PUSH，数据推送，包含发送方欲传输数据

    * ACK，数据收到响应，表明发送方已收到某些数据

    * PING，保持存活，用于替代窗口探查，同步窗口信息和保持连接活跃

* 快速连接建立，可靠连接断开

    AP-KCP 建立连接无需握手，接收方收到序号为0的包则直接建立连接，以此消除握手延迟并提升启动的传输速率。断开时采用类似TCP四次挥手的模式，保证断开时所有链路中的数据均被传输完成。

* 激进的拥塞控制策略（仍有优化空间）
  
    AP-KCP 基于丢包计算发送窗口，若丢包率不超过一定值则以指数增加发送窗口，否则减少。因此发送窗口将容忍一定的丢包率并维持窗口大小在较高水平。

* 密码学支持

    基于 `ring` 提供了下面几种 AEAD 密码支持：

    * aes-256-gcm

    * aes-128-gcm

    * chacha20-poly1305

    每个AP-KCP包均被加密，密文随着 Tag 和 Nonce 一起发送，三个部分任何字节出现错误均无法被解密。加密后的封包可通过随机性测试，以此绕过 ISP 的探测和 QoS 限制。

* 前向错误纠正（待实现）

* 高效异步 IO

    使用 Rust 和 smol，拥有更高的并发效率和计算效率，同时兼顾稳定和安全性。默认情况下，会根据 CPU 数量启用对应的线程数量，并由异步执行器调度各协程执行异步操作。

## 其他

这个项目是我的计算机网络课程的课程设计，目前还很 Buggy，请不要过于自信地部署使用，或是用于渗透等非法用途。代码参考了原始 C 语言实现，tokio-kcp 和 mkcp。

使用它进行长时间高强度的网络传输约等于进行 DDoS，链路上各节点的负载会很高，使用前请三思。

请网络中心的老师们不要打我，都是你们换的长城宽带逼的。
