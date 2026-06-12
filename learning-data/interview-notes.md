# 服务端开发与考核速答框架

> 随学随记，每个概念学完后立即记录，不遗漏。
> 格式：过程 → 原理 → 坑点 → 一句话兜底

## plan-01-basics 基础篇

### TCP 三次握手

**过程：**

- 第一次：客户端发 `SYN`，`seq=x`，进入 `SYN_SENT`
- 第二次：服务端回 `SYN+ACK`，`seq=y`、`ack=x+1`，进入 `SYN_RCVD`
- 第三次：客户端回 `ACK`，`seq=x+1`、`ack=y+1`，双方进入 `ESTABLISHED`
- 注意：`SYN` 标志位本身要消耗一个序号，故 `ack` 都是对方 seq+1

**原理：**

- 本质是**双向同步初始序列号 ISN**，不是简单"确认连接"
- 走完三次后，双方同时确认四件事：我能发、我能收、对面能发、对面能收
- 这是建立可靠双向信道的最小步数

**为什么两次不行：**

- ① 历史失效 SYN 报文延迟到达会让服务端误建连，资源被白白挂起
- ② 两次只能确认客户端 ISN（`x`），服务端 ISN（`y`）没有任何包确认

**坑点：**

- 第三次 ACK 丢失：客户端已 `ESTABLISHED`，服务端仍 `SYN_RCVD`；客户端发的首个数据包带 `ack=y+1` 可隐式补救
- SYN flood 不是被三次握手"防住"，恰恰是利用半连接队列被打满
- 真正的 SYN flood 防御靠 `tcp_syncookies`、增大 `tcp_max_syn_backlog`、限流

**关键参数：**

- 半连接队列：`tcp_max_syn_backlog`
- 全连接队列：`min(somaxconn, listen() backlog)`
- 抓包命令：`tcpdump -i any -nn 'tcp[tcpflags] & (tcp-syn|tcp-ack) != 0'`

**一句话兜底：** 三次握手是双向同步 ISN 的最小步数，少一次服务端 ISN 没人确认，可靠性塌方。

### TCP 四次挥手

**过程（主动关闭方视角）：**

- 第一次：客户端发 `FIN`，`seq=u`，进入 `FIN_WAIT_1`
- 第二次：服务端回 `ACK`，`ack=u+1`，服务端进入 `CLOSE_WAIT`，客户端进入 `FIN_WAIT_2`
- 第三次：服务端发完剩余数据后发 `FIN`，`seq=w`，进入 `LAST_ACK`
- 第四次：客户端回 `ACK`，`ack=w+1`，进入 `TIME_WAIT`，等 2MSL 后才 `CLOSED`
- 服务端收到第四次 ACK 立即 `CLOSED`

**状态机：**

- 主动方：`ESTABLISHED → FIN_WAIT_1 → FIN_WAIT_2 → TIME_WAIT → CLOSED`
- 被动方：`ESTABLISHED → CLOSE_WAIT → LAST_ACK → CLOSED`

**原理：**

- TCP 是**全双工**，两个方向各关一次共 4 次
- 第二次 ACK 和第三次 FIN 不能合并：服务端收到 FIN 后**还可能有数据要发**，所以先回 ACK 安抚客户端，发完数据再发自己的 FIN
- 半关闭状态：客户端在 `FIN_WAIT_2` 时不再发但仍能收

**TIME_WAIT 为什么等 2MSL（默认 120s）：**

- ① 保证最后一个 ACK 丢失时服务端能重传 FIN，自己有机会再 ACK 一次
- ② 让本次连接的延迟报文在网络中消亡，避免被新连接（同四元组）当成自己的数据

**坑点 1 — CLOSE_WAIT 堆积：**

- 99% 是应用没正确调 `close(fd)`
- 排查路径：`netstat -anp | grep CLOSE_WAIT` 定位进程
- 再用 `lsof -p <pid>` 看 fd、`jstack <pid>` 看线程栈
- 重点查代码 `try/catch` 是否漏了 `finally` 关流

**坑点 2 — TIME_WAIT 过多：**

- 场景：高并发短连接 + 服务端主动关闭
- 查看：`netstat -an | grep TIME_WAIT | wc -l`
- 解决（按工程优先级）：
  - 架构：让对端主动关（如 Nginx → 后端时让后端先关）
  - 配置：开 `tcp_tw_reuse` + 加大 `ip_local_port_range`
  - 应用：用 HTTP keep-alive、连接池
- ⚠️ **不要开** `tcp_tw_recycle`，NAT 环境会丢包，Linux 4.12 已删除

**易混点：**

- `tcp_tw_reuse`：复用 TIME_WAIT 端口给**新出向连接**，安全
- `tcp_tw_recycle`：快速回收（含入向），危险，已废弃
- `close()` vs `shutdown()`：前者减引用计数到 0 才关；后者按方向关，可只关写

**一句话兜底：** 四次挥手是双向独立关闭全双工连接，主动方在 TIME_WAIT 等 2MSL 防 ACK 丢失和延迟报文串包；CLOSE_WAIT 堆积找应用代码，TIME_WAIT 过多让对端主动关或开 tcp_tw_reuse。

### OSI 七层与 TCP/IP 五层模型

**两套模型对应：**

- OSI 七层：物理 / 数据链路 / 网络 / 传输 / **会话 / 表示** / 应用
- TCP/IP 五层：物理 / 数据链路 / 网络 / 传输 / 应用（会话+表示并入应用）
- 工程默认用 TCP/IP 五层视角

**每层职责与数据单元：**

- 物理层：传输比特流，单位 `Bit`
- 数据链路层：相邻节点可靠传输 + MAC 寻址，单位 `Frame`
- 网络层：路由选址 + 跨网传输，单位 `Packet`
- 传输层：端到端可靠传输 + 流量控制，单位 `Segment`（TCP）/ `Datagram`（UDP）
- 应用层：给应用提供服务接口，单位 `Message`

**每层协议与设备：**

- 应用层：`HTTP` `HTTPS` `DNS` `FTP` `SMTP` `SSH`
- 传输层：`TCP` `UDP`（端口号在这层）
- 网络层：`IP` `ICMP` `ARP` `OSPF` `BGP` ；设备 = **路由器**、防火墙
- 数据链路层：`Ethernet` `PPP` ；设备 = **交换机**、网桥、网卡
- 物理层：设备 = 集线器、中继器
- ⚠️ ARP 在 PDF 中漏列，记一下

**报文封装过程：**

- 发送：自上而下层层加头
- `HTTP数据` → `+TCP头` → `+IP头` → `+Eth头+FCS` → 比特流
- 接收：自下而上层层剥头
- 每层只看自己那层的头，不解析上层数据

**分层工程价值：**

- 解耦复杂度：每层单一职责，可独立演进（如 HTTP/3 把传输层从 TCP 换成 QUIC）
- 可替换性：物理介质换光纤/5G 上层无感
- 协议复用：HTTP 和 SMTP 共用 TCP，不重复造可靠传输
- 故障定位：按层走 `ping → traceroute → nslookup → telnet → curl`

**易混点：**

- 交换机（L2，看 MAC）不能跨子网；路由器（L3，看 IP）才能
- L4 负载均衡：看 TCP/IP，速度快，看不到 HTTP 内容
- L7 负载均衡：解析 HTTP，能按 URL/Header 路由，开销大
- IP 分片在**网络层**做（MTU 1500），不是 TCP 层

**一句话兜底：** OSI 是 7 层参考模型，TCP/IP 是 5 层工程实现；每层加一个头实现单一职责，路由器在 L3 看 IP 跨子网、交换机在 L2 看 MAC 同网段，排查按层走。

### TCP 与 UDP 区别及应用场景

**六维核心对比：**

- 连接性：TCP 面向连接（三次握手）；UDP 无连接，直接发包
- 可靠性：TCP 可靠（确认+重传+排序+去重）；UDP 不可靠，发了就不管
- 顺序性：TCP 有序（按序号重组）；UDP 无序，先到先处理
- 流控/拥塞控制：TCP 有；UDP 无
- 传输单位：TCP 字节流（无边界）；UDP 数据报（**保留消息边界**）
- 首部开销：TCP 20-60 字节；UDP **固定 8 字节**

**首部字段差异：**

- UDP 首部 4 个字段：源端口、目的端口、长度、校验和
- TCP 首部多出来的全是为了"可靠+有序+流控"
- 6 个标志位：`URG` `ACK` `PSH` `RST` `SYN` `FIN`
- 序号/确认号实现重传与排序，窗口实现流控
- 可选项可放时间戳、SACK 等优化

**典型协议归属：**

- 基于 TCP：`HTTP` `HTTPS` `FTP` `SMTP` `SSH` `MySQL` `Redis` `Kafka`
- 基于 UDP：`DNS` `DHCP` `SNMP` `NTP` `RIP`、音视频、游戏、IoT
- HTTP/3 用 QUIC 跑在 UDP 上，是关键转折点

**DNS 为什么用 UDP：**

- 数据量小（<512 字节），不需要分片
- 延迟敏感，不能承受 TCP 三次握手开销
- 无状态，超时重发即可，服务端能扛百万级 QPS
- 例外：响应 >512 字节 或 区域传送（AXFR/IXFR）时转 TCP

**坑点：**

- UDP "保留消息边界"是优势：一次 `sendto` 一个完整报文
- TCP 字节流没边界，应用层必须自己设计分隔（定长/分隔符/长度前缀）
- UDP **校验和只能检测损坏，不能恢复数据**；错了直接丢弃
- TCP 校验和也只检测，但靠"确认+超时重传"实现可靠性
- IPv4 的 UDP 校验和可填 0 不校验；IPv6 强制必须校验

**选型决策：**

- 选 TCP：能容忍延迟，但不能容忍数据错乱/丢失（订单、IM 正文）
- 选 UDP：要快、能容忍丢包（音视频、游戏、监控指标）
- 折中：UDP + 应用层自建可靠传输（QUIC、KCP、Raknet）

**HTTP/3 + QUIC 的革命：**

- 解决 TCP 队头阻塞（丢一个包后面全卡）
- 解决内核态 TCP 难升级（QUIC 在用户态实现）
- 标配多路复用 + 0-RTT 重连 + 连接迁移

**一句话兜底：** TCP 用复杂度换可靠（连接+确认+重传+有序），UDP 用简单换速度（无连接+无校验恢复+保留边界）；DNS/音视频/游戏选 UDP，HTTP/数据库/IM 正文选 TCP，HTTP/3 用 QUIC 把可靠性从内核搬到用户态。

### TCP 滑动窗口与拥塞控制

**流控 vs 拥塞控制（最易混）：**

- 流控：解决"接收方处理不过来"，端到端，靠 `rwnd`，由接收方在 ACK 里告知
- 拥塞控制：解决"网络处理不过来"，全网视角，靠 `cwnd`，由发送方推断
- **实际发送量 = `min(rwnd, cwnd)`**

**滑动窗口结构（发送方视角）：**

- 发送缓冲区分四段：已发送已 ACK / 已发送未 ACK / 可发送未发送 / 不能发送
- 中间两段合起来是滑动窗口
- 收到 ACK 时窗口右滑，左边界永远不退（累积确认）

**零窗口探测：**

- 接收方 buffer 满，回 `rwnd=0`
- 发送方启动持续计时器，定期发 1 字节探测包问"还能收吗"
- 真实故障：消费者卡住不读 socket，看似网络问题实为应用阻塞

**拥塞控制四大算法：**

- 慢启动：`cwnd=1 MSS`（现代 IW10），每收 ACK +1，**每 RTT 翻倍**（指数）
- 拥塞避免：`cwnd ≥ ssthresh` 时切入，每 RTT +1（线性）
- 快重传：连续收到 **3 个 dup ACK** 立即重传丢失包，不等 RTO 超时
- 快恢复：丢包时不回 `cwnd=1`，而是 `ssthresh=cwnd/2`、`cwnd=ssthresh`，直接进入拥塞避免

**为什么"快"重要：**

- RTO 超时通常 200ms+
- 快重传只需一个 RTT
- 快恢复避免连接彻底重启慢启动雪崩

**现代拥塞算法演进：**

- `Reno` / `NewReno`：经典四算法，高带宽下慢
- `Cubic`：Linux 默认，三次函数代替线性，恢复快；基于丢包，长肥管道次优
- `BBR`：Google 2016，基于"带宽 × RTT"主动测带宽，弱网提升 2-25 倍
- 切换：`sysctl -w net.ipv4.tcp_congestion_control=bbr`

**TCP 可靠传输 6 机制：**

- 应用数据分割（按 MSS）
- 数据包编号（序号）
- 校验和（检测损坏）
- 流量控制（rwnd）
- 拥塞控制（cwnd）
- 超时重传（RTO 自适应）

**坑点：**

- Nagle 算法 + 延迟 ACK 同时开会有 200ms 延迟死锁，低延迟场景必须 `TCP_NODELAY`
- 高带宽长链路要调 `tcp_rmem` `tcp_wmem`，否则 cwnd 涨不起来
- `cwnd` 不是配额，是在途未确认数据的瞬时上限

**排查工具：**

- `ss -nti`：实时看每个连接的 `cwnd` `ssthresh` `rtt` `retrans`
- `tcpdump`：抓 dup ACK 看是否触发快重传
- `nstat -a`：内核计数器，看 retransmit 次数

**一句话兜底：** 滑动窗口靠 rwnd 做端到端流控，拥塞控制靠 cwnd 在慢启动/拥塞避免间切换；3 个 dup ACK 触发快重传+快恢复让 cwnd 减半，超时才回到慢启动；现代云厂商切 BBR 主动测带宽，弱网吞吐翻倍。

### TCP 粘包问题与解决方案

**核心认知（先校准）：**

- 粘包是**伪问题**，不是 TCP bug
- TCP 是字节流协议，从设计上就没有"消息"概念
- "粘包/半包"本质是应用层没定义边界
- UDP 是数据报协议，天生有边界，所以没粘包

**两个根因：**

- 发送方：Nagle 算法把小包合并发送（攒够 MSS 或上一个 ACK 到才发）
- 接收方：内核 buffer 累积多包，应用 `recv` 一次拿到多个消息字节流
- 关闭 Nagle：`setsockopt TCP_NODELAY`

**三种工程解法：**

- 方案 1 定长包：每条固定字节，简单但浪费、不灵活
- 方案 2 分隔符：`\r\n` `\0` 等，灵活但**正文必须转义**，二进制不适用
- 方案 3 长度前缀（TLV）：固定字节包头记长度，最通用最工业级

**长度前缀的工业代表：**

- `Dubbo`：16 字节定长 header（magic + 类型 + 序列化方式 + reqId + body 长度）
- `HTTP/1.1`：`Content-Length` 头标识 body 长度
- `HTTP/2`：每帧 9 字节头（24 位长度 + 类型 + flags + stream ID）
- `MQTT`：1-4 字节可变长度编码（小消息更省）
- `Redis RESP`：`*3\r\n$5\r\nhello\r\n` 长度+分隔符组合
- `gRPC`：5 字节 frame header（压缩标志 + 4 字节长度）

**Netty 内置解码器：**

- `FixedLengthFrameDecoder`：定长
- `DelimiterBasedFrameDecoder` / `LineBasedFrameDecoder`：分隔符
- `LengthFieldBasedFrameDecoder`：**最常用**，五参数 `lengthFieldOffset` / `lengthFieldLength` / `lengthAdjustment` / `initialBytesToStrip` / `maxFrameLength`
- 业务代码 `channelRead` 里收到的就是完整消息

**坑点：**

- 分隔符方案二进制内容必崩，图片/语音必须用长度前缀
- 长度字段统一用大端（网络字节序）
- 必须设 `maxFrameLength` 上限，防止恶意 4GB 撑爆服务端
- Protobuf 不自带边界，外面要套长度前缀
- HTTP/1.1 没粘包是**已经解决了**（头部分隔符 + body Content-Length + chunked），不是没问题

**HTTP/1.1 三种边界组合：**

- 请求/响应头：`\r\n\r\n` 分隔符
- 请求/响应体：`Content-Length: N` 长度前缀
- 流式响应：`Transfer-Encoding: chunked`，每 chunk 自带长度

**一句话兜底：** TCP 粘包是伪问题，根因在应用层没定义边界；三种解法定长/分隔符/长度前缀，工业级首选长度前缀；Netty 用 `LengthFieldBasedFrameDecoder` 一行代码搞定，UDP 因数据报协议天生没这毛病。

### TCP 报文格式与字段含义

**整体结构：**

- TCP 段 = 首部 + 数据
- 首部固定 20 字节（无可选项），最多 60 字节（含可选项）
- 按 32 bit（4 字节）一行布局

**9 个核心字段：**

- 源端口 / 目的端口：各 16 bit，组成**四元组（源IP+源端口+目的IP+目的端口）**唯一标识连接
- 序号 `seq`（32 bit）：**对字节计数**，标识本段第一字节在流中位置；2^32-1 后回绕
- 确认号 `ack`（32 bit）：期望收到的下一字节序号 = 已收最后字节 +1；累积确认；只有 ACK=1 时有效
- 首部长度（4 bit）：单位是 **4 字节字**，最大 15 → 60 字节；无可选项时为 5 → 20 字节
- 保留位（6 bit）：必须为 0，现代 ECN 借用其中 2 位
- 标志位（6 bit）：`URG` `ACK` `PSH` `RST` `SYN` `FIN`
- 窗口大小（16 bit）：接收窗口 rwnd，最大 64KB，可通过 **Window Scale** 扩到 1GB
- 校验和（16 bit）：覆盖**伪首部+TCP头+TCP数据**，跨层校验防 IP 错投
- 紧急指针（16 bit）：仅 URG=1 时有效，现代几乎不用

**6 标志位含义：**

- `URG`：紧急指针有效（几乎不用）
- `ACK`：确认号有效（除第一个 SYN 外都置 1）
- `PSH`：接收方立即推给应用（telnet 等交互场景）
- `RST`：重置连接，立即终止不走四次挥手
- `SYN`：发起连接（三次握手前两次）
- `FIN`：关闭方向（四次挥手）
- 口诀：建连看 SYN，挥手看 FIN，异常看 RST，确认看 ACK

**SYN/FIN 消耗序号：**

- SYN 和 FIN 不带数据但**各消耗一个序号**
- 三次握手：客户端 ISN=x，发完 SYN 后下次数据从 seq=x+1 开始
- 服务端 ack=x+1 表示"收到了 SYN 标志位序号 x"

**重要可选项：**

- `MSS`：握手时协商最大段大小，默认 1460 = 1500 MTU - 20 IP - 20 TCP
- `Window Scale`：窗口扩大因子，让 16 bit 窗口能表达 GB 级
- `SACK`：选择性确认，告知对方"x-y 范围已收"，避免不必要重传
- `Timestamps`：时间戳，精确测 RTT + PAWS 防序号回绕

**RST 触发场景：**

- 连接到没监听的端口
- 进程崩溃（内核代发 RST）
- keep-alive 探测失败
- `SO_LINGER=0` + `close()` 强制 RST
- 对已关闭 socket 发数据

**坑点：**

- 首部长度单位是 4 字节字，4 bit 最大 15 而不是 60
- 窗口 64KB 上限是字段位宽，实际靠 Window Scale 扩展
- 校验和包含**伪首部**（IP 和协议号），是跨层校验
- VPN/隧道场景 MSS 失准要 `iptables clamp`
- 紧急指针、URG 现代不用，防火墙会清掉

**抓包工具：**

- `tcpdump -nn -X -i any port 80`：16 进制看头部
- `Wireshark`：图形化字段解析 + TCP Stream 重组

**一句话兜底：** TCP 首部 20-60 字节，9 个核心字段实现可靠+有序+流控；四元组定连接、seq/ack 按字节累积确认、6 标志位驱状态机、窗口字段+扩大因子撑流控、可选项 MSS/SACK/时间戳是现代 TCP 的关键。

### UDP 报文格式

**结构：**

- UDP 首部固定 8 字节，按 32 bit 一行，共两行
- 字段：源端口(16) / 目的端口(16) / 长度(16) / 检验和(16)
- 首部之后直接跟数据，无可选项、无标志位

**四字段详解：**

- 源/目的端口：各 16 bit，TCP 和 UDP 端口号空间独立互不干扰
- 长度：UDP 首部+数据总字节数，最小 8（空数据报），最大 65535
- 实际数据上限：65535 - 8(UDP头) - 20(IP头) = 65507 字节
- 检验和：端到端，覆盖**伪首部 + UDP首部 + 数据**

**伪首部（12 字节，不传输，仅参与检验和计算）：**

- 源 IP(4) + 目的 IP(4) + 0(1) + 协议号 17(1) + UDP 总长度(2)
- 目的：跨层校验，IP 层错投到错误主机能被检测出来
- IPv4 检验和可选（填 0 不校验），IPv6 强制

**vs TCP 首部：**

- 大小：UDP 固定 8 vs TCP 20-60 字节
- 缺失：无序号/确认号（不重传）、无窗口（不流控）、无标志位（无状态机）、无可选项
- 设计取舍：首部精简到极致，可靠性交给应用层

**坑点：**

- UDP 端口与 TCP 端口独立，别混淆
- IPv4 可关闭检验和（填 0），IPv6 不行
- 伪首部不传输，计算完就丢弃

**一句话兜底：** UDP 首部固定 8 字节仅 4 字段，检验和通过伪首部跨层校验；精简设计的代价是零可靠性保障，适合 DNS/音视频等低延迟场景。

### IP 报文格式

**结构：**

- 固定 20 字节 + 可选项（最多 40），共 5 行 × 32 bit
- 第1行：版本(4) + 首部长度(4) + TOS(8) + 总长度(16)
- 第2行：标识(16) + 标志(3，DF/MF) + 片偏移(13)
- 第3行：TTL(8) + 协议(8) + 首部检验和(16)
- 第4行：源 IP(32)
- 第5行：目的 IP(32)

**关键字段：**

- 首部长度：单位 4 字节字，min=5(20B) / max=15(60B)
- 总长度：IP 首部+数据总字节，max 65535
- 标识：同源数据报所有分片共享，用于重组
- TTL：最大跳数（非秒），每跳-1，到 0 丢弃+ICMP，`traceroute` 原理
- 协议：TCP=6、UDP=17、ICMP=1
- 首部检验和：**仅校验 IP 首部**，不校验数据；每跳重算（TTL 变）

**分片机制：**

- 标志 DF=1 不分片，超 MTU 则丢弃+ICMP → PMTUD
- 标志 MF=1 还有后续分片
- 片偏移：单位 8 字节，数据长度必须是 8 的倍数
- 例：4000B 数据过 MTU=1500 → 3 片（1480+1480+1040）

**坑点：**

- 首部长度单位是 4 字节字（不是字节），4 bit 最大 15 对应 60B
- TTL 是跳数不是秒数
- IP 检验和只验首部，数据校验是上层的事
- `tcpdump` 抓 IP 层：`tcpdump -nn -v ip`

**一句话兜底：** IP 首部 20 字节 5 行，核心任务寻址+分片+选路；TTL 防环路每跳-1，首部检验和只验首部每跳重算，数据正确性由 TCP/UDP 各自负责。

### 以太网报文格式

**帧结构：**

- 帧首部 14B：目的 MAC(6) + 源 MAC(6) + 类型(2)
- 数据：46 ~ 1500 字节
- 帧尾 FCS：4 字节 CRC32（网卡硬件计算/校验）
- 整帧 64 ~ 1518 字节

**MAC 地址：**

- 48 bit（6 字节），出厂固化，`aa:bb:cc:dd:ee:ff` 格式
- 全 `ff:ff:ff:ff:ff:ff` = 广播
- 首字节最低位 1 = 组播
- 仅同子网有效，跨路由器被替换（IP 不变）

**MTU：**

- 以太网 MTU = 1500 字节，仅指数据部分
- 超过 MTU → IP 层分片（不是以太网做）
- DF=1 + 超 MTU → 丢弃 + ICMP → PMTUD
- 不同链路 MTU 不同：PPPoE 1492，拨号 576

**最小帧限制：**

- 最小帧 64 字节（CSMA/CD 冲突检测要求）
- 数据至少 46 字节：64 - 14(头) - 4(FCS)
- ARP(28B) 不够要填充 18 字节

**坑点：**

- MTU 是 payload 上限，不含首部和 FCS
- FCS 校验失败静默丢弃，上层无感知
- 跨路由器 MAC 变 IP 不变——分层解耦核心体现

**一句话兜底：** 以太网帧 L2 层，14B 头+46~1500 数据+4B FCS；MAC 同网段寻址跨路由替换，MTU=1500 仅计数据，FCS 硬件 CRC32 校验。

### HTTP 协议版本演进

**HTTP 1.0 (1996)：**

- 每次请求独立建连（短连接），用完即断
- 无状态，无 Host 头（一台机器一个域名）

**HTTP 1.1 (1999) 五大改进：**

- 长连接 Keep-Alive 默认开，`Connection: close` 才关
- Host 头：虚拟主机基石，Nginx 靠它区分站点
- 缓存升级：ETag/If-None-Match 精确校验，Cache-Control 避开时钟偏差
- 断点续传：`Range: bytes=1000-` + `206 Partial Content`
- 管道化：不等响应连续发请求，但 FIFO 导致 HOL——实际废弃

**HTTP 2.0 (2015) 四大革命：**

- 二进制分帧：9 字节帧头 + payload，告别文本协议
- 多路复用：一个 TCP 多 Stream 并行，响应可乱序（解决应用层 HOL）
- HPACK：静态字典 + 动态字典 + Huffman，Header 压缩 85%+
- Server Push：服务端主动推，但 Chrome 已移除

**HOL 阻塞的三层递进：**

- HTTP 1.x 管道化 → FIFO 响应排队 → 应用层 HOL
- HTTP/2 多路复用 → 解决应用层 HOL
- TCP 丢包 → 后续 Segment 缓存等待 → TCP 层 HOL 仍在
- HTTP/3（QUIC/UDP）→ 各 Stream 独立传输 → 彻底消灭 HOL

**坑点：**

- 1.1 管道化浏览器基本不开，别用
- 2.0 的 Server Push 已被主流浏览器废弃
- 2.0 多路复用 ≠ HOL 全解决，TCP 层丢包依然卡

**一句话兜底：** HTTP 1.0→1.1 长连接+Host+缓存，1.1→2.0 二进制+多路复用+HPACK；应用层 HOL 被 2.0 解决，TCP 层 HOL 等 HTTP/3(QUIC)。

### HTTP 与 HTTPS 区别

**本质：**

- HTTPS = HTTP over TLS，在 HTTP 和 TCP 间插 TLS 层

**五维对比：**

- 端口：80 vs 443
- 传输：明文 vs TLS 加密
- 认证：无 vs CA 证书链
- 完整性：无 vs TLS MAC
- 性能：1 RTT vs +TLS 握手（TLS1.3 共 2 RTT）

**HTTPS 三个"能"：**

- 加密（防窃听）：所有报文 TLS 加密，抓包只能看 `TLS Application Data`
- 认证（防冒充）：CA 证书链校验域名+有效期
- 防篡改（防修改）：TLS 记录层 MAC 检测任何字节改动

**混合加密：**

- 非对称（RSA/ECDSA）传对称密钥，慢但安全
- 对称（AES）传业务数据，快
- 不纯非对称：大数据量加解密太慢
- 不纯对称：密钥怎么安全分发是鸡生蛋问题

**性能代价：**

- HTTP：1 RTT（仅 TCP）
- HTTPS TLS 1.2：3 RTT（TCP + TLS 2RTT）
- HTTPS TLS 1.3：2 RTT（TCP + TLS 1RTT）
- 会话复用（Session Ticket）可跳过 TLS 握手

**坑点：**

- 内网 HTTP 也不安全（ARP 欺骗可劫持明文）
- 证书过期/域名不匹配浏览器直接拦
- 性能瓶颈在 TLS 握手阶段，不在加密

**一句话兜底：** HTTP 明文 80，HTTPS 443 用 TLS 加密+认证+防篡改；混合加密：非对称安全交换对称密钥，对称(AES)高效传业务数据。

### HTTPS 握手流程（TLS 握手）

**三个目标：**

- 协商加密套件
- 安全交换对称密钥
- 认证服务器身份（CA 证书）

**TLS 1.2 RSA 握手（2 RTT）：**

- ① ClientHello：加密套件列表 + 随机数1
- ② ServerHello + 证书(含公钥)：选套件 + 随机数2
- ③ 客户端验证证书，生成 PreMaster Secret，用服务端公钥加密发送
- ④ 双方从 PreMaster + 随机数1+2 派生会话密钥（Master Secret）
- ⑤⑥ ChangeCipherSpec + Finished 双向验证
- 缺陷：无前向安全性（私钥泄露 → 解开历史所有 PreMaster）

**TLS 1.2 ECDHE 握手（2 RTT，主流）：**

- ① ClientHello：加密套件列表 + 随机数1
- ② ServerHello + 证书 + 临时 ECDHE 公钥A：选套件 + 随机数2
- ③④ 客户端发临时 ECDHE 公钥B，双方 ECDHE 计算共享 PreMaster
- ⑤⑥ 派生会话密钥 + Finished
- 优势：前向安全性——临时私钥握手后丢弃

**TLS 1.3（2018，1 RTT）：**

- ClientHello 直接带密钥协商参数，1 RTT 完成
- 砍掉 RSA 交换、CBC/RSA/RC4/SHA1 等旧算法
- 仅保留 AEAD（AES-GCM、ChaCha20-Poly1305）
- 支持 0-RTT 重连（预共享密钥）

**坑点：**

- RSA 握手没有前向安全性，生产禁掉
- TLS 1.0/1.1 已被 PCI 安全标准禁用（2020），至少 1.2
- 0-RTT 有重放攻击风险，GET 安全 POST 危险

**一句话兜底：** TLS 握手 RSA 2 RTT 无前向安全、ECDHE 2 RTT 前向安全（临时密钥）、TLS 1.3 精简 1 RTT 砍 RSA 强制 ECDHE。

### 对称与非对称加密

**对称加密：**

- 同一密钥加解密，快（AES 有 CPU 硬件指令集 AES-NI）
- 代表：AES-128/256-GCM（主流）、ChaCha20（移动端）
- 痛点：密钥如何安全分发给对方（鸡生蛋）
- 模式演进：ECB(禁用) → CBC(淘汰中) → GCM(推荐，AEAD 加密+认证)
- DES(56bit)暴力可破已淘汰

**非对称加密：**

- 公私钥成对，公钥加密私钥解密、私钥签名公钥验证
- 代表：RSA(2048-4096bit)、ECDSA(256bit)、Ed25519(256bit)
- 痛点：慢（RSA 比 AES 慢 100-1000 倍）
- 256bit ECDSA ≈ 3072bit RSA 安全性（RSA 有 sub-exponential 攻击）

**混合加密：**

- 握手阶段：非对称（RSA/ECDHE）交换/协商出对称密钥
- 数据传输阶段：对称（AES-GCM）高效加密

**现代推荐：**

- 对称：AES-256-GCM 或 ChaCha20-Poly1305
- 非对称（签名）：Ed25519 > ECDSA > RSA
- 不要再选：DES、3DES、RSA 密钥交换、AES-CBC、AES-ECB

**一句话兜底：** 对称快(AES-GCM)但难分发，非对称慢(ECDSA)但安全交换密钥；HTTPS 混合二者：非对称握手协商、对称传数据。

### GET 与 POST 区别

**三大常见误解（先纠正）：**

- GET 长度 2KB 不是协议限制，是浏览器/Nginx/Tomcat 的实现限制
- POST 不比 GET 更安全，HTTP 下都明文；安全靠 HTTPS
- GET 技术上能写数据，但违反语义且危险（爬虫会触发）

**核心对比：**

- 参数位置：GET URL query string / POST Body
- 语义：GET 读(幂等) / POST 写(非幂等)
- 缓存：GET 可缓存 / POST 默认不缓存
- 书签：GET 可收藏 / POST 不可
- 回退刷新：GET 无害 / POST 浏览器提示确认
- 编码：GET `urlencoded` / POST `multipart/form-data` 或 `application/json`

**HTTP 方法全景：**

- GET(读)、POST(创建)、PUT(全量更新)、PATCH(部分更新)、DELETE(删除)
- HEAD(仅取头)、OPTIONS(CORS 预检)、TRACE(诊断，禁用)、CONNECT(代理隧道)

**工程选型：**

- GET：搜索/分页/筛选等无害参数，放 URL 方便分享
- POST：表单/JSON/文件上传，放 Body 避开 URL 长度瓶颈和日志泄露

**一句话兜底：** GET URL 传参幂等读可缓存，POST Body 传参非幂等写不缓存；长度限制是实现的锅，安全靠 HTTPS 不靠方法。

### HTTP 状态码

**五大分类：**

- 1xx：收到，处理中
- 2xx：成功
- 3xx：去别的地方拿（重定向）
- 4xx：你的问题（客户端错误）
- 5xx：我的问题（服务端错误）

**关键状态码：**

- 200 OK / 201 Created(新建资源) / 204 No Content(无body) / 206 断点续传
- **301 永久重定向**(浏览器缓存新地址，慎用！) / **302 临时重定向** / **304 缓存未过期**(无body，省带宽)
- **400 Bad Request** / **401 未认证(没登录)** / **403 无权限(登录了)** / 404 Not Found / 405 方法错误 / 429 限流
- 500 内部异常 / **502 网关收到无效上游响应** / 503 服务暂不可用 / **504 网关超时**

**易混对比：**

- 301 vs 302：301 浏览器永久缓存跳转收不回，302 每次问原地址
- 401 vs 403：401"没登录"(无凭证)，403"没权限"(有凭证但不够)
- 502 vs 504：502 上游返回了错误，504 上游没返回（超时）

**坑点：**

- 301 一旦被浏览器缓存，想取消重定向几乎不可能
- 304 无 body，仅返回少量头，是缓存带宽优化的关键
- 5xx 错误不要返回详细堆栈给客户端（安全风险）

**一句话兜底：** 1-5 分类记忆，301/302 重定向缓存是坑，401=没登录 403=没权限，502=上游坏了 504=上游超时。

### 重定向与转发

**本质：**

- Redirect：浏览器端跳转，302 + Location 头，浏览器自动发第二次请求
- Forward：服务端内部跳转，Servlet 容器内部转交，浏览器无感知

**五维对比：**

- 地址栏：Redirect 变化 / Forward 不变
- 请求次数：Redirect 2 次 / Forward 1 次
- 跨站点：Redirect 可以 / Forward 不行
- request 共享：Redirect 不能 / Forward 可以
- 代码：`response.sendRedirect()` / `request.getRequestDispatcher().forward()`

**选型：**

- Redirect：登录跳转、跨站跳转、PRG 防重复提交（写操作）
- Forward：MVC 内部渲染 Controller→JSP、统一错误页（读操作）

**PRG 模式（Post-Redirect-Get）：**

- POST 处理后 → Redirect → 浏览器 GET 新页面
- 用户 F5 刷新的是安全的 GET，不会重复 POST
- 这是 Web 开发防重复提交的核心模式

**坑点：**

- Redirect 后 request 属性全丢失，跨请求传数据用 URL 参数或 Session
- Forward 地址栏不变，用户刷新会重放上游逻辑

**一句话兜底：** Redirect 浏览器两次请求地址栏变可跨站，Forward 服务端一次请求内部转交 request 共享；写操作 PRG(Redirect) 防重复提交。

### Cookie 与 Session

**为什么需要：**

- HTTP 无状态，每次请求独立
- Cookie/Session 给 HTTP 加上"记忆"

**Cookie（客户端）：**

- 小段文本，`Set-Cookie` 响应头存，`Cookie` 请求头带
- 容量 4KB，客户端可见可篡改
- 安全三属性：`HttpOnly`(防XSS) + `Secure`(仅HTTPS) + `SameSite=Lax`(防CSRF)

**Session（服务端）：**

- 数据存服务端(内存/Redis)，客户端只存 Session ID
- 机制：浏览器带 `JSESSIONID` Cookie → 服务端查 Session 表
- 弊端：单机内存重启丢、分布式机器间不共享

**Cookie vs Session：**

- 位置：客户端 vs 服务端
- 安全：低(可见) vs 高(仅ID在客户端)
- 容量：4KB vs 服务端资源
- 开销：不占服务端 vs 每次查Session

**分布式 Session 三方案：**

- Sticky Session：固定用户到同机器（简单但下线丢）
- Redis 集中存储：标准方案，所有机器共享
- JWT 无状态：Token 自含信息，服务端不存，吊销难

**安全铁三角（生产必设）：**

```
Set-Cookie: token=xxx; HttpOnly; Secure; SameSite=Lax
```

**一句话兜底：** Cookie 客户端 4KB 每次自动带不安全，Session 服务端存仅 ID 放客户端更安全；生产 Cookie 设 HttpOnly+Secure+SameSite，分布式 Session 用 Redis 或 JWT。

### 浏览器输入 URL 全过程

**六阶段：**

- ① URL 解析：提取协议(https)、域名、路径、参数
- ② DNS 解析：浏览器缓存→hosts→路由器→ISP DNS→根域→.com→域名，UDP 53
- ③ TCP 三次握手：SYN→SYN+ACK→ACK，1 RTT
- ④ TLS 握手(TLS1.2 ECDHE 2 RTT, TLS1.3 1 RTT)
- ⑤ HTTP 请求：构建请求→Nginx→应用服务器→返回 HTML
- ⑥ 浏览器渲染：HTML→DOM树 + CSS→CSSOM树 → Render Tree → 布局 → 绘制 → 合成

**RTT（DNS 缓存命中时）：**

- HTTP：TCP 1 RTT + 请求 1 RTT = 2 RTT
- HTTPS TLS1.2：TCP 1 + TLS 2 + 请求 1 = 4 RTT
- HTTPS TLS1.3：TCP 1 + TLS 1 + 请求 1 = 3 RTT

**渲染关键路径：**

- HTML 解析构建 DOM 树（遇 `<script>` 阻塞）
- CSS 解析构建 CSSOM 树（不阻塞 DOM 但阻塞渲染+阻塞 JS）
- DOM + CSSOM 合并为 Render Tree
- 布局(Layout)算盒模型 → 绘制(Paint) ←→ 合成(Composite)
- 优化：CSS 放 `<head>`，JS 放 `</body>` 前

**一句话兜底：** URL→DNS→TCP(1 RTT)→TLS(2/1 RTT)→HTTP(1 RTT)→渲染(DOM+CSSOM→Layout→Paint)；CSS 不阻塞 DOM 但卡渲染，JS 阻塞 DOM 解析放底部。

### 进程与线程

**定义：**

- 进程：资源分配最小单位，独立地址空间
- 线程：CPU 调度最小单位，共享进程地址空间
- 协程：用户态轻量线程，程序自行调度，OS 无感知

**内存布局：**

- 线程共享：堆、方法区（类信息/静态变量/常量池）
- 线程独享：栈、本地方法栈、程序计数器
- 栈必须独享：每个线程有独立调用链

**对比：**

- 地址空间：进程独立 / 线程共享
- 通信：进程需 IPC / 线程直接共享变量
- 创建开销：进程大(复制页表) / 线程小(栈+PC)
- 切换开销：进程慢(切 CR3 + TLB 刷新) / 线程快 / 协程最快(几十ns)
- 隔离性：进程强 / 线程弱(System.exit 全挂)

**上下文切换：**

- 线程切换：保存/恢复寄存器、PC、栈指针，不切页表
- 进程切换：+切 CR3 寄存器 + TLB 刷新（全失效），短期性能下降
- 协程切换：用户态，仅少量寄存器，不经内核

**协程：**

- Go goroutine / Java Virtual Thread / Kotlin coroutine
- 线程等 I/O 时让给其他协程执行
- I/O 密集场景一个线程跑上万协程

**坑点：**

- fork 初期 COW 共享只读内存，写入时才复制
- 线程数 > CPU 核心数反慢（上下文切换开销）
- 线程公式：CPU 密集 N+1，I/O 密集 N×2

**一句话兜底：** 进程独立地址空间隔离强切换慢，线程共享地址空间切换快隔离弱，协程用户态调度切换不经内核。

### 进程间通信方式 IPC

**为什么需要：**

- 进程独立地址空间，一个进程不能访问另一个的内存
- 需要 OS 提供的 IPC 机制才能协作

**六大 IPC：**

- 管道(Pipe)：匿名(父子)/命名(任意)，FIFO 半双工字节流，`ps aux | grep java`
- 信号(Signal)：仅发信号编号不传数据，`kill -9` / `Ctrl+C`
- 消息队列：有格式消息可分类取，克服管道无格式+信号不传数据
- **共享内存：最快 IPC，0 次拷贝**，两进程映射同一物理内存直接读写（需配信号量同步）
- 信号量：P/V 原子操作计数器，不做数据传输只做同步互斥
- Socket：跨网络通信，TCP/UDP/Unix Domain Socket

**速度对比：**

- 管道/消息队列：2 次拷贝（用户→内核buffer→对方用户）
- 共享内存：0 次拷贝（直接读写同一物理内存）
- Socket：跨网络开销最大，同机 Unix Domain Socket 比 TCP 快

**一句话兜底：** 六大 IPC：管道/信号/消息队列/共享内存(最快0拷贝)/信号量(同步)/Socket(跨网)；共享内存+信号量是高频大数据量标配。

### 用户态与内核态

**两种态：**

- 用户态(Ring 3)：运行应用程序，受限内存访问，不能执行特权指令
- 内核态(Ring 0)：运行OS内核，访问所有内存+外设，能执行特权指令

**为什么分：**

- 隔离保护：防止程序窃取其他进程内存或直接操作硬件

**用户态→内核态三种切换：**

- ① 系统调用（主动）：`read`/`write`/`fork`，`syscall`指令触发，内核服务完后 `sysret` 返回
- ② 异常（被动）：除零/缺页/非法地址，CPU 切内核处理（SIGSEGV 或修复）
- ③ 外设中断（被动）：网卡收包/磁盘IO完成/时钟中断，硬件异步通知 CPU

**一次 read() 的旅程：**

- 用户 `read()` → glibc 设寄存器 → `syscall` → 内核查fd/验权/读盘 → 拷数据 → `sysret` → 返回
- 模式切换本身开销小（几十-几百周期），真正开销在内核处理（如磁盘IO）

**时钟中断与调度：**

- 时钟中断定期触发 → CPU 切内核 → 内核检查时间片用完 → 触发调度切进程
- 没有时钟中断，死循环用户程序永远霸占 CPU

**一句话兜底：** Ring 0 内核态上帝模式、Ring 3 用户态沙箱；三种切换：系统调用(主动)、异常(被动)、中断(被动)；时钟中断是抢占式调度的基础。

### 进程地址空间

**四大区（从低到高）：**

- 代码区(Text)：程序二进制指令，只读，进程间可共享
- 静态区：Data(已初始化全局/静态变量) + BSS(未初始化，OS自动赋0，不占ELF文件空间)
- 堆区(Heap)：`malloc`/`new`，向高地址增长，程序员手动释放/GC，线程共享
- 栈区(Stack)：局部变量/函数参数/返回地址，向低地址增长，编译器自动管理，线程独享

**堆 vs 栈：**

- 速度：栈快(一条SP指令) / 堆慢(找空闲块+元数据)
- 大小：栈小(8MB) / 堆大(GB级)
- 生命周期：栈=函数返回即毁 / 堆=手动free或GC
- 碎片：栈无(LIFO) / 堆有
- 方向：栈↓ / 堆↑（相向而行，可在中间相遇）

**线程共享：**

- 共享：堆、静态区(Data+BSS)、代码区
- 独享：栈、程序计数器

**坑点：**

- 递归+大局部变量→Stack Overflow(栈8MB爆)
- Java对象全在堆上，`new`频繁会GC压力大
- 指针变量在栈，指向的对象在堆

**一句话兜底：** 低→高 Text(代码只读)→Data+BSS(全局/静态)→Heap(↑,线程共享)→Stack(↓,线程独享)；栈快小短命，堆慢大灵活。

### 内存管理方式：分段、分页、段页式

**为什么需要：**

- 虚拟地址→物理地址映射，给进程"独占连续内存"的幻觉

**分段管理：**

- 按程序逻辑分：代码段、数据段、堆栈段，大小可变
- 地址：(段号, 段内偏移) → 段表 → 物理地址
- 优点：无内碎片（段按需分配）
- 缺点：有外碎片（段大小不一致，空闲空间不连续不凑够大段）

**分页管理：**

- 固定 4KB 页(pages)和页框(frames)，离散分配
- 地址：(页号, 页内偏移) → 页表 → 物理地址
- 优点：无外碎片（所有页框等大，任何页放任何框）
- 缺点：有内碎片（最后一页用不满），页表自身占内存

**内碎片 vs 外碎片：**

- 内碎片 = 分配内部未用完（页内空间浪费）
- 外碎片 = 空闲空间散布各分配块间，单个空洞不够用

**段页式管理：**

- 先分段，再段内分页：(段号, 页号, 页内偏移)
- 现代 x86 保护模式采用，Linux 实把段基址设 0 主要靠分页

**MMU 与 TLB：**

- MMU：CPU 内硬件，自动虚拟→物理翻译
- TLB：MMU 内缓存页表项，命中 ~1 周期，不命中需查内存页表
- 大页(Huge Page 2MB/1GB)：扩大 TLB 覆盖 512 倍，命中率飙升

**一句话兜底：** 分段无内碎有外碎，分页无外碎有内碎，段页式结合；MMU 翻译 TLB 缓存，大页提 TLB 命中率。

### 页面置换算法

**为什么要置换：**

- 物理内存满需加载新页，必须踢旧页
- 目标：减少缺页中断（磁盘I/O 毫秒级代价）

**三种算法：**

- FIFO：先入先出，简单但有 Belady 异常（增页框反增缺页），实际不用
- LRU：踢最近最少使用的页，利用时间局部性，性能好
  - 纯 LRU 开销大（每次访问更新链表），OS 用 Clock 近似
  - Clock（二次机会法）：循环链表 + 访问位，0 踢 1 清 0 绕圈
- OPT：踢未来最远才用的页，理论最优但不可实现（需预知未来），作评价基准

**Belady 异常（FIFO 独有）：**

- 3 框 9 次缺页，4 框反而 10 次
- 根因：FIFO 不看使用频率只看入队顺序

**Java LRU 实现：**

- `LinkedHashMap(accessOrder=true)` + `removeEldestEntry()`
- get/put 自动把条目移链表尾，超容量踢链表头

**一句话兜底：** FIFO 有 Belady 异常不用，LRU 利用时间局部性 OS 用 Clock 近似，OPT 理论最优不可实现；Java 用 LinkedHashMap 做 O(1) LRU。

### 死锁

**定义：**

- 多线程/进程相互持有对方所需资源，形成循环等待

**四个必要条件（缺一不锁）：**

- 互斥：资源一次只能一个用（synchronized）
- 请求与保持：持旧要新，阻塞不放旧（持A等B）
- 不可剥夺：不能被抢走已有的锁
- 循环等待：P1→P2→...→Pn→P1 等待环

**破坏死锁四方案：**

- 破坏互斥：CAS 无锁操作（AtomicInteger），仅简单场景
- 破坏请求保持：tryLock 一次性拿全部，拿不到释放重试
- 破坏不可剥夺：超时释放已持锁，DB deadlock 超时回滚
- **破坏循环等待：资源编号按序申请——最实用**
  - 转账场景：先锁 min(账户A, 账户B)，再锁 max(账户A, 账户B)
  - 零额外开销，不重试不回滚，所有加锁场景通用

**排查：**

- Java：`jstack <pid>` 直接显示 `Found 1 deadlock`
- MySQL：`SHOW ENGINE INNODB STATUS` → LATEST DETECTED DEADLOCK

**一句话兜底：** 死锁需四条件(互斥/请求保持/不可剥夺/循环等待)同时满足；破坏循环等待（按序加锁）最实用。

### Java 面向对象三大特性

**封装：**

- 属性私有化 + public 方法暴露，隐藏内部实现
- 防止外部直接 `balance = -10000`，统一校验/日志/事务入口

**继承：**

- `extends` 单继承父类，`implements` 多实现接口
- 子类复用父类属性方法，可扩展新功能
- **组合优于继承**：继承强耦合父类实现，组合只依赖接口

**多态：**

- 同一方法调用，不同对象表现不同行为
- 两种路径：继承 + 重写(@Override) / 接口 + 实现
- 本质：运行时动态绑定，编译期只做类型检查不选方法

**静态绑定 vs 动态绑定：**

- 静态绑定(编译期)：方法重载，根据参数类型选，静态分派
- 动态绑定(运行时)：方法重写/接口实现，JVM 找实际对象类型，动态分派
- JVM 栈帧"动态连接"：运行时符号引用 → 直接引用

**重写规则：**

- 不能抛比父类更宽的检查异常（里氏替换）
- 访问权限不能比父类更窄

**一句话兜底：** 封装隐藏数据控制访问；继承单extends多implements(组合优于继承)；多态动态绑定=重写/接口(运行时)，≠重载(编译期)。

### Java 与 C++ 区别

**六维对比：**

- 继承：Java 单 extends+多 implements / C++ 多继承
- 指针：Java 无（用引用，安全）/ C++ 有指针（可运算，危险）
- 内存：Java JVM 自动 GC / C++ 手动 new/delete
- 编译：Java → .class 字节码 JVM JIT / C++ → 原生机器码
- 平台：Java 跨平台 / C++ 编译特定平台
- 安全：Java 数组越界检查、无裸内存 / C++ 越界/野指针/缓冲区溢出风险

**本质：**

- Java 牺牲底层控制和极端性能，换安全(GC/无指针)和跨平台(JVM)

**一句话兜底：** Java 无指针有引用 GC 跨平台安全，C++ 有指针手动内存管理能力更强但风险高。

### 多态实现原理（复习 c-01-27）

> 本质是动态绑定：编译看左边（引用类型，静态分派），运行看右边（实际对象，动态分派）。JVM invokevirtual 从实际类型搜方法表。

**一句话兜底：** 编译静态分派(重载)，运行动态分派(重写/接口)；invokevirtual 从子类往上搜方法表。

### static 和 final 关键字

**static：**

- 属性：类级别，所有实例共享一份，类加载时初始化(一次)，`ClassName.field`
- 方法：类名直接调用，无 `this`，只能访问静态成员
- 代码块：`static{}` 类加载执行一次

**final：**

- 基本类型变量：值不可改
- 引用类型变量：不可指向新对象，但**对象内容可改**（`final List + add=合法`）
- 方法：子类不能重写，private 方法隐式 final
- 类：不可继承（`String`、`Integer`），所有方法隐式 final
- 私有构造器：另一种"不可继承"方式（内部类无效）

**static + final = 常量：**

- `public static final int MAX_SIZE = 100`
- 命名：SCREAMING_SNAKE_CASE

**一句话兜底：** static 类级共享无 this，final 锁引用不锁内容(基本值不变/引用不新指/方法不可重写/类不可继承)。

### 抽象类与接口

**抽象类：**

- `abstract` 修饰，不能 `final`，不能实例化
- 有构造方法，有成员变量，可有普通方法 + 抽象方法
- 单继承 `extends`

**接口：**

- 抽象方法集合，多实现 `implements`
- 无构造方法，变量仅 `public static final` 常量
- Java 8+：`default` 方法(默认实现) + `static` 方法(工具方法)

**相同点：**

- 都不能实例化
- 都可定义抽象方法，子类/实现类必须实现

**不同点：**

- 构造：抽象类有 / 接口无
- 变量：抽象类各种类型 / 接口仅 static final 常量
- 继承：抽象类单继承 / 接口多实现
- 方法：抽象类可有普通方法 / 接口 Java8+ 可有 default

**选型：**

- 需共享状态(成员变量) + 构造逻辑 → 抽象类
- 只需行为契约 → **接口优先**（更灵活，多实现）
- 抽象类不可替代：成员变量、构造方法、protected/private 封装

**一句话兜底：** 抽象类 is-a 有构造有状态单继承，接口 can-do 无构造无状态多实现；Java8 default 模糊边界；优先接口。

### 泛型与泛型擦除

**泛型：**

- 参数化类型，用于类(`class Box<T>`)/接口(`Comparable<T>`)/方法
- 编译期类型安全，消除手动强转

**类型擦除（Java 伪泛型）：**

- 编译后类型参数被去掉，JVM 只看到原始类型
- `<T>` → Object，`<T extends X>` → X
- `List<String>` 和 `List<Integer>` 运行时都是 `ArrayList.class`
- 原因：兼容 Java 5 之前的旧代码，只改 javac 不改 JVM

**擦除导致的限制：**

- 不能 `instanceof List<String>`（运行时类型已擦除）
- 不能 `new T[10]`（泛型数组）
- 不能用基本类型 `List<int>`（必须 Object 子类型）
- 静态上下文不能用类参数 `static T value`
- 反射可绕过泛型：`add.invoke(list, 123)` 绕过 `List<String>` 约束

**通配符（PECS）：**

- `? extends T`：Producer 读，只能 get 不能 add
- `? super T`：Consumer 写，可以 add，get 出来是 Object

**一句话兜底：** Java 伪泛型编译擦除兼容旧代码；T→Object，`<T extends X>`→X；PECS: `? extends` 读 `? super` 写。

### 反射原理与应用

**定义：**

- 运行时动态获取类元信息并操作（方法/字段/构造器/注解）

**原理：**

- JVM 加载类 → 方法区 Class 对象 → 反射 API 映射为 Method/Field/Constructor
- 获取 Class：`类名.class` / `obj.getClass()` / `Class.forName(全限定名)`

**核心 API：**

- `getMethods()`：所有 public 方法（含继承）
- `getDeclaredMethods()`：本类所有方法（含 private，不含继承）
- `method.invoke(obj, args)`：动态调用
- `setAccessible(true)`：绕过权限访问 private

**应用场景：**

- Spring IoC：`@Component` 扫描 → 反射创建 Bean → 注入
- JDK 动态代理 AOP：`Proxy + InvocationHandler + method.invoke`
- ORM：MyBatis SQL 结果 → Entity 字段 setter
- SPI：`Class.forName("com.mysql.cj.jdbc.Driver")`

**性能：**

- 反射慢（权限检查 + 装拆箱 + 少内联），框架缓存 Method 对象/用 MethodHandle 优化
- 启动反射一次缓存，运行时直接拿——Spring 的策略

**一句话兜底：** 反射运行时读 Class 元信息动态调用；getMethods(含继承)≠getDeclaredMethods(本类全部)；Spring/AOP/ORM 的基石。

### Java 异常体系

**Throwable 家族：**

- Throwable → Error + Exception
- Error：JVM 级严重问题(OOM/StackOverflow)，不捕获
- Exception：
  - RuntimeException(非受检)：NPE/越界/ClassCast，程序逻辑 bug 应修复代码
  - CheckedException(受检)：IOException/SQLException，必须 try-catch/throws

**设计哲学：**

- RuntimeException = 程序员 bug（代码逻辑可避免）
- CheckedException = 外部不确定性（程序不可控，必须显式处理）

**最佳实践：**

- 不要吞异常（catch 后不做任何处理）
- 不要用异常做控制流（用正则预检代替 catch NumberFormatException）
- 捕获后记录上下文 + 保留根因链(cause)
- Spring `@Transactional` 默认仅回滚 RuntimeException+Error

**一句话兜底：** Throwable→Error(JVM级不捕获)+RuntimeException(非受检修代码)+CheckedException(受检必须处理)。

### ArrayList 与 LinkedList

**ArrayList：**

- 底层数组，O(1) 随机访问
- 默认容量 10，扩容 1.5 倍（`Arrays.copyOf`）
- 中间插入/删除 O(n)（数组拷贝移位），尾部追加均摊 O(1)

**LinkedList：**

- 底层双向链表，O(n) 随机访问
- 头尾增删 O(1)，中间插入也需 O(n)（先遍历定位）
- 实现 Deque 接口，可做栈/队列/双向队列（官方推荐代替 Stack）

**核心对比：**

- 读多：ArrayList O(1)
- 头尾增删多：LinkedList O(1)
- 遍历：ArrayList 连续内存 CPU 缓存友好，比 LinkedList 快几十倍
- 都线程不安全

**坑点：**

- LinkedList 中间插入不是 O(1)（定位位置需 O(n)）
- 预知大小用 `new ArrayList<>(size)` 避免多次扩容
- 大量尾追加用 ArrayList（均摊 O(1) + 少 GC），LinkedList 每次 new Node 导致 GC 压力

**一句话兜底：** ArrayList 数组 O(1) 随机读扩容 1.5 倍；LinkedList 双向链表头尾 O(1) 做 Deque；多数场景 ArrayList 胜（缓存友好）。

### fail-fast 与 fail-safe

**fail-fast（快速失败）：**

- 遍历时 modCount 改变→抛 ConcurrentModificationException
- 原理：迭代器 `expectedModCount` vs 集合 `modCount`
- 代表：ArrayList、HashMap（java.util 包）
- 安全删除：`Iterator.remove()` / `removeIf`(Java8+) / 普通 for 倒序
- 禁止：`foreach` + `list.remove()` → CME

**fail-safe（安全失败）：**

- 遍历前复制快照，在拷贝上遍历，原集合修改不影响
- 代表：`CopyOnWriteArrayList`、`ConcurrentHashMap`（j.u.c 包）
- 缺点：遍历看不到修改（旧快照）、写操作每次复制开销大
- 适用：读多写少（配置、白名单），能接受读"稍旧"

**一句话兜底：** fail-fast: modCount+CME(java.util)，用 Iterator.remove/removeIf/倒序 for 安全删；fail-safe: 快照拷贝(j.u.c)，读多写少。

### HashMap 详解

**数据结构：**

- 数组(table) + 链表(Node) + 红黑树(TreeNode)
- 默认容量 16，负载因子 0.75
- 链表 ≥8 且数组 ≥64 → 树化；≤6 → 退化为链表

**为什么容量始终 2^N：**

- `(n-1) & hash` 代替 `hash % n`，位运算快 10 倍
- 扩容时 `hash & oldCap` 把链表拆两半（原位 / 原位+oldCap）

**哈希函数：**

- `(key.hashCode()) ^ (key.hashCode() >>> 16)`
- 高 16 位异或低 16 位，让高位参与低位运算，减少碰撞

**put 步骤：**

- ① 算 hash ② table 空→resize 初始化 ③ `(n-1)&hash` 定位桶
- ④ 桶空直接放 / TreeNode 走树插入 / 链表尾插+查重覆盖
- ⑤ 链表≥8→treeifyBin（需数组≥64） ⑥ size>threshold→resize 2 倍扩容

**JDK 1.7 vs 1.8：**

- 1.7：数组+链表，头插法，扩容 rehash，并发扩容死循环 CPU 100%
- 1.8：数组+链表+红黑树，**尾插法**，`hash & oldCap` 拆分，修死循环（但依然非线程安全）
- 树化阈值 8：泊松分布下概率 < 千万分之一，防御 DoS 碰撞攻击

**坑点：**

- 1.8 尾插法修复死循环 ≠ 线程安全，多线程用 ConcurrentHashMap
- 树化需要数组 ≥64（否则先扩容不树化）

**一句话兜底：** 数组+链表+红黑树，2^N 容量位运算取模，1.7 头插死循环 1.8 尾插+树化解 DoS；线程安全用 ConcurrentHashMap。

### ConcurrentHashMap

**线程安全演进：**

- Hashtable：全表 synchronized，极低
- CHM 1.7：Segment 分段锁(ReentrantLock，默认 16 段)，不同段可并发
- CHM 1.8：CAS + synchronized，锁粒度到桶首节点，结构同 HashMap 1.8

**CHM 1.8 核心机制：**

- 读 get()：全程无锁，volatile 保证可见性
- 写 put()：桶空→CAS 无锁写；桶非空→synchronized 锁桶首节点
- 扩容：多线程协同 transfer，不单线程扛

**1.7 vs 1.8：**

- 结构：1.7 Segment[]+HashEntry[]+链表 → 1.8 Node[]+链表+红黑树
- 锁：1.7 Segment(ReentrantLock) → 1.8 CAS+synchronized
- 粒度：1.7 固定 16 段 → 1.8 桶级(随扩容增长)
- 并发度：1.7 固定 → 1.8 动态

**为什么禁止 null key/value：**

- 多线程下 get(null) 无法区分 key 不存在 vs value 本为 null

**一句话兜底：** CHM 1.7 Segment 分段锁 16 并发度→CHM 1.8 CAS+synchronized 桶级锁，get 无锁 volatile 可见，禁止 null。

### 序列化与反序列化

**定义：**

- 序列化：对象 → 字节流（持久化/传输）
- 反序列化：字节流 → 对象

**Java 原生：**

- `Serializable` 标记接口
- `ObjectOutputStream.writeObject` / `ObjectInputStream.readObject`
- `transient` 字段不参与序列化（密码/缓存）
- `static` 字段不参与（属于类不属于对象）
- `serialVersionUID`：不一致抛 `InvalidClassException`，显式声明避免类变更导致历史数据反序列化失败

**应用场景：**

- Redis RDB 快照、Dubbo/gRPC 远程通信、分布式 Session、深拷贝

**局限与替代：**

- Java 原生：性能差、仅 Java、安全漏洞（反序列化 RCE）
- JSON(Jackson/Gson)：文本跨语言，Web 用
- ProtoBuf：二进制极快，gRPC 标配
- Kryo/Hessian/FST：高性能二进制

**一句话兜底：** 对象↔字节流；transient 不序列化；serialVersionUID 控制兼容；生产用 JSON/ProtoBuf 替代原生。

### String 详解

**不可变性：**

- JDK9: `private final byte[] value` + coder
- 不可变 → 常量池复用、hashCode 缓存、安全、线程安全

**常量池：**

- 字面量 `"hello"` 编译期入池，运行时加载复用
- `new String("hello")` 强制堆新对象（1-2 个对象）
- `intern()`：若池有则返引用，无则入池
- JDK7 后常量池移入堆

**String vs StringBuilder vs StringBuffer：**

- String：不可变，安全，少量拼接/做 key
- StringBuilder：可变，不安全，单线程大量拼接（首选）
- StringBuffer：可变，安全(synchronized)，多线程拼接
- 循环拼接禁止用 `+`（每次创建新对象）

**关键优化：**

- 常量折叠：`"a"+"b"` 编译期直接 → `"ab"`，1 个对象
- JDK9 Compact Strings：纯 Latin-1 字符 1 字节/char（原 2 字节），省 ~10-15% 内存

**一句话兜底：** String final byte[] 不可变，常量池复用 intern；拼接用 Builder(Buffer 安全)；循环禁止 +。

### 单例模式

**四种实现：**

- 饿汉式：`static final INSTANCE = new Singleton()`，类加载创建，线程安全但浪费
- DCL：`volatile + synchronized + 双 null 检查`，懒加载，volatile 防指令重排
- 静态内部类(Holder)：JVM 类加载线程安全，懒加载，最优雅，无需 volatile
- **枚举(首选)**：`enum Singleton { INSTANCE }`，防反射、防序列化、最简单

**volatile 为什么必须（DCL）：**

- `new` 非原子：分配→初始化→赋值，可能重排为分配→赋值→初始化
- 线程 B 看到非 null 拿到未初始化对象 → 崩

**反射与序列化安全：**

- 前三种反射可破(`setAccessible + newInstance`)，枚举不行(JVM 禁)
- 前三种序列化需 `readResolve()` 返回 INSTANCE，枚举自带

**一句话兜底：** 饿汉(类加载)/DCL(volatile+双检)/Holder(懒加载最优雅)/枚举(防反射防序列化首选)。

### 工厂模式

**定义：**

- 定义创建产品接口，子类决定何种产品，将创建逻辑从调用方抽离

**简单工厂：**

- 一个工厂类 switch/if 根据参数 new 不同产品
- 优点：解耦调用方；缺点：加产品需改工厂（违反开闭原则）

**工厂方法（GoF 标准）：**

- 工厂也抽象化：`Factory接口 + 具体工厂(AlipayFactory/WechatFactory)`
- 加新产品加新类不动旧代码（符合开闭原则）
- Spring IoC ≈ 巨型工厂

**一句话兜底：** 简单工厂 switch(违反开闭)→工厂方法每产品一工厂(开闭原则)；Spring IoC 是工厂模式实践。

### 抽象工厂模式

**定义：**

- 提供一个接口创建**一族相关产品**，保证产品族内部一致（如 Dark 主题所有控件）

**vs 工厂方法：**

- 工厂方法：一个工厂→一个产品（AlipayFactory→Alipay）
- 抽象工厂：一个工厂→一族产品（DarkUIFactory→DarkButton+DarkTextField+DarkDialog）
- 切换产品族 = 换一个工厂实例

**优缺点：**

- 优点：产品族内部约束，风格统一
- 缺点：加新产品需改所有工厂类（侵入大）

**一句话兜底：** 工厂方法管一个产品，抽象工厂管一族产品；切换主题换工厂实例；加新产品族成员改所有工厂。

### 基础篇综合考察题

**初始化顺序：**

- 父静态→子静态→父实例块→父构造→子实例块→子构造

**重载 vs 重写：**

- 重载：同类不同参，编译期静态分派
- 重写：父子同签，运行时动态分派（@Override，异常不能比父类宽）

**Object 核心方法：**

- `toString`：默认类名@hashCode，一般重写
- `equals`/`hashCode`：equals 等则 hashCode 必等（反之不成立），重写 equals 必须重写 hashCode
- `clone`：需 Cloneable 接口，默认浅拷贝
- `getClass`：反射入口
- `notify`/`wait`：线程通信
- `finalize`：Java9+ 废弃

**hashCode 契约（HashMap 原理）：**

- HashMap 先用 hashCode 定位桶，再用 equals 比较
- equals 相等但 hashCode 不同 → HashMap 放不同桶 → containsKey 找不到 → bug

**包装类缓存：**

- Integer/Byte/Short/Long：[-128, 127]
- Character：[0, 127]
- Boolean：true/false
- `Integer==` 在缓存内 true，外 false

**一句话兜底：** 初始化静态→实例父→子；重载编译期重写运行时；重写 equals 必须重写 hashCode；Integer 缓存 -128~127。

---

## plan-02-jvm JVM篇

### JVM 运行时数据区域

**线程共享：**

- 堆(Heap)：对象实例+数组，GC 主战场，新生代(Eden+S0+S1)+老年代，逃逸分析可标量替换栈上分配
- 方法区(MetaSpace, JDK8+)：类信息/常量池/静态变量/JIT 代码，直接内存，取代 PermGen 解决类膨胀 OOM

**线程私有：**

- 虚拟机栈：栈帧(局部变量表+操作数栈+动态链接+返回地址)，递归深→StackOverflowError
- 本地方法栈：Native 方法，HotSpot 与虚拟机栈合一
- 程序计数器(PC)：下条指令地址，唯一不 OOM 的区域

**JDK8 元空间 vs 永久代：**

- PermGen 堆内固定大小→类多 OOM；MetaSpace 直接内存自动扩容

**坑点：**

- StackOverflowError ≈ 递归太深；MetaSpace OOM ≈ 动态代理类爆炸

**一句话兜底：** 堆(对象GC主战场)+MetaSpace(类信息直接内存)线程共享；栈(栈帧)+PC(唯一不OOM)线程私有。

### 堆内存分配策略

**堆分代结构：**

- 新生代：Eden(80%) + S0(10%) + S1(10%)，复制算法
- 老年代：标记-清除/整理算法，存活率高

**五条分配策略：**

- ① 对象优先 Eden，满→Minor GC，存活进 Survivor
- ② 大对象直接老年代，避免复制拷贝（`PretenureSizeThreshold`）
- ③ 年龄晋升：每熬 Minor GC Age+1，15 岁进老年代（对象头 4 bit 上限）
- ④ 动态年龄：同龄对象 > Survivor 一半→直接晋升，防止撑爆
- ⑤ 空间担保：Minor GC 前查老年代剩余，不够→Full GC→OOM

**为什么分代：**

- 新生代对象短命，复制算法拷贝少高效
- 老年代对象长寿，标记-整理避免大量拷贝
- 弱分代假说：绝大多数对象朝生夕死

**一句话兜底：** Eden→Survivor→Age+1→15岁→老年代；大对象直进；同龄>Survivor一半晋升；空间不够Full GC。

### JIT 即时编译器

**执行模式：**

- 解释器 + JIT 编译器混合（默认），先解释快速启动，热点→JIT 编译为本地机器码

**热点探测：**

- 方法调用计数器 + 回边计数器，超阈值（Client 1500/Server 10000）→ 编译
- HotSpot 名字的由来

**分层编译（Java 7+）：**

- Level 0：纯解释 → Level 1-3：C1（编译快） → Level 4：C2（优化激进）
- C1 启动快优化少，C2 编译慢性能高

**关键优化：**

- 方法内联（省栈帧开销，`final`/`static`/`private` 易内联）
- 逃逸分析（不逃逸→栈上分配/标量替换→减少 GC）
- 锁消除（逃逸分析确认无竞争→去锁）
- 锁粗化（连续小锁合并大锁）
- 死代码消除

**预热问题：**

- 应用启动先慢后快 = JIT 预热
- AOT 提前编译（JDK 9）牺牲 profiling 优化换快速启动

**一句话兜底：** 解释器+JIT 混合；热点探测计数→C1 快速→C2 深度；分层编译提升→机器码；预热导致先慢后快。

### 内存分配方式

**两种堆分配：**

- 指针碰撞：堆规整(Serial/ParNew GC后)，指针后移 O(1)
- 空闲列表：堆碎片(CMS不压缩)，从列表找够大的块

**TLAB（线程本地分配缓冲）：**

- Eden 内每个线程私有小空间（默认 1%）
- TLAB 内指针碰撞无锁分配，用完再向 Eden 申请（低频同步）
- `-XX:+UseTLAB` 默认开

**栈上分配：**

- 逃逸分析→对象不逃逸→标量替换→在栈上分配
- 完全不参与 GC，高频临时对象场景收益大

**一句话兜底：** 堆规整→指针碰撞(Serial)；碎片→空闲列表(CMS)；TLAB 线程私有无锁；栈上分配逃逸分析避 GC。

### 对象创建过程

**五步：**

- ① 类加载检查：常量池符号引用→类是否已加载，否→先加载
- ② 分配内存：指针碰撞/空闲列表/TLAB（详见 c-02-04）
- ③ 初始化零值：全字段置默认零值（int=0, ref=null），保证有默认值
- ④ 设置对象头：Mark Word(hashCode+GC年龄+锁状态)+Klass Pointer(指向Class)+对齐填充(8字节倍数)
- ⑤ 执行 <init>：构造方法+实例初始化块，零值→目标值

**对象头 Mark Word（位复用）：**

- 无锁：hashCode+GC年龄(4bit)
- 偏向锁：线程ID+epoch
- 轻量级锁：指向栈中 Lock Record
- 重量级锁：指向 ObjectMonitor

**最小对象大小：**

- 64 位+指针压缩：Mark Word(8B)+KlassPtr(4B)+Padding(4B)=16 字节

**一句话兜底：** 五步：类检查→分配→零值→对象头(Mark Word+KlassPtr)→<init>；最小 16 字节。

### 对象引用类型：强/软/弱/虚

**四种引用强度：**

- 强引用(Strong)：永不回收，宁愿 OOM，普通引用默认
- 软引用(Soft)：内存不足 GC 回收后还不够才 OOM，`SoftReference`，适合图片缓存
- 弱引用(Weak)：下次 GC 必回收，`WeakReference`，WeakHashMap key / ThreadLocal key
- 虚引用(Phantom)：`get()` 永远 null，只能配合 ReferenceQueue 跟踪对象 GC 后清理资源（NIO 直接内存）

**ThreadLocal 为什么用弱引用：**

- key 用弱引用→外部强引用消失后 GC 自动回收 key→防 key 泄漏
- 但不防 value 泄漏——key 回收后 value 仍强可达变成脏 entry，必须手动 `remove()`

**一句话兜底：** 强(不GC)→软(内存不足GC/缓存)→弱(GC必回收/ThreadLocal key)→虚(不可访问仅跟踪GC/直接内存清理)。

### 类加载过程

**五阶段：**

- 加载：全限定名→字节流→方法区运行时结构→堆上 Class 对象(入口)
- 验证：四层校验——
  - 文件格式(CAFEBABE/版本兼容)
  - 元数据(继承合法/final 不继/抽象实现)
  - 字节码(防恶意操作)
  - 符号引用(存在性/权限，解析阶段发生)
- 准备：static 变量分配内存+赋**零值**（非代码初值）。例外：final static 常量直接赋终值
- 解析：符号引用(`com/User`)→直接引用(内存地址/偏移量)
- 初始化：执行 `<clinit>`——static 赋值+static 块按代码顺序合并执行

**触发时机：**

- new、反射、访问 static 变量/方法、子类初始化触发父类、main 方法类启动

**坑点：**

- `static { x=5; } static int x=3;` → clinit 顺序：先 x=5 后 x=3→最终 3
- 准备阶段的零值 ≠ 初始化阶段的代码值

**一句话兜底：** 加载→验证→准备(零值)→解析(符号→直接)→初始化(clinit=static赋值+块按序执行)；final static 备即赋值。

### 双亲委派机制

**三层架构：**

- Bootstrap：rt.jar 核心类，C/C++ 实现，打印为 null
- Extension：lib/ext/*.jar（Java 9+ → Platform）
- Application：classpath 用户类

**委派流程：**

- 底→上检查缓存（加载过？→直接返回）
- 顶→下尝试加载（Bootstrap→Ext→App→自己 findClass）
- 找不到 → ClassNotFoundException

**两大好处：**

- 防篡改：`java.lang.String` 永由 Bootstrap 加载，classpath 假 String 不生效
- 防重复：所有加载器共享同一个 Object 类定义

**打破双亲委派的三个场景：**

- JDBC SPI：Bootstrap 通过 TCCL 向下委派给 Application 加载驱动
- Tomcat：WebappClassLoader 先己后父，隔离不同 Web 应用
- OSGi：网状加载拓扑，按包级暴露依赖

**一句话兜底：** 双亲委派=先缓存→委父→父找不到自加载；防篡改防重复；SPI(TCCL)/Tomcat(先己后父)/OSGi(网状)打破。

### Tomcat 类加载机制

**为什么打破双亲委派：**

- 应用间类隔离（不同 Spring 版本）
- 热部署（卸载应用→丢弃 WebAppClassLoader→卸载类）

**WebAppClassLoader 六步：**

- ① 查本地 cache → ② 查系统 cache → ③ Ext→Bootstrap（核心类）→ ④ 自己 findClass(WEB-INF/lib) → ⑤ AppClassLoader（公共库）→ ⑥ 抛异常

**vs 标准双亲委派：**

- 标准：自己→App→Ext→Bootstrap→回退
- Tomcat：自己→Ext→Bootstrap→自己→App
- 绕开 AppClassLoader：核心类安全→应用隔离→公共库共享

**一句话兜底：** Tomcat WebAppClassLoader 绕开 App 先 Ext→Bootstrap→自己加载应用类→最后 App 共享公共库，实现隔离与热部署。

### 存活算法与两次标记

**引用计数法（JVM 不用）：**

- 对象计数器 +1/-1，0→可回收
- 致命缺陷：循环引用（A⇋B）计数器永不归零→内存泄漏

**可达性分析（JVM 用）：**

- 从 GC Roots 出发沿引用链搜索，不可达→垃圾
- GC Roots：虚拟机栈引用(局部变量)、静态变量引用、JNI 引用、活动线程、synchronized 持有对象

**两次标记：**

- 第一次标记：不可达对象
- finalize() 检查：未执行过→放入 F-Queue→Finalizer 低优线程执行
- 第二次标记：finalize 后仍不可达→真正回收；期间重连 GC Root→自救成功
- ⚠️ finalize Java 9+ @Deprecated，用 try-finally/Cleaner 替代

**一句话兜底：** JVM 用可达性分析(GC Roots)不用引用计数(循环引用)；两次标记+finalize自救(finalize已废弃)。

### 垃圾回收算法

**复制算法（新生代）：**

- 两块内存，存活对象复制到另一块，清空旧区
- 无碎片，简单高效，Eden:S0:S1=8:1:1（浪费 10%）
- 存活多时拷贝开销大

**标记-清除（CMS 老年代）：**

- 标记存活→清除未标记
- 不移动对象，但产生碎片→分配失败→Full GC

**标记-整理（老年代）：**

- 标记→存活对象移一端→清边界外
- 无碎片，但移动对象 STW 开销大

**分代收集：**

- 新生代=复制（存活率低<10%，拷贝少）
- 老年代=标记-整理/标记-清除（存活率高，不能浪费一半内存）

**SafePoint：**

- 线程安全暂停点，GC 必须等所有线程到达才能 STW
- 一线程迟迟不进→JVM 等死→GC 时间暴涨

**一句话兜底：** 复制(新生代10%浪费无碎)→标记清除(碎片CMS)→标记整理(无碎STW)；分代按存活率选算法。

### GC 类型：Minor / Major / Full

- Minor GC：新生代 Eden 满，频繁快 STW 短
- Major GC：老年代不足，通常伴随 Minor GC
- Full GC：全堆 + MetaSpace，最慢 STW 最长

**Full GC 触发条件：**

- 老年代不足、元空间不足、System.gc()、CMS promotion failure、空间担保失败
- Promotion Failure：Minor GC 存活 > Survivor + 老年代剩余 → Serial Old 单线程长时间 STW

**一句话兜底：** Minor(Eden满快)→Major(老年代)→Full(全堆最慢)；Full GC 五触发避免之。

### 垃圾收集器：Serial/Parallel/CMS/G1/ZGC

**Serial/ParNew：**

- Serial：单线程 STW，客户端小内存（JDK 1.3）
- ParNew：Serial 多线程版，仅配对 CMS

**Parallel Scavenge + Old：**

- 关注吞吐量（运行时间/总时间），自适应策略
- 适合后台批处理/数据分析

**CMS（JDK 5-14，低停顿）：**

- 四阶段：初始标记(STW)→并发标记→重新标记(STW)→并发清除
- 三大缺陷：CPU敏感、浮动垃圾（需预留空间）、碎片（碎片严重→Serial Old 单线程长时间 STW）
- JDK 14 移除

**G1（JDK 9 默认，推荐）：**

- Region 分区（不再连续分代），优先回收垃圾最多的 Region
- 四阶段：初始→并发→最终标记→筛选回收
- 标记-整理不产生碎片，可精准控制停顿 `MaxGCPauseMillis`

**ZGC（JDK 11+，极低延迟）：**

- < 10ms STW，TB 级堆（16TB），全程几乎并发
- 着色指针 + 读屏障，STW 不随堆大小增加

**选型速查：**

- 小内存→Serial；后台批处理→Parallel；JDK8 Web→CMS；JDK9+通用→G1(默认)；极低延迟大堆→ZGC

**一句话兜底：** Serial(单线程)→Parallel(吞吐)→CMS(低停顿/碎片/JDK14移除)→G1(Region/精准/JDK9默认)→ZGC(<10ms/TB)。

### GC 配置与调优思路

**关键参数：**

- `-Xms = -Xmx`：避免动态扩缩 STW
- 堆 = OS 2/3，不超 32GB（指针压缩失效），>8GB 优选 G1
- 必设 `MaxMetaspaceSize` 上限防元空间泄漏
- G1 `MaxGCPauseMillis` 软目标（默认 200ms），非硬保证
- GC 日志：JDK 9+ `-Xlog:gc*`

**调优优先级：**

- ① 灭 Full GC（最致命） ② 控停顿（满足 SLA） ③ 提吞吐（GC < 5-10%）

**常见问题：**

- Minor GC 频繁→加大 Eden / Survivor 溢出→加大 Survivor / 老年代涨快→查缓存泄漏 / Full GC 频繁→加大堆或排元空间 / CMS 碎片→升级 G1

**工具：** GCeasy 上传日志自动分析瓶颈

**一句话兜底：** Xms=Xmx 堆=OS 2/3，>8GB G1，必设 MaxMetaSize；调优三优先：灭 Full GC > 控停顿 > 提吞吐。

### JVM 调优命令：jps/jinfo/jstat/jstack/jmap

- `jps -l -v`：查看 Java 进程 PID + JVM 参数
- `jinfo -flags <pid>`：查看 JVM 启动参数
- `jstat -gc <pid> 1000 10`：GC 统计（盯 FGC/FGCT），-gcutil 看百分比
- `jstack -l <pid>`：线程栈快照，末尾直接报死锁
- `jmap -dump:live,file=heap.hprof <pid>`：导出堆 dump（live 仅存活，前先 Full GC）

**典型排查链路：**

- CPU 高：`top -Hp → 找 nid → printf %x → jstack | grep hex → 定位代码`
- 内存高：`jstat -gc → jmap -dump → MAT → Dominator Tree 找大对象`
- 死锁：`jstack -l | grep deadlock`

**一句话兜底：** jps 找 pid/jstat 看 GC/jstack 查死锁/jmap 导 dump→MAT 分析；CPU 高 top+H→jstack，内存高 jmap→MAT。

### JDK 新特性

**核心演进：**

- JDK 8(LTS)：Lambda+Stream+HashMap 红黑树+MetaSpace
- JDK 9：G1 默认 GC+模块化
- JDK 11(LTS)：ZGC(<10ms STW)+Lambda var
- JDK 14：移除 CMS+弃用 Parallel+SerialOld
- JDK 17(LTS)：分代 ZGC+密封类+Pattern Matching
- JDK 21(LTS)：虚拟线程正式 GA

**LTS 版本链：** 8 → 11 → 17 → 21，企业选型优先 LTS。

**一句话兜底：** 8(Lambda+Stream)→11(ZGC)→14(移CMS)→17(密封类)→21(虚拟线程)；LTS优先。

### 硬件故障排查

**三步原则：**

- ① 隔离（摘除故障机器 Nginx 权重=0）→ ② 保留现场（瞬时+历史）→ ③ 排查

**保留现场七大维度：**

- 网络连接：`ss -antp`（替代 netstat，连接多更快），看 TIME_WAIT/CLOSE_WAIT
- 网络统计：`netstat -s` + `sar -n DEV`，看丢包/网卡跑满
- 进程资源：`lsof -p <pid>`，看打开文件和连接
- CPU：`mpstat` / `vmstat` / `uptime`
- I/O：`iostat -x`，看磁盘 util%/日志写满
- 内存：`free -h`，看 SWAP/SLAB 是否挤占 JVM
- 内核：`dmesg`，检查 OOM Killer 是否杀了进程

**典型排查链路：**

- CPU 高→`top -Hp + jstack`；内存高→`free -h + jstat -gc`；OOM→`dmesg | grep oom`；网络慢→`ss + sar`

**一句话兜底：** 隔离→保留现场→排查；ss>netstat；dmesg 看 OOM Killer；free 看 SWAP；iostat 看 I/O。

### JVM 命令总结（复习 c-02-15 + c-02-17）

**Java 层：** jps(进程)/jstat(GC)/jstack(线程/死锁)/jmap(堆dump)/jinfo(参数)
**OS 层：** top -Hp(线程CPU)/free(内存SWAP)/ss(网络)/iostat(I/O)/dmesg(OOM Killer)

**三种故障链：**
- CPU 高→top -Hp→jstack
- 内存泄漏→jstat -gc→jmap -dump→MAT Dominator Tree
- 死锁→jstack -l→Found deadlock

**排查原则：** 先隔离再排查，先 OS 再 JVM，先 GC 再代码。

### 内存泄漏排查

**定义：**

- 对象不被业务使用但仍被 GC Roots 强引用，GC 无法回收
- 泄漏是因，OOM 是果

**典型现象：**

- Old 区持续上升，Full GC 多轮无明显下降
- gcutil 看 OU 每轮 GC 后不回落到基线

**排查套路：**

- jstat -gc → 观 OU 趋势 → jmap -dump → MAT → Dominator Tree → 找 Retained Heap 最大对象 → GC Root 路径 → 定位泄漏代码
- JDK 9+ jmap 部分功能被 jhsdb 取代
- jstack 不响应时用 `kill -3 <pid>` 替补

**经典泄漏场景：**

- ThreadLocal 不 remove()：线程池线程复用→Entry 强引 value 永不回收
- 集合加对象忘了删：HashMap cache 从不清除
- 监听器注册不解绑

**一句话兜底：** 泄漏=GC Root 不必要引用；jstat→jmap→MAT Dominator Tree→GC Root；ThreadLocal 不 remove 是经典泄漏。

### OOM 故障排查与实战案例

**工具：**

- JDK 9+：jhsdb 替代 jmap（`jhsdb jmap --heap --pid`）
- jmap 连不上：gcore 生成 core dump 兜底

**实战案例：报表系统 OOM 四轮优化**

- Round 1：3GB→6GB 堆，OOM 消失但 GC 5 秒+
- Round 2：三项参数调优——
  - `MaxTenuringThreshold=3`：报表对象长命，快速晋升老年代
  - `CMSScavengeBeforeRemark`：CMS Remark 前先 Minor GC 清新生代减少扫描
  - `ParallelRefProcEnabled`：并行处理弱引用（从 4.5 秒缩短）
- Round 3：8C16G 大堆 Minor GC>1 秒→换 G1 目标 200ms→停顿小运行平滑
- Round 4（治本）：select*→必须字段 + Cache 弱引用 + 限制文件大小/拆分查询

**教训：**

- 加内存缓口气→改代码治根
- 大堆 >6-8GB 用 G1 优过 CMS
- 报表类对象长命→压低 MaxTenuringThreshold 不是坏事




### JUC 调优：HttpClient 连接池阻塞

**案例：**

- fast 接口(100ms)+slow 接口(2s) 共享 HttpClient 连接池→整体不可用
- jstack→大多数线程阻塞在 fast（不是 slow）——具迷惑性
- 真相：slow 占连接池不释放→fast 拿不到连接→进得多积得快→jstack 快照 fast 占多数

**解决方案：**

- 隔离：fast/slow 独立连接池
- 熔断：slow 超时降级返默认值
- 监控：连接池水位+等待队列长度+slow 耗时
- CountDownLatch 控多线程合并

**核心：不同 SLA 请求必须隔离连接池。**


### SWAP 故障排查

**案例：**

- 实例频繁卡顿，GC 时间比其他实例长得多
- vmstat→GC 时 si/so 飙升，free→SWAP 使用率高
- slabtop→dentry 异常高

**根因：**

- 运维 `find / | grep` 全盘扫描→文件元数据缓存到 SLAB
- SLAB 挤占物理内存→JVM 堆被 swap 到硬盘
- GC 访问被 swap 的页→磁盘级延迟→GC 时间暴涨

**解决：** `swapoff -a` 关闭 SWAP

**结论：** 高并发场景 SWAP 是万恶之源，建议禁用。




### Cache 故障排查

**案例 1：HashMap 无界缓存**

- 无上限/无超时/无 LRU→Old 区持续涨→jmap dump→MAT 发现巨型 HashMap
- 修复：Guava Cache + 弱引用 + maximumSize + expireAfterWrite

**案例 2：hashCode/equals 未重写**

- 自定义 Key 没重写 equals+hashCode→put 进去取不出来→隐式泄漏
- 修复：重写 `equals`+`hashCode`（@EqualsAndHashCode）

**案例 3：文件句柄泄漏**

- close 没放 finally→fd 泄漏→`Too many open files`
- 修复：try-with-resources

**一句话兜底：** Cache 泄漏三场景（无界缓存/key缺hashCode-equals/fd泄漏）；修复=Guava+重写hashCode/equals+try-with-resources。




### CPU 飙高排查

**五步法：**

- top(Shift+P 按CPU排序) → top -Hp $pid(找线程) → printf %x(10→16进制) → jstack dump → less grep(定位)

**三种根因：**

- GC 线程忙：堆快满反复无效回收(GC thrashing)→CPU 全耗在 GC→应用假死，根因在内存
- 业务死循环：while(true)没break / HashMap 1.7 并发死循环
- 大量计算：正则回溯/大 JSON 解析/不合理自旋

**关键：CPU 高先看是否 GC 线程在跑，不一定是业务代码问题。**




---

## plan-03-multithreading 多线程篇

### 线程状态

**五种基本态：** 新建→就绪→运行→阻塞→死亡

**Java 六态：**

- NEW：new 未 start
- RUNNABLE：就绪+运行合并
- BLOCKED：等 synchronized 锁（被动）
- WAITING：wait/join/park 无限等（主动）
- TIMED_WAITING：sleep(ms)/wait(ms)/join(ms)
- TERMINATED：run 结束

**BLOCKED vs WAITING：** BLOCKED 等锁被动争，WAITING 主动等通知。




### 线程状态切换

**方法 → 状态：**

- `start()`：NEW→RUNNABLE
- `sleep(ms)`：不释放锁→TIMED_WAITING→时间到自动恢复
- `wait()`：**释放锁**→WAITING→需 notify 唤醒，**必须在 synchronized 中**
- `join()`：不释放锁→WAITING→目标线程终止恢复
- `yield()`：让出 CPU 仍在 RUNNABLE
- `notify/notifyAll`：notify 随机唤醒一个，notifyAll 唤醒所有但只有一个拿锁

**sleep vs wait：**

- sleep(Thread)：不释锁，不需 synchronized，时间到自恢复
- wait(Object)：释放锁，必须 synchronized，等 notify

**一句话兜底：** sleep 抱锁睡(TIMED_WAITING)→wait 放锁等(WAITING, 必须synchronized)；notifyAll 唤醒所有但一拿锁。




### 阻塞唤醒过程

**Monitor 三队列：**

- Entry Set：竞争锁失败→BLOCKED
- Owner：持有锁→RUNNABLE
- Wait Set：wait 后放锁→WAITING
- 流程：Entry Set→Owner→wait→Wait Set→notify→回到 Entry Set 重新竞争锁（不是直接运行！）

**wait 必须 synchronized：** wait 需要"释放锁"，没持有 Monitor 直接抛 IllegalMonitorStateException

**三种创建线程：**

- Thread（继承，不推荐） / Runnable（实现，优先） / Callable（有返回值+可抛异常）

**start vs run：** start()=新线程异步 / run()=当前线程同步

**一句话兜底：** Monitor三队列 Entry→Owner→Wait→notify回Entry重争锁；wait必须synchronized(Object)；Runnable优先Callable有返回值。




### 线程池

**七大参数：**

- corePoolSize(核心)/maximumPoolSize(最大)/keepAliveTime(空闲存活)/workQueue(队列)/threadFactory/handler

**任务流程：**

- core 满→入队→队列满→开到 max→max 满→拒绝策略

**四种拒绝策略：**

- AbortPolicy(抛异常/默认) / CallerRunsPolicy(调用者跑/缓冲) / DiscardOldestPolicy(丢最老) / DiscardPolicy(直接丢)

**禁止 Executors 创建线程池：**

- Fixed(无界队列 OOM) / Cached(无上限线程 OOM) → 手动 ThreadPoolExecutor + 有界队列

**线程数公式：** CPU 密集 N+1 / IO 密集 N*2

**一句话兜底：** 七参数 core→队列→max→拒绝；Abort/CallerRuns 常用；禁止 Executors 用 ThreadPoolExecutor+有界队列。




### 线程安全：CAS / synchronized / ReentrantLock

**CAS：**

- V(内存值) A(预期) B(新值)，V==A 则 V=B，不等则重试
- AtomicInteger 实现，读多写少
- 缺陷：ABA(AtomicStampedReference 版号解) / 自旋浪费 CPU / 只单变量

**synchronized：**

- 实例方法锁 this / 静态方法锁类.class / 代码块锁指定对象
- 对象头 Mark Word，monitorenter/monitorexit
- JDK6+ 锁升级：无锁→偏向(单线程)→轻量(CAS自旋)→重量(OS互斥)
- 自动释放，不可中断，非公平

**ReentrantLock：**

- API 层，CAS 自旋 + volatile，必须手动 unlock(finally)
- 可超时 tryLock / 可中断 lockInterruptibly / 可选公平 / 多 Condition
- vs synchronized：自动 vs 手动/不可中断 vs 可中断/非公平 vs 可选

**公平 vs 非公平：** 公平 FIFO 不饿吞吐低/非公平直接抢吞吐高可能饥饿；默认非公平。




### Java 内存模型 JMM

**JMM 三大特性：**

- 原子性：synchronized(monitorenter/monitorexit)
- 可见性：volatile(写回主内存/读刷新) + synchronized + final
- 有序性：volatile(内存屏障禁止重排) + synchronized(单线程串行)

**volatile：**

- 保证可见性和有序性，**不保证原子性**（i++ 非原子）
- DCL 单例中禁止重排：分配→初始化→赋值不乱序

**锁优化六招：**

- 减锁时间 / 减锁粒度(CHM 分段) / 锁粗化(提循环外) / 读写锁 / CAS 替代 / 锁消除(JIT 去锁)

**ThreadLocal：**

- 每个线程 ThreadLocalMap，key 弱引用，value 强引用
- 线程池复用→不 remove→value 强引永不回收→内存泄漏

**一句话兜底：** JMM 原子(synced)+可见(volatile)+有序(volatile)；volatile 不原子；ThreadLocal 用完 remove。




### JUC 工具类

**AQS：** volatile state + CLH 双向队列，ReentrantLock/CountDownLatch/CyclicBarrier/Semaphore 的基石

- CountDownLatch：倒计数归零→主线程继续，一次性，适合多线程汇总
- CyclicBarrier：等人齐→一起继续，可复用，适合分片处理
- Semaphore：令牌控制并发数，acquire/release，限流用
- CompletableFuture：现代异步编排，supplyAsync→thenApply→thenAccept

**volatile 底层：** lock 前缀 = 内存屏障：禁止重排+强制写回+缓存行失效

**happens-before：** 程序顺序/volatile 写-读/unlock-lock/传递性/线程 start-join

**一句话兜底：** AQS=state+CLH；CountDownLatch(一次倒计数)/CyclicBarrier(可复用等人齐)/Semaphore(令牌限流)；volatile=lock前缀+内存屏障。




---

## plan-04-mysql MySQL篇

### Why MySQL

**NoSQL 四大家：** HBase(列存/分析)/Redis(K-V/缓存)/Neo4j(图/社交)/MongoDB(文档/灵活schema)
**Aerospike：** T级KV，内存索引+SSD，<1ms，广告推荐
**MySQL 为什么主流：** ACID强事务+生态成熟+适应性强(单机→分库分表→主从)
**选型：** 事务→MySQL；高并发缓存→Redis；海量分析→HBase；关系→Neo4j

**一句话兜底：** MySQL 是关系型+ACID首选；NoSQL各有分工；选型看：事务/并发/分析/关系/flexibility。




### ACID 事务

**ACID：** 原子性(undo 回滚)/一致性(undo+redo+约束)/隔离性(锁+MVCC)/持久性(redo)

**四种隔离级别：**
- RU：脏读+不可重复读+幻读 全有
- RC：防脏读，不可重复读+幻读仍有
- RR(MySQL默认)：防脏读+不可重复读，MVCC 快照读防幻读，当前读仍有
- Serializable：全防，串行执行性能最低

**三种并发问题：** 脏读(读未提交)/不可重复读(同事务两次结果不同/update)/幻读(同事务行数不同/insert)

**一句话兜底：** undo=原子/redo=持久/锁+MVCC=隔离；四级RC防脏→RR防不可重复→Serializable全防；MySQL默认RR+MVCC。




### 事务隔离级别与 MVCC

**RC vs RR：**

- RC：每个 SQL 新 ReadView→读最新已提交→不可重复读
- RR(默认)：事务开始生成一次 ReadView→始终看快照→可重复读

**MVCC 实现：**

- 隐藏列：DB_TRX_ID(修改事务ID)+DB_ROLL_PTR(回滚指针→undo链)
- undo log 版本链：当前→undo1→undo2...
- Read View：记录活跃事务列表，比 trx_id 判断版本可见性

**快照读 vs 当前读：**

- 快照读(普通SELECT)→MVCC版本，不加锁
- 当前读(UPDATE/DELETE/SELECT FOR UPDATE)→最新版本+加锁，Next-Key Lock防幻读

**MVCC 范围：** 只在 RC 和 RR 工作，RU 不适用，Serializable 不需要

**一句话兜底：** MVCC=隐藏列+undo链+Read View; RC每SQL新View, RR事务一次View; 快照读不加锁, 当前读Next-Key Lock。




### MVCC 多版本并发控制（复习 c-04-03）

> 版本号=事务ID；trx_id+roll_ptr版本链；Read View判断可见性。INSERT不产生旧版本(无roll_ptr)；DELETE标记删除，purge线程最终清理。MVCC只适用RC和RR。

**一句话兜底：** MVCC=trx_id+版本链+Read View；RC每SQL新View, RR整个事务复用一个View；INSERT无旧版本。




### MySQL 锁

**表锁 vs 行锁：** 表锁(MyISAM/低并发/无死锁) / 行锁(InnoDB独有/高并发/会死锁)

**S/X：** S共享(LOCK IN SHARE MODE，兼容S) / X排他(FOR UPDATE，不兼容任何) / IS/IX意向锁(表级，多粒度协调)

**InnoDB 三种行锁：**

- Record Lock：锁索引一条记录
- Gap Lock：锁索引间隙，防 INSERT
- Next-Key Lock：Record+Gap，RR 默认，防幻读（当前读）

**核心：** 锁索引非数据行，无索引→全表锁；死锁自动检测回滚小事务

**一句话兜底：** Record(一行)/Gap(间隙)/Next-Key(行+间隙防幻读)；S兼容S X不兼容；无索引=全表锁。




### B+ 树索引

**InnoDB vs MyISAM：** InnoDB(事务/行锁/聚簇) vs MyISAM(无事务/表锁/非聚簇)

**B+ 树优势：** IO 少(一页 16KB 树矮胖)/查询稳(都到叶)/范围快(叶子双向链表)

**聚簇 vs 非聚簇：**
- 聚簇：叶子存完整行数据(主键索引)，唯一
- 非聚簇：叶子存主键 ID→需回表查聚簇索引
- 覆盖索引：SELECT 列全在索引中→不回表

**最左前缀：** 联合索引(a,b,c)从最左列匹配，缺 a 不走，范围后后续列失效

**一句话兜底：** B+树叶存数据+链表；聚簇=完整行/非聚簇=主键ID→回表；最左前缀缺首不走范围后失效。




### 聚簇索引深入（复习 c-04-06）

> 聚簇=数据即索引(叶子完整行); 主键用自增ID防页分裂; 无主键InnoDB自动生成隐藏row_id(6字节)。一张表只有一个聚簇索引。

**一句话兜底：** 聚簇=完整行数据主键索引; 自增ID最佳; UUID随机插入页分裂; 无PK隐藏row_id兜底。




### 覆盖索引与回表

- 回表：非聚簇索引→主键ID→聚簇索引再查→两次索引树
- 覆盖索引：SELECT 列全在索引中→一次查完，`Using index`
- 避免 SELECT * 就是为了尽量覆盖索引减少回表

**一句话兜底：** 覆盖索引=列全在索引不回表(Using index)；回表=非聚簇→主键→聚簇再查。




### 索引失效场景

**失效场景：**
- LIKE '%x' 遮前缀 / 函数(YEAR/计算) / 隐式转换(varchar 传数字) / OR 无索引列 / != / IS NULL / NOT IN / 联合索引缺最左
- 区分度<0.1 MySQL 也可能不选索引

**优化：**
- LIKE 'x%' 可用，'%x%'考虑 ES / OR 改 UNION / 深分页子查询+覆盖索引先定位 ID 再回表 / CHAR 查询效率 > VARCHAR

**一句话兜底：** 失效九场景(LIKE%x/函数/隐式转换/OR/!=/ISNULL/NOTIN/缺最左)；OR改UNION；深分页子查询覆盖索引。




### EXPLAIN 执行计划

**关键四列：**
- type：system > const > eq_ref > ref > range > index > ALL（至少 range 争取 ref）
- key：实际索引，NULL=没走
- rows：扫描行数预估
- Extra：Using index(覆盖)✅ / Using filesort❌ / Using temporary❌

**表设计规范：** ≤500w 行 / ≤40 列 / ≤5 索引 / ≤200 表/库

**三范式：** 1NF 列不可分 / 2NF 完全依赖主键 / 3NF 无传递依赖（实际适度反范式）

**一句话兜底：** EXPLAIN 看 type/key/rows/Extra；type 避 ALL 争 range+；filesort+临时表需优化；表≤500w/5索引。




### JOIN 查询

- LEFT JOIN：左表全保留，右表无匹配=NULL
- RIGHT JOIN：右表全保留（用 LEFT 替代）
- INNER JOIN：只匹配行

**底层算法：** Index Nested-Loop(有索引最优) / Block Nested-Loop(无索引 Join Buffer) / Hash Join(8.0.18+)

**优化：** 小表驱动大表 / 被驱动表关联字段必建索引 / 类型一致

**一句话兜底：** LEFT全保留/INNER只匹配; 被驱动表必建索引走Index Nested-Loop; 8.0.18+ Hash Join更快。




### 数据库范式（复习 c-04-10）

> 1NF 列不可分 / 2NF 完全依赖PK / 3NF 无传递依赖。实际适度反范式减少 JOIN。列不可分、完全依赖PK、不间接依赖。




### 主从复制

**原理：** 主dump线程发binlog→从I/O收→Relay Log→SQL回放

**binlog 三种格式：** STATEMENT(小但不一致)/ROW(强一致推荐)/MIXED(折中)

**三种复制：** 异步(不等/可能丢)/半同步(等一个确认/推荐)/全同步(等所有/最慢)

**主从延迟应对：** 关键业务读主/缓存记录路由/非关键容忍

**一句话兜底：** binlog推荐ROW; 半同步推荐; 主从延迟=关键读主/缓存路由/非关键容忍。




### binlog / redo log / undo log

**三种日志：**

- binlog：Server 层，逻辑日志(SQL/行变化)，追加写，主从复制+时间点恢复
- redo log：InnoDB 层，物理日志(页+偏移)，循环写，崩溃恢复 WAL 保证持久性
- undo log：InnoDB 层，旧版本，事务回滚+MVCC 版本链，保证原子性

**WAL：** 先写 redo log 再异步刷脏页，崩溃后重放 redo 恢复

**二阶段提交：** redo prepare → 写 binlog → redo commit，保证主从一致

**一句话兜底：** binlog=Server逻辑追加主从; redo=InnoDB物理循环WAL持久；undo=回滚+MVCC原子；二阶段提交保一致。




### 数据一致性问题

**案例：** INSERT→SELECT→UPDATE 三步连做，主从延迟~50ms → 偶发查不到刚插入的数据

**诊断：** `SHOW SLAVE STATUS` → Seconds_Behind_Master

**解决：** 关键操作直连主库 / 分库降并发 / 缓存过渡 / 延迟查询

**原则：写后立即要读的必须读主库；刚 insert 的数据内存里就有别多查一次。**




### 集群架构

**演进：** Keepalived+VIP(脑裂)→MMM(多写冲突)→MHA(脚本/成熟)→MHA+Arksentinel(哨兵自动)

**脑裂：** 网络抖动→VIP漂移到多台→双写→数据冲突

**应对：** 原主 read_only + binlog server 补齐 + 分布式哨兵仲裁

**故障转移流程：** 确认真挂→read_only→数据补齐→提Slave为主→重定向→改连接

**一句话兜底：** Keepalived脑裂→MMM误判→MHA成熟；脑裂防双写read_only+补齐+哨兵仲裁。




### 故障转移与恢复（复习 c-04-16）

> 原主 read_only 防双写；binlog server 补齐再提 Slave；哨兵仲裁防误判。




### MySQL 综合考察题

**分库分表：** 水平(hash/range/id分担并发) / 垂直(冷热分离)；Sharding-JDBC(client) / Mycat(proxy)

**四个问题：** 跨库→ES / 倾斜→细粒度hash / 分布式事务→最终一致 / 深分页→游标(带上最大ID)

**数据迁移：** 双写不中断：老写+新写→程序补齐(updateTime)→一致后切换

**自增ID：** Snowflake推荐(41时间+10机器+12序列) / UUID无序不推荐主键 / Redis步长

**一句话兜底：** 分库分表hash/range路由；四问题ES+细粒度hash+最终一致+游标；迁移双写；ID用Snowflake。




### 分库分表深入（复习 c-04-18）

> 水平拆行/垂直拆列；hash均匀/range易扩；ES跨库查/细粒度防倾斜/最终一致/游标深分页；双写不中断迁移；Snowflake(41时间+10机器+12序列)。




### 自增 ID 方案（复习）

> Snowflake(41时间+10机器+12序列)推荐；UUID无序→页分裂不适合主键；Redis/步长/独立服务各有适用场景。




### 数据迁移（复习 c-04-18）

> 双写老+新 → 补齐历史(updateTime) → 校验 → 切流；不停机不锁表。




### 性能优化：深分页与查询优化

**深分页问题：** 分库后 LIMIT offset,N → 每库扫 offset 行归并 → N 倍数据 → 内存暴涨

**四种解法：**

- 全局视野：LIMIT 0, X+Y 服务层排序（翻页深性能降）
- 游标翻页：WHERE time>last_time LIMIT N（禁止跳页但恒定，推荐）
- 模糊翻页：每库 offset/N（不精确但恒定）
- 二次查询：先查→timemin→二次 between 定位（精确复杂）

**Controller：** 大结果集→JSON 解析内存翻倍→OOM。分页+精简 DTO+流式导出。

**一句话兜底：** 深分页=每库offset翻倍→游标翻页最优；查询必分页必精简字段；大导出走异步流式。




### 主从延时优化（复习 c-04-13/15）

> Seconds_Behind_Master诊断；分库/直连主库/缓存路由/延迟查询；写后即读必读主库。




### 深分页（复习 c-04-22）

> 分库后每库扫offset行归并→内存暴涨；游标翻页(WHERE id>last_id LIMIT N)最优。




### SQL 调优

**动态 SQL 陷阱：** `WHERE 1=1` + 参数全空 → `SELECT *` 全表 → 内存 OOM

**防范：** 强制分页 LIMIT / 必填参数校验 / 最大行数兜底 / MyBatis `<where>` 标签

**一句话兜底：** WHERE 1=1+空参=全表OOM；防=强制分页+参数校验+<where>+流式导出。




---

## plan-05-redis Redis篇

### Why Redis

**Redis 为什么快：** 纯内存(O(1))/单线程无锁/epoll多路复用/C语言/自定义VM

**Redis vs Memcached：** Redis 多结构+持久化+主从(单线程 2-4w TPS)；Memcached 仅String无持久化(多线程 20-40w TPS)

**缓存类型：** 本地(Guava/Caffeine, JVM内 ns级) / 分布式(Redis, 网络IO μs级) / 混合二级

**一句话兜底：** Redis=内存+单线程+多结构+持久化；Memcached=仅String+多线程快但无持久；本地快但多实例不一致。




### Memcached 对比（复习 c-05-01）

> Redis(多结构+持久化+主从) vs Memcached(仅KV+无持久+多线程高吞吐)；绝大多数场景Redis已取代。




### Tair（复习 c-05-01）

> 淘宝分布式KV; MDB(纯内存)/RDB(内存+持久化+复杂结构)/LDB(LevelDB/最高可靠)；中心化配置节点。




### Guava Cache

**定位：** JVM本地缓，ns级，多实例不共享

**核心 API：** maximumSize/expireAfterWrite/expireAfterAccess/refreshAfterWrite/recordStats

**五大优势：** 过期淘汰(时间+LRU) / 分段锁并发 / 防击穿(同key单线程回源) / 异步刷新 / 命中率统计

**vs Map：** Map无过期/淘汰/击穿防护/监控；Guava全内置。CacheLoader自动回源。

**一句话兜底：** 本地ns级缓存；CacheLoader防击穿；expireAfterWrite/expireAfterAccess控制过期；不跨实例共享。




### EVCache 与 etcd

**EVCache：** Netflix 基于 Memcached 全球分布式缓存，Ephemeral(短暂)+Volatile(易失)，3000万 QPS/1-5ms/99%命中，分钟级扩容

**etcd：** CP+Raft，Go版ZK(K8s标配)，服务发现/配置中心/分布式协调，gRPC/HTTP API

**一句话兜底：** EVCache=短暂易失全球缓存；etcd=CP+Raft强一致KV(K8s标配)。




### Redis 五大数据类型

- **String**(SDS)：缓存/计数器/分布式锁，SET/GET/INCR/SETNX
- **List**(QuickList)：队列/时间线，LPUSH/RPUSH/BRPOP(阻塞消费)
- **Set**(IntSet/HT)：去重/好友交集/抽奖，SADD/SINTER/SUNION/SDIFF
- **ZSet**(Ziplist/SkipList+Dict)：排行榜/延时队列，ZADD/ZRANGE/ZRANK(logN)
- **Hash**(Ziplist/HT)：对象存储/购物车，HSET/HGET/HINCRBY，小对象省内存

**一句话兜底：** String万能/List队列/Set去重/ZSet排行榜/Hash对象；底层SDS/QuickList/SkipList/Ziplist。




### Redis 底层数据结构

**SDS：** len+free+buf，O(1)长度，二进制安全，预分配扩容

**Dict：** ht[0]+ht[1]，渐进式 rehash（每次操作搬一点，不阻塞）

**SkipList：** 多层有序链表 O(logN)，ZSet=SkipList(score排序)+Dict(O1查分)，比红黑树简单/范围快/易序列化

**Ziplist：** 连续内存紧凑存储（小数据省内存），元素多升级为 QuickList/SkipList/HT

**QuickList：** 双链表+ziplist 节点（List 底层）

**一句话兜底：** SDS(O1/安全)→Dict(渐进rehash)→SkipList(跳表/ZSet)→Ziplist(紧凑)→QuickList(List)。




### 渐进式 Rehash（复习 c-05-07）

> ht[0]+ht[1]双表；每操作搬一桶；新写ht[1]；防一次性阻塞。




### Ziplist 与 IntSet（复习 c-05-07）

> Ziplist(连续内存紧凑/连锁更新/7.0→Listpack替); IntSet(有序整数数组/二分查找/纯整数Set底层)




### ZSet 跳表实现（复习 c-05-07）

> ZSet=SkipList(score排序OlogN)+Dict(member→score O1); 跳表优于红黑树(简单/范围查快/易序列化)。




### RDB 与 AOF 持久化

**RDB：** 全量快照，fork 子进程写 dump.rdb，恢复快，可能丢快照间数据

**AOF：** 每条写命令追加，fsync(always/everysec推荐/no)，安全但文件大恢复慢。AOF rewrite 压缩无效命令

**混合持久化(4.0+)：** RDB 全量+AOF 增量=恢复快且安全

**生产推荐：** everysec + 混合持久化(`aof-use-rdb-preamble yes`)

**一句话兜底：** RDB快照(快/可能丢)→AOF日志(安全/大/慢)→混合(全量+增量)；生产 everysec+混合。




### Redis 事务

**四原语：** MULTI(开启)/EXEC(执行)/DISCARD(中止)/WATCH(CAS乐观锁)

**关键：** 不支持回滚！语法错全不执行，运行错部分执行。只保证一致性和隔离性。

**WATCH CAS：** 监控key→被改则事务不执行→秒杀场景防超卖

**一句话兜底：** MULTI/EXEC/WATCH四原语；无回滚(运行错部分执行)；WATCH CAS乐观锁秒杀。




### 内存淘汰与过期策略

**过期删除：** 惰性(访问时)+定期扫描(100ms随机20个, >25%过期继续)

**内存淘汰八策略：**
- noeviction：不淘汰拒绝写(持久化)
- allkeys-lru：全局LRU(缓存推荐)
- allkeys-lfu：全局LFU频率(4.0+)
- volatile-*：仅TTL key(volatile-lru/lfu/random/ttl)

**近似 LRU：** 随机采样 5 个淘汰最久未访问，非精确 LRU

**一句话兜底：** 过期=惰性+定期扫描；淘汰=allkeys-lru(缓存推荐)/noeviction(持久化)；近似LRU随机5个。




### 缓存读写模式

**Cache Aside(99%用)：** 读miss查DB回填；写先更DB→删缓存(不更新！)

**为什么删不更新：** 并发写乱序→更新缓存可能被过时数据覆盖

**陷阱：** 先删缓存再更DB→中间读到旧值回填→不一致。必须先更DB再删。

**Read/Write Through：** 缓存层封装DB / **Write Behind：** 异步刷DB延迟低但可丢数据

**一句话兜底：** Cache Aside(先更DB后删缓存)；禁止先删后更DB；Write Behind异步低延迟但可丢数据。




### 多级缓存与一致性

**多级缓存：** 浏览器→CDN→本地(Caffeine)→Redis→DB。不用本地磁盘(iowait低)。

**一致性：** 延迟双删(删→更DB→sleep→再删)偷解决旧值回填；更可靠=Canal监听binlog+MQ异步删。

**一句话兜底：** 四级缓存(浏览器→CDN→本地→Redis)；延迟双删；Canal+MQ异步删更可靠。




### 缓存雪崩 / 穿透 / 击穿

**雪崩：** 大量key同时过期/Redis宕→DB炸。解决：TTL+随机/高可用/本地兜底

**穿透：** 查不存在的key→每次落DB。解决：缓存空值/布隆过滤器/参数校验

**击穿：** 一个热点key过期→海量并发同查一条→DB炸。解决：互斥锁/热点永不过期/预热

**一句话兜底：** 雪崩(大量key同时)→TTL随机+高可用；穿透(不存在)→缓存null+布隆；击穿(一个热点过期)→互斥+永不过期。




### 数据不一致

> rehash 漂移致脏；MQ 重试删缓存 / 缩短 TTL / 分层不漂移

### 数据并发竞争

> 热点过期海量并发→互斥锁/多备份分散/默认值快速返回

### HotKey 问题

> 发现：预估/Spark实时/Hadoop离线；解决：key后缀拆分多slot/主从扩容/本地前置缓存

### BigKey 问题

> 大 value 阻塞单线程；拆小 key/Hash 分段/长 TTL 避免频繁重建




### 数据分区

- Hash取余(均匀/节点变全迁移)→一致性Hash(环/影响邻近/虚拟节点均匀)→RedisCluster hash槽(16384slot, crc16%16384)

### 主从模式

- 一主多从读写分离，PSYNC全量(RDB+缓冲)+增量同步，Master挂需人工切换，适合初期

### 哨兵模式

- Sentinel监控+自动Failover: PING→SDOWN→≥quorum→ODOWN→Raft选Leader→选新Master(优先级>偏移量>runid)

### 集群模式

- 16384slot分布多Master，CRC16(key)定位slot，内置故障转移，写多/数据量大场景

**架构选型：** 初期主从→读多哨兵→写多/大数据集群

**一句话兜底：** 分区hash取余→一致性hash→hash槽；高可用主从→哨兵(Raft选Leader)→集群(slot分片)。




### 数据分区（重讲）

**Hash 取余：** hash(key)%N，简单均匀但节点增减→全量迁移，不能动态扩缩

**一致性 Hash：** 节点/key 映射到环，key 顺时针找节点。增减只影响相邻弧段（非全量）。虚拟节点解倾斜

**Redis Cluster hash 槽：** 16384 slot，CRC16(key)%16384 定槽位，按槽迁移。16384=心跳 2KB 刚好

**一句话兜底：** 取余(全移)→一致性hash(环/虚拟节点/邻近移)→Cluster槽(16384/按槽移/灵活)。




### 主从模式（详细）

**架构：** 一主多从，Master 写 Slave 读，读写分离

**复制流程：**

- 首次：Slave PSYNC→Master BGSAVE(RDB)+积压buffer→发RDB→Slave清空加载→buff命令补发→一致
- 运行中：命令传播，Master 每条写异步发给所有 Slave
- 断连重连：offset 在 backlog 内→部分复制（只发缺失）；超出→全量
- Redis 4.0 PSYNC2：Slave 重启后也可部分复制（repid+offset 写入 RDB）

**backlog：** 环状缓冲区，默认 1MB，断线期间的写命令暂存。太小→断连稍久触发全量。

**缺点：** Master 单点不能自动切换、写压力集中、异步复制有延迟、Slave 多 Master 同步压力大

**适用：** 读多写少、业务初期、可接受人工切换




### 哨兵模式（详细）

**架构：** 至少 3 个 Sentinel 进程，监控主从，自动故障检测 + 转移

**SDOWN vs ODOWN：**

- SDOWN：PING 无响应 ≥down-after-milliseconds →单 Sentinel 认为下线（主观，可能误判）
- ODOWN：SDOWN 后广播询问 →≥quorum 确认 →客观下线（只对 Master）

**Raft 选 Leader：** 随机超时发起 + 一 term 一票 + 多数派 →Leader 执行故障转移

**选新 Master：** ①过滤不健康 ②slave-priority 高 ③offset 大(数据最新) ④runid 小(最稳定)

**故障转移：** 升 Slave 为 Master →其余重定向新 Master →旧 Master 恢复后降级（防脑裂）

**一句话兜底：** SDOWN→广播→quorum→ODOWN→Raft选Leader→选最佳Slave→升主→旧主降级。




### 集群模式（详细）

**为什么：** 哨兵只解决高可用，不解决容量和写扩展。集群 = 分片 + 高可用

**Hash 槽：** 16384 slot，CRC16(key)%16384，每 Master 负责部分 slot

**路由：** MOVED(永久重定向/下次直接找新节点) vs ASK(临时/槽迁移中/下次还问原节点)

**Gossip：** 节点定时 PING/PONG 交换状态 →PFAIL→多数确认→FAIL→Slave 竞选→新 Master 接管 slot

**vs 哨兵：** 哨兵(单 Master 全量/仅高可用/需独立 Sentinel)/集群(多 Master 分片/高可用+扩展/内置 Gossip/客户端需智能)

**一句话兜底：** 16384槽分片；Gossip通信；MOVED永久/ASK临时；集群=分片+高可用，哨兵仅高可用。




### 分布式锁（详细）

**为什么需要：** 分布式多 JVM，本地锁管不了别的实例，需要共享锁点

**演进：**

- SETNX(无过期→死锁/del 误删) → SET NX EX(原子加锁+过期防死锁+唯一标识) → Lua 校验删除(防误删)
- Redisson 看门狗：每 10 秒续期 30 秒，业务完锁不过期
- RedLock：N/2+1 个独立 Redis 多数派确认，防主从切换丢锁

**乐观锁：** WATCH+MULTI/EXEC CAS 乐观锁，适合秒杀

**一句话兜底：** SET NX EX(原子加锁)→Lua脚本(原子解锁)→看门狗(续期)→RedLock(多数派防丢锁)。




### Redis 实战

**分布式锁局限：** 客户端阻塞/时钟漂移/主从切换丢锁/RedLock 也不完美

**Redis vs ZK 分布式锁：** Redis(自旋CAS/非公平/极高性能/可能丢锁)→ZK(监听通知/公平/慢/CP不丢锁)
选型：性能优先幂等→Redis；金融强一致→ZK/etcd

**心跳：** REPLCONF ACK 每秒检测连接+辅助 min-slaves 安全写+检测命令丢失重传

**优化五维度：**
- 读写：不用 KEYS→SCAN 分页 / 大 value 拆小 key
- Key：只缓存热数据，冷数据查 DB
- QPS：<10w 单池 / >100w 多级（本地→Redis→DB）
- 命中率：核心业务 >99%，预留容量

**一句话兜底：** Redis锁(快/可能丢)→ZK(CP/可靠); min-slaves安全写; 优化=SCAN+小KV+热数据+多级+高命中。




---

## plan-06-kafka Kafka篇

### Why Kafka

**MQ 三大作用：** 异步(主流程快返)/削峰填谷(流量排队匀速消费)/解耦(生产消费不直连)

**MQ 选型：** RabbitMQ(路由灵活/几万QPS)→RocketMQ(电商毫秒/几十万QPS/Java)→Kafka(海量吞吐/2000万QPS/回放)

**Kafka 核心概念：**

- Topic(主题/消息分类)→Partition(有序队列/水平扩展/单分区内有序)→Replica(副本/Leader读写)
- Producer(分区策略:指定/keyhash/轮询)→Consumer(offset记进度)
- Consumer Group：组内一分区只给一消费者(防重复)，挂掉 Rebalance
- **关键差异：** 消息消费后不删除(基于时间/大小保留)，支持回放

**一句话兜底：** MQ=异步+削峰+解耦；Kafka=海量吞吐+分区+消费者组+消息持久化回放。




### MQ 对比深入（复习 c-06-01）

| | RabbitMQ | RocketMQ | Kafka |
|------|----------|----------|-------|
| 语言 | Erlang | Java | Java/Scala |
| 吞吐 | 几万~几十万 | 几十万 | 几十万~2000万 |
| 延迟 | 毫秒 | 毫秒 | 较高(批处理) |
| 杀手锏 | AMQP路由 | 事务消息 | 持久化回放 |
| 选型 | 路由灵活 | 电商交易 | 大数据流 |

**一句话兜底：** 响应→RocketMQ; 路由→RabbitMQ; 海量回放→Kafka。




### Kafka 概念（复习 c-06-01）

> Topic→Partition(并行/扩展)→ConsumerGroup(负载均衡)→Replica(高可用)；单分区内有序跨分区无序；offset 记录消费者进度；消息不删支持回放。




### Kafka 架构与存储原理

**存储：** Partition=多个 LogSegment(.log+.index+.timeindex)，文件名以首消息 offset 命名，顺序追加写

**三大性能基石：**

- 顺序写：磁盘追加 ~0.03ms(随机 ~10ms)
- 零拷贝：sendfile 系统调用，磁盘→内核→网卡 DMA 直传，CPU 不参与
- 页缓存：OS 管理，不占 JVM 堆，GC 不影响

**ISR：** Leader+同步 Follower，落后 >10s 踢出，追上再加回

**LEO vs HW：** LEO=下条写入位置 / HW=ISR 全确认的最大 offset(消费者可见边界)

**一句话兜底：** LogSegment 顺序追加；零拷贝+页缓存=极高吞吐；ISR 保可靠性；HW 定消费边界。




### How Kafka（复习 c-06-04）

> 三大基石(顺序写/零拷贝/页缓存)；场景：日志收集/消息系统/用户行为跟踪。




### 生产消费流程

**Producer 流程：** 拦截器→序列化→分区器→缓冲区→Sender 线程异步批量(触发:batch.size/linger.ms)

**acks：** 0(不等/最快/丢)/1(Leader确认/默认)/all(ISR全确认/最安全)

**重试乱序：** retries 可能导致乱序→`max.in.flight=1`或幂等 enable.idempotence=true

**Leader 选举：** ISR 中选新 Leader；ISR 全挂→unclean.leader.election 决定是否从 OSR 选

**Rebalance：** 消费者增/减、Topic 变、分区数变 →Coordinator 触发→Group Leader 制定方案(RangeAssignor)→同步全组→全组暂停恢复

**一句话兜底：** Producer(拦截→序列→分区→缓冲→Sender); acks(0/1/all); Rebalance 全组停消费; ISR 选主+Follower FETCH 同步。




### ISR 深入（复习 c-06-04/06）

> ISR=Leader+同步Follower; 落后>10s踢出; ISR中选Leader; ISR全挂→unclean.leader.election; HW=ISR最小LEO




### 消息确认机制（复习 c-06-06）

> acks: 0(不等/最快/丢) → 1(Leader确认/默认) → all(ISR全确认/最安全)；支付订单用all，日志用0或1。




### 高可用性（复习）

> Partition多副本(N+1容N台宕)+ISR自动选Leader+acks=all(ISR全确认不丢)


### Rebalance（详细）

**为什么可怕：** 全组暂停消费→积压雪崩。GC>session.timeout→误判假死→循环 Rebalance

**关键参数：** session.timeout.ms(心跳超时/默认45s)/max.poll.interval.ms(poll间隔/默认300s)/max.poll.records(拉取量)

**优化：** 加大超时/减小拉取/单独线程处理/静态Group(2.3+)/Sticky分配策略

**分配策略：** Range(默认/字典序) → RoundRobin(轮询) → Sticky(轮询+Rebalance尽量保持/推荐)

**一句话兜底：** Rebalance=全组停消费; GC超时→循环雪崩; Sticky+静态Group最小化。




### 消息不丢失不重复

**三种语义：** At-most(丢不重)/At-least(不丢可重/Kafka默认)/Exactly-once(不丢不重)

**Producer 幂等：** PID+SequenceNumber→Broker 匹配去重(enable.idempotence=true)，仅单 Session 有效

**Consumer 去重：** 唯一键 Redis/DB 去重 + 业务幂等 UPDATE SET

**防丢三道防线：**
- Producer: acks=all+retries=MAX+幂等
- Broker: replication≥3+minISR≥2+unclean=false
- Consumer: 先处理后提交 offset（不是反过来！）

**Leader Epoch：** 替换 HW，每次 Leader 切换 epoch+1，恢复时精准截断，避免多截/少截

**一句话兜底：** PID+SN(幂等)→唯一键(去重)→三道防线(acks=all/副本≥3/先处理再提交)→LeaderEpoch精准截断。




### 消息积压处理

**紧急处理：** 增加分区→扩容消费者→分发程序拆到N个临时Topic→清完恢复

**预防：** 提高并行度(分区+消费者)/批量消费减少IO/降级跳过非关键消息

### 消息顺序性

**Kafka顺序：** 单Partition内严格有序，跨Partition无序

**全局有序：** 1Topic+1Partition+单线程(低吞吐)

**局部有序(推荐)：** key hash路由同Partition+Consumer端按key分内存队列→串行线程，既保证单key有序又高吞吐

**一句话兜底：** 积压=增分区→加消费→拆临时Topic; 顺序=key hash同分区+按key分队列串行消费。




---

## plan-07-spring Spring篇

### IoC 与 DI

**IoC(控制反转)：** 对象创建控制权从程序→容器，不 new 只声明依赖，容器 = Map<Bean名, 实例>

**DI(依赖注入)：** 字段@Autowired(最常用)/构造器(final不可变/测试友好/推荐)/Setter(可选依赖)

**refresh 12 步：** ②解析 BeanDefinition→⑥注册 BeanPostProcessor(AOP入口)→⑪初始化所有单例 Bean(核心)

**一句话兜底：** IoC=控制权反转; DI=构造器注入推荐; refresh=12步启动Spring容器。




### AOP 动态代理

**AOP：** 切面(Pointcut+Advice)集中管理横切关注点(日志/权限/事务)

**JDK Proxy：** 反射+实现接口(必须有接口), InvocationHandler.invoke 拦截

**CGLib：** 字节码(ASM)+子类继承(不需要接口/不能final), 创建慢调用快, SpringBoot2x 默认

**通知：** @Before/@After/@AfterReturning/@AfterThrowing/@Around(环绕最灵活)

**陷阱：** @Transactional AOP 代理, this 内部调用不走代理不生效

**一句话兜底：** AOP=动态代理集中横切逻辑; JDK(接口反射)→CGLib(子类字节码/SpringBoot2x默认); this不触发代理。


### Bean 生命周期 — refresh() 12 步

**过程：**

- refresh() 被 synchronized 锁住，保证只执行一次，分四阶段 12 步
- 阶段 A 准备与加载（1-3）：
  - 第 1 步 `prepareRefresh()`：记录启动时间、校验必填属性、初始化 earlyApplicationEvents
  - 第 2 步 `obtainFreshBeanFactory()`：销毁旧 BF → 创建 DefaultListableBeanFactory → 解析 XML/注解生成 BeanDefinition
  - 第 3 步 `prepareBeanFactory(beanFactory)`：设 ClassLoader、SPEL 解析器、注册 ApplicationContextAwareProcessor、忽略 Aware 接口自动装配
- 阶段 B 后置处理与扩展注册（4-6）：
  - 第 4 步 `postProcessBeanFactory(beanFactory)`：子类扩展点（如 Web 容器注册 request/session Scope）
  - 第 5 步 `invokeBeanFactoryPostProcessors(beanFactory)`：按 PriorityOrdered → Ordered → 普通 顺序执行 BD RPP（含 ConfigurationClassPostProcessor 解析 @Configuration）再执行 BF PP
  - 第 6 步 `registerBeanPostProcessors(beanFactory)`：实例化所有 BPP 并注册到 beanPostProcessors 列表，此时不执行
- 阶段 C 基础设施初始化（7-10）：
  - 第 7 步 `initMessageSource()`：国际化
  - 第 8 步 `initApplicationEventMulticaster()`：创建事件广播器（默认同步 SimpleApplicationEventMulticaster）
  - 第 9 步 `onRefresh()`：子类扩展，Spring Boot 在此启动内嵌 Web 容器
  - 第 10 步 `registerListeners()`：注册监听器，派发 earlyApplicationEvents
- 阶段 D 实例化与收尾（11-12）：
  - 第 11 步 `finishBeanFactoryInitialization(beanFactory)`：freeze 配置 → preInstantiateSingletons 实例化所有非懒加载单例（最耗时）
  - 第 12 步 `finishRefresh()`：初始化 LifecycleProcessor → 发布 ContextRefreshedEvent → 注册 LiveBeansView

**原理：**

- step 5 的 ConfigurationClassPostProcessor 必须排第一（PriorityOrdered），因为它把 @Bean 方法解析为新 BeanDefinition
- 后续 BF PP 才能看到这些新增的 BeanDefinition
- step 6 注册的 BPP 在后续每个 Bean 初始化时逐个拦截
- step 11 freezeConfiguration() 冻结 BeanDefinition，防单例创建后定义被修改

**坑点：**

- ① 第 5 步实例化了所有 BF PP 和 BPP（通过 getBean），但这些 Processor 自身不走后续 BPP 拦截——因为 BPP 在第 6 步才注册
- ② 默认事件广播器 SimpleApplicationEventMulticaster 是同步的——publishEvent 在所有监听器的 onApplicationEvent 依次执行完才返回
- ③ Spring Boot 内嵌 Web 容器在第 9 步 onRefresh 创建，在所有单例 Bean 初始化之前（第 11 步之前）

**一句话兜底：** refresh() 12 步关键节点：step2 加载定义、step5 解析 @Configuration、step6 注册 BPP、step9 启动 Web 容器、step11 创建所有单例。

### Bean 生命周期 — 完整流程与扩展点

**过程：**

- 阶段一 Instantiation（实例化）：
  - 反射调用构造器创建对象，此时属性全是默认值
  - InstantiationAwareBeanPostProcessor.postProcessBeforeInstantiation 若返回非 null，直接跳过后续实例化
  - 构造器选择：单构造器直接用；多构造器默认无参，@Autowired 标注的优先
- 阶段二 Populate（属性赋值）：
  - InstantiationAwareBeanPostProcessor.postProcessAfterInstantiation 返回 false 可阻止填充
  - AutowiredAnnotationBeanPostProcessor 在此处理 @Autowired 和 @Value
  - 再注入 BeanDefinition.propertyValues（XML property）
- 阶段三 Initialization（初始化）：
  - step 3-5：Aware 回调（BeanNameAware → BeanFactoryAware → ApplicationContextAware）——由 ApplicationContextAwareProcessor 在 postProcessBeforeInitialization 中驱动
  - step 6：BPP.postProcessBeforeInitialization() ——可包装 Bean、修改属性
  - step 7：@PostConstruct → InitializingBean.afterPropertiesSet()
  - step 8：init-method（XML 或 @Bean(initMethod)）
  - step 9：BPP.postProcessAfterInitialization() ——AbstractAutoProxyCreator 在此创建 AOP 代理
- 阶段四 Destruction（销毁）：
  - 容器关闭时 @PreDestroy → DisposableBean.destroy() → destroy-method
  - Lifecycle Bean 的 stop() 在 destroy() 之前调用

**原理：**

- Aware 从窄到宽（BeanName → BeanFactory → ApplicationContext），后一步包含前一步能力
- @PostConstruct 本质上是 CommonAnnotationBeanPostProcessor 在 postProcessBeforeInitialization 中反射调用
- AOP 代理在 step 9 创建，因为 init 方法可能涉及 this.method() 调用，代理需在 init 之后才能正确拦截
- 哪一步 BPP 返回了新对象，后续流程就基于新对象，最终存入单例池的是 postProcessAfterInitialization 的返回值

**坑点：**

- ① Prototype Bean 不执行任何销毁回调（destroy-method、@PreDestroy 均不生效），Spring 创建完即放手
- ② init-method 或 afterPropertiesSet 抛异常 → 后续 BPP 不走 → Bean 不进单例池 → disposableBeans 无记录 → @PreDestroy 不执行
- ③ 构造器循环依赖无法解决（Spring 需要实例才能放入三级缓存），只有 setter/field 注入可通过三级缓存解决

**扩展点分类：**

- 影响多个 Bean：BeanFactoryPostProcessor（改定义）、BeanPostProcessor（改行为）、InstantiationAwareBeanPostProcessor（替换实例化）
- 影响单个 Bean：Aware（拿资源）、InitializingBean/@PostConstruct（自定义初始化）、DisposableBean/@PreDestroy（清理）

**一句话兜底：** Bean 生命周期=实例化→属性赋值→Aware→BPP.before→init→BPP.after(AOP代理)→销毁; 扩展点分多Bean(BFPP/BPP)和单Bean(Aware/init/destroy)。

### 循环依赖 — 三种场景与解决能力

**场景分类：**

- ① Prototype 循环依赖：❌ 无法解决。每次 getBean 创建新实例且不缓存，无半成品机制
- ② 构造器注入循环依赖：❌ 无法解决。实例化与依赖注入绑定，new A(b) 需要 B 实例但 B 还没创建
- ③ Field/setter 注入循环依赖：✅ 三级缓存解决。new A() 不需要参数就能拿到半成品，提前暴露

**@Lazy 方案：**

- 原理：注入了代理对象而非真实 B，第一次调方法时才去容器获取真正的 B
- 局限：构造器注入不能用（类型不匹配），引入代理开销，本质是绕开而非解决
- 工程建议：优先重构消除循环依赖（引入中间类），其次 Field/setter + 三级缓存，最后 @Lazy 兜底

**一句话兜底：** 构造器循环无法解(实例化绑依赖无半成品)，Field/setter 可解(三级缓存提前暴露)；@Lazy 是绕开不是解决。

### 循环依赖 — 三级缓存机制详解

**三级缓存定义：**

| 缓存 | 变量名 | 存什么 | 放时机 | 取时机 |
|------|--------|--------|--------|--------|
| 一级 | `singletonObjects` | 成品 Bean | Bean 创建全部完成 | 正常 getBean 返回 |
| 二级 | `earlySingletonObjects` | 半成品早期引用 | 从三级升级（首次 get） | 其他 Bean 注入时 |
| 三级 | `singletonFactories` | ObjectFactory 工厂 | 实例化后、属性填充前 | 发现循环依赖时调 getObject() |

**为什么必须三级：**

- 没二级：每次取用都调 factory.getObject() → 可能生成不同的 AOP 代理 → 同一个 Bean 出现多个版本
- 没三级：实例化后直接把对象放二级 → 没有机会在"被使用前"做 AOP 代理包装
- 三级缓存解决了"半成品 + 代理唯一 + 延迟包装"的三角问题

**关键辅助结构：**

- `singletonsCurrentlyInCreation`：Set&lt;String&gt;，记录正在创建中的 Bean 名称
- 作用①：检测循环依赖（getBean 时发现目标 Bean 在此 Set 中，说明循环）
- 作用②：控制三级缓存只在 Bean 创建的窗口期可用

**A↔B 完整推演流程：**

```
1. getBean("A") → 一级无 → 标记 singletonsCurrentlyInCreation += "A"
2. 实例化 A：new A() → 半成品 A₀
3. addSingletonFactory("A", lambda) → 三级 += {"A": factory}
4. populateBean("A") → 发现 @Autowired B → getBean("B")
5. getBean("B") → 一级无 → 标记 singletonsCurrentlyInCreation += "B"
6. 实例化 B → addSingletonFactory("B", lambda) → 三级 += {"B": factory}
7. populateBean("B") → 发现 @Autowired A → getBean("A")
8. getSingleton("A"):
   - 查一级 → 无
   - A 在 singletonsCurrentlyInCreation 中？是！触发循环依赖处理
   - 查二级 → 无（还没人取过 A 的早期引用）
   - 查三级 → 拿到 factory
   - factory.getObject() → getEarlyBeanReference → A*（可能是原始或代理）
   - 二级 += {"A": A*}，三级 -= "A"
   - 返回 A*
9. B.a = A* 注入完成 → B 初始化 → B 成品 → 一级 += {"B": B}
   → singletonsCurrentlyInCreation -= "B"
10. 回到 A：A.b = B成品（从一级直接拿）→ A 初始化 → A 成品
11. addSingleton("A", A成品) → 一级 += {"A": A成品}，二级 -= "A"
    → singletonsCurrentlyInCreation -= "A"
```

**AOP 代理去重机制：**

- `getEarlyBeanReference` 创建早期代理后记录到 `earlyProxyReferences`
- `postProcessAfterInitialization` 检查该集合：已有早期代理 → 原样返回，不重复创建
- 保证 B 持有的早期代理 A* 和最终一级缓存的 A 成品是同一个对象

**一句话兜底：** 一级存成品、二级存半成品引用、三级存工厂lambda；二级保代理唯一，三级保可包装；singletonsCurrentlyInCreation 是循环检测开关。

### Spring 注解 — Bean 声明与注入 + @Autowired 原理

**四大声明注解：**

- `@Component`：通用组件注解
- `@Service`：业务逻辑层，本质是 `@Component` 的特化别名
- `@Repository`：数据访问层，额外享有持久层异常翻译（SQLException → DataAccessException）
- `@Controller`：展现层，Spring MVC 使用

**注入注解对比：**

| 注解 | 来源 | 默认策略 | 配合 | 状态 |
|------|------|---------|------|------|
| `@Autowired` | Spring | 按类型 byType | `@Qualifier("name")` 切按名称 | 主力 |
| `@Resource` | JDK | 按名称 byName | `@Resource(name="x")` | JDK 11 弃用，17 移除 |

**@Autowired 实现原理（AutowiredAnnotationBeanPostProcessor）：**

- 实现接口：`InstantiationAwareBeanPostProcessor` + `MergedBeanDefinitionPostProcessor` + `BeanFactoryAware`
- 步骤 1：`postProcessMergedBeanDefinition` — 反射扫描 Field/Method，找 @Autowired/@Value 注入点，封装为 InjectionMetadata **缓存**
- 步骤 2：`postProcessProperties` — 从缓存取注入点，调 `resolveDependency()` → `getBean()` → 反射注入
- 分两步的原因：步骤 1 时 Bean 还没实例化，只能做元数据缓存；步骤 2 时 Bean 已实例化，才能反射设值

**@SpringBootApplication 三合一：**

- `@SpringBootConfiguration`：底层即 `@Configuration`，支持 @Bean 方法
- `@EnableAutoConfiguration`：自动配置核心（下一节详述）
- `@ComponentScan`：默认扫描当前类所在包及子包

**一句话兜底：** @Component/@Service/@Repository/@Controller 底层等价，Repository 额外翻译持久层异常；@Autowired=按类型+AutowiredAnnotationBeanPostProcessor 分两步(缓存注入点/反射注入)。

### Spring 注解 — 自动配置 + SpringMVC + @Transactional

**@EnableAutoConfiguration 自动配置原理：**

- 步骤 1：`@Import(AutoConfigurationImportSelector)` → `selectImports()` 扫描所有 jar 的 `META-INF/spring.factories`
- 步骤 2：提取 `EnableAutoConfiguration` key 对应的 100+ 候选自动配置类
- 步骤 3：按 `@ConditionalOnClass` / `@ConditionalOnBean` / `@ConditionalOnProperty` 等过滤
- 步骤 4：返回符合条件的配置类 → 注册为 BeanDefinition
- Spring Boot 3.x 改为 `META-INF/spring/...AutoConfiguration.imports` 文件，思路不变

**SpringMVC 请求流程（8步）：**

- ① 请求到达 `DispatcherServlet`（前端控制器）
- ② `HandlerMapping` 根据 URL 找 Handler（Controller 方法）
- ③ `HandlerAdapter` 适配并执行 Handler
- ④ Controller 返回 `ModelAndView`（Model 是数据，View 是逻辑视图名）
- ⑤ `ViewResolver` 解析逻辑视图名为真正 View 对象
- ⑥ `View.render()` 渲染 HTML
- ⑦ `@ResponseBody` 场景：`HttpMessageConverter` 序列化为 JSON 直接写 response
- ⑧ 返回响应给客户端
- 核心组件：DispatcherServlet、HandlerMapping、HandlerAdapter、HandlerInterceptor、ViewResolver、HttpMessageConverter

**@Transactional 事务传播属性（7种）：**

| 传播属性 | 有事务时 | 无事务时 | 使用场景 |
|---------|---------|---------|---------|
| `REQUIRED`（默认） | 加入当前 | 新建 | 99% 场景 |
| `REQUIRES_NEW` | 挂起当前，新建独立 | 新建 | 日志、审计等必须独立提交 |
| `NESTED` | 创建 Savepoint | 等价 REQUIRED | 批量处理部分失败可回滚 |
| `MANDATORY` | 加入当前 | 抛异常 | 强依赖事务上下文 |
| `NEVER` | 抛异常 | 非事务执行 | 不允许有事务 |
| `NOT_SUPPORTED` | 挂起，非事务执行 | 非事务执行 | 调远程 API 不占连接 |
| `SUPPORTS` | 加入当前 | 非事务执行 | 可有可无的查询 |

**REQUIRES_NEW vs NESTED 核心区别：**

- REQUIRES_NEW：独立事务(独立连接)，外层回滚不影响内层
- NESTED：Savepoint 嵌套(复用连接)，外层回滚会连带内层；内层回滚只回滚到 Savepoint，不影响外层

**一句话兜底：** 自动配置=spring.factories候选+@Conditional过滤；SpringMVC=DispatcherServlet→HandlerMapping→HandlerAdapter→ViewResolver/HttpMessageConverter；事务传播默认REQUIRED，REQUIRES_NEW完全独立，NESTED外层滚内层滚。

### Spring 源码 — 设计模式视角

**1. 单例模式：**

- Spring 是容器级单例：beanName → singletonObjects(ConcurrentHashMap) 唯一映射
- 和 GoF 单例区别：不限制构造器、可注册同名类不同 beanName 多实例、容器启停管生命周期
- 本质是注册式单例，不是访问控制式单例

**2. 工厂模式：**

- `BeanFactory` 接口层级：HierarchicalBeanFactory → ListableBeanFactory → DefaultListableBeanFactory
- `ApplicationContext` 高级容器：预加载、国际化、事件发布、AOP
- `FactoryBean<T>`：自定义 Bean 创建逻辑，`getObject()` 返回的对象进容器，MyBatis MapperFactoryBean 是典型案例

**3. 代理模式：**

- JDK 动态代理：`JdkDynamicAopProxy` + `InvocationHandler`，必须有接口
- CGLib 代理：`CglibAopProxy` + `Enhancer`，继承目标类，不能是 final
- Spring Boot 2.x 默认 CGLib（`proxy-target-class=true`）

**4. 观察者模式（事件驱动）：**

- 三组件：`ApplicationEvent`（事件）、`ApplicationListener`（监听器）、`ApplicationEventMulticaster`（广播器）
- 内置事件：ContextRefreshedEvent（容器就绪）、ContextClosedEvent（容器关闭）
- 自定义：`publishEvent(new XxxEvent(source, data))` + `@EventListener` 监听
- 默认同步执行，异步需配 `TaskExecutor` 或加 `@Async`

**5. 适配器模式：**

- AOP 场景：`AdvisorAdapter` 将 MethodBeforeAdvice/AfterReturningAdvice/ThrowsAdvice 统一适配为 `MethodInterceptor`
- SpringMVC 场景：`HandlerAdapter` 将 @Controller/HttpRequestHandler/旧式 Controller 统一适配，DispatcherServlet 只调 `adapter.handle()`

**一句话兜底：** Spring五模式：单例(容器级beanName唯一)、工厂(BeanFactory+FactoryBean)、代理(JDK接口/CGLib继承)、观察者(事件同步需异步配TaskExecutor)、适配器(Advisor统一MethodInterceptor/HandlerAdapter统一Handler)。

## plan-08-springcloud SpringCloud 篇

### Why SpringCloud

**定位：**

- SpringCloud 是一系列 Spring Boot 风格框架的有序集合，解决分布式系统基础设施问题
- SpringBoot 解决单个微服务快速搭建（自动装配 + Starter + 内嵌容器 + Actuator）
- SpringCloud 解决多个微服务协同（注册发现、负载均衡、网关、熔断、配置中心、链路追踪）
- 关系：SpringBoot 是地基，SpringCloud 是建在地基上的分布式全家桶

**SpringBoot 解决三大痛点：**

- ① 搭建繁琐：Starter 一键引入 + 自动装配替代 Maven 手工配 + XML 配置
- ② 部署依赖 Tomcat：内嵌容器 + fat jar + `java -jar` 替代 war + 外部容器
- ③ 监控简陋：Actuator 端点（health/metrics/env/loggers/threaddump）替代单一 /health 接口

**一句话兜底：** SpringBoot=单服务快速搭建(自动装配+Starter+内嵌容器)，SpringCloud=多服务协同全家桶(注册发现/网关/熔断/配置)；SpringBoot是SpringCloud基石。

### SpringBoot 深入

**Actuator 核心端点：**

- `/actuator/health`：聚合健康检查（含依赖中间件状态），K8s liveness/readiness probe
- `/actuator/metrics`：内存/CPU/GC/HTTP 指标，Prometheus 抓取
- `/actuator/env`：环境变量和配置属性
- `/actuator/loggers`：运行时动态修改日志级别（POST 改 level，不重启）
- `/actuator/threaddump`：线程转储，排查死锁

**自定义 Starter 五步：**

- ① 创建 `@ConfigurationProperties` 配置属性类
- ② 编写自动配置类，`@ConditionalOnMissingBean` 确保用户优先
- ③ `spring.factories`（或 Boot 3.x 的 `.imports`）注册自动配置类
- ④ 可选：`spring-configuration-metadata.json` 提供 IDEA 提示
- ⑤ 发布 Maven 依赖，命名规范 `xxx-spring-boot-starter`

**一句话兜底：** Actuator=生产级监控端点(health/metrics/loggers不重启改级别)；自定义Starter=配置类+自动配置类+spring.factories+条件注解+发布。

### Gateway 网关

**Gateway（WebFlux 非阻塞）vs Zuul 1.x（Servlet 阻塞）：**

- Gateway 用少量 EventLoop 线程（CPU 核数 × 2）消化海量并发
- Zuul 1.x 一个请求一个线程，IO 阻塞导致线程耗尽
- 性能约 1.6 倍，新项目默认选 Gateway

**三大组件：**

- Route：ID + 目标 URI + Predicate 列表 + Filter 列表
- Predicate：匹配请求属性（Path/Method/Header/Query/Host/Weight 等），全部 true 则命中该路由
- Filter：GatewayFilter（单路由）+ GlobalFilter（全局），分 pre/post 两阶段执行

**一句话兜底：** Gateway=WebFlux非阻塞网关(Route+Predicate+Filter)；pre过滤→转发→post过滤；线程模型优于Zuul1.x阻塞模型约1.6倍。

### Eureka 服务注册与发现

**核心机制（关键数字）：**

- 服务注册：Client 启动时向 Server 注册自己的 IP/端口/元数据，存入 Registry 内存表
- 心跳续约：每 30s 发一次心跳
- 服务注销：90s（3 个心跳周期）无心跳则剔除
- 客户端缓存：Client 本地缓存服务列表，所有 Server 全挂后调用链路仍不断

**CAP 定位：** Eureka 是 AP 系统（优先可用性），Server 间异步同步，允许短暂不一致。Zookeeper 是 CP 系统（优先一致性），Leader 选举期间拒绝服务。

**两级缓存（Server 端）：**

- readOnlyCacheMap（ConcurrentHashMap，30s 同步一次）
- readWriteCacheMap（Guava Cache，180s 过期）
- 代价：新服务上线最长延迟 = 180s(readWriteCacheMap) + 30s(readOnlyCacheMap) + 30s(Client) ≈ 4 分钟

**自我保护机制：**

- 触发条件：实际续租次数 < 期望值（实例数 × 频率 × 85%）
- 触发后不再注销任何过期实例，宁可保留死实例也不错杀
- 开发环境可关：`enable-self-preservation=false`，生产环境不要关

**一句话兜底：** Eureka=AP注册中心(30s心跳/90s剔除/客户端缓存保高可用)；两级缓存提升性能但增加发现延迟；自我保护宁可保死不杀活。

### Feign 声明式 HTTP 客户端

**核心原理：** JDK 动态代理 + 注解解析 → 接口方法调用自动转换为 HTTP 请求

**七大组件：**

- InvocationHandlerFactory：生成 JDK 代理
- Contract：解析 @GetMapping/@PathVariable 等注解
- Encoder/Decoder：请求序列化 / 响应反序列化
- Client：执行 HTTP（默认 HttpURLConnection 无连接池，生产换 Apache HttpClient）
- Retryer：IO 异常自动重试
- RequestInterceptor：统一添加请求头（TraceId/Token 透传）
- Logger：日志级别 NONE/BASIC/HEADERS/FULL

**最佳实践：** 继承特性（API 模块供提供方和消费方共用）、@SpringQueryMap 传多参数、自定义 ErrorDecoder 转业务异常、FULL 日志仅开发用

**一句话兜底：** Feign=JDK动态代理+接口注解自动转HTTP调用；7组件可替换；默认HttpURLConnection无连接池需换ApacheHttpClient。

### Ribbon 客户端负载均衡

**内置算法：**

- RoundRobinRule：轮询（默认）
- RandomRule：随机
- BestAvailableRule：最小并发
- WeightedResponseTimeRule：响应时间动态权重
- ZoneAvoidanceRule：同 Zone 优先（多机房）

**自定义场景：**

- 灰度发布：按 Header 标记路由到不同版本实例
- 多版本隔离：老客户端不调新版本服务
- 故障隔离：暂时踢出异常实例，排查完再恢复

**一句话兜底：** Ribbon=客户端负载均衡(从注册中心拉实例列表+本地选)；默认轮询/随机/最小并发/就近路由；灰度/多版本/故障隔离通过自定义IRule实现。

### Hystrix 服务容错保护

**服务雪崩：** 调用链中一个点慢导致上游线程堆积、资源耗尽、蔓延全系统。根因：同步调用+无超时+无降级+无隔离

**五大核心能力：**

- 封装请求：@HystrixCommand 包装方法调用，Hystrix 接管执行
- 资源隔离：线程池（独立线程池，彻底隔离，适合远程调用）/ 信号量（计数限流，轻量，适合网关/本地缓存）
- 断路器三态：CLOSED（正常）→ 失败达阈值 → OPEN（全走fallback）→ 休眠窗口到 → HALF_OPEN（放一个试探）→ 成功则CLOSED/失败则OPEN
- 失败回退：fallbackMethod 返回缓存/默认值/空结果
- 指标监控：成功/失败/超时/拒绝实时统计

**四种限流算法：**

- 计数器：简单但有临界突变问题（窗口边界两倍流量）
- 滑动窗口：小窗口拆分精确计数，解决临界突变
- 令牌桶：定速发令牌，桶容量=允许突发上限
- 漏桶：定速漏出，绝对平滑不允许突发

**三个坑：**

- ① 线程池隔离导致ThreadLocal丢失（线程切换），需重写HystrixConcurrencyStrategy传递
- ② 配置写死代码紧急难调，应设commandKey其余放配置中心动态下发
- ③ Gateway用线程池开销大（上百个线程池），应用信号量隔离+限流

**一句话兜底：** Hystrix=Command封装+线程池/信号量隔离+断路器(CLOSED/OPEN/HALF_OPEN)+fallback降级；限流算法令牌桶允许突发/漏桶绝对平滑；ThreadLocal丢失通过重写ConcurrencyStrategy解决。

### Sentinel 流量控制与熔断降级

**Sentinel vs Hystrix 核心差异：**

| 维度 | Hystrix | Sentinel |
|------|---------|---------|
| 规则配置 | 注解代码/配置中心 | Dashboard控制台UI实时生效 |
| 熔断恢复 | HALF_OPEN 试探一个 | 时间窗口到直接恢复 |
| 规则持久化 | 依赖外部配置中心 | 默认内存，需对接Nacos/Apollo |
| 框架依赖 | Spring Cloud Netflix | 无框架依赖 |
| 维护状态 | 2018年停维 | 阿里持续维护 |
| 监控粒度 | 请求级 | 500台集群+单机秒级 |

**关键差异——熔断恢复：** Hystrix做二次确认（放一个请求试探），更保守；Sentinel不做试探（窗口到直接恢复），更干脆

**坑：** Sentinel Dashboard规则默认存内存，停服即丢；生产必须持久化到Nacos

**一句话兜底：** Sentinel=Dashboard控制台+无试探直接恢复+规则需持久化到Nacos；替代已停维的Hystrix，新项目首选。

### Nacos 注册中心+配置中心

**定位：** Nacos = Eureka(注册中心) + Config(配置中心) + Bus(配置变更通知) 三合一

**数据模型三级隔离：**

- Namespace（最粗）：环境隔离 dev/test/prod，完全隔离
- Group（中等）：项目分组 ORDER_GROUP/PAYMENT_GROUP
- DataId（最细）：具体配置文件，命名规范 `${app}-${env}.${ext}`

**@RefreshScope 动态刷新：** 配置变更→发布RefreshEvent→标记Bean为脏→下次调用时用新配置重建（懒刷新，非即时）

**保护阈值 vs Eureka自我保护：**

- 触发条件：健康实例数/总实例数 < 阈值
- 行为：返回所有实例(含不健康)，宁可部分失败不可全雪崩
- Nacos额外支持 AP/CP 模式切换（Distro协议/Raft协议），比Eureka纯AP和ZK纯CP更灵活

**一句话兜底：** Nacos=Eureka+Config+Bus三合一；Namespace/Group/DataId三级隔离；@RefreshScope懒刷新；保护阈值与Eureka自我保护同AP逻辑。

### Bus / Stream 消息驱动与配置广播

**Spring Cloud Stream：**

- 通过 Binder 抽象层统一 MQ 编程模型，屏蔽 RabbitMQ/Kafka 底层差异
- 换 MQ 只换 Binder 依赖，业务代码不改
- 核心概念：Destination（消息目的地）、Binder（适配器）、Input（消费端）、Output（生产端）
- 新版已用函数式模型取代旧的 @EnableBinding/@StreamListener API

**Spring Cloud Bus：**

- 利用 MQ 广播配置变更事件 RefreshRemoteApplicationEvent
- 一次变更全网同步：POST /actuator/bus-refresh → 发事件到 MQ → 所有实例收到 → 各自刷新
- 用 Config Server + Git 架构时 Bus 几乎标配；用 Nacos 则不需要（Nacos 内置长轮询推送）

**一句话兜底：** Stream=MQ编程抽象层(Binder适配)，换MQ只换依赖不改代码；Bus=MQ广播配置刷新事件，一次变更全网同步。

### Sleuth 链路追踪

**核心概念：**

- Trace ID：一次请求在分布式系统中的唯一全局标识，贯穿整个调用链不变
- Span ID：每个服务处理单元的唯一标识，标记开始、过程和结束，用于统计各段耗时
- 关系：一个 Trace 包含多个 Span，Span 之间有父子关系，形成调用树

**Sleuth 作用：**

- 追踪服务间调用关系：经过哪些服务、处理时长
- 耗时分析：采样请求耗时，定位性能瓶颈
- 链路优化：发现频繁调用的服务，针对性优化
- 数据聚合：将追踪数据发送给 Zipkin 存储和可视化展示

**一句话兜底：** Sleuth=Trace ID(全局) + Span ID(分段)；追踪调用链+统计耗时；配合Zipkin聚合展示。

### 安全认证 — JWT 与 Token 实践

**四种认证方式对比：**

- Session：最传统，多节点丢失需 nginx 粘性 Cookie 或 Redis 集中存储
- HTTP Basic：请求头 Base64 加密用户名密码，最简单但安全性低
- Token：存储用户信息+加密算法加密，服务端解密获取信息，无状态
- JWT：RSA 算法生成 token，封装用户 id 和有效期，标准格式 Header.Payload.Signature

**JWT 认证流程：**

- ① 认证服务校验用户信息，返回认证结果
- ② JWTUtils 用 RSA 算法生成 token，封装用户 id 和有效期
- ③ 服务间参数通过请求头传递，服务内部通过 ThreadLocal 上下文传递
- ④ Hystrix 线程池隔离导致 ThreadLocal 丢失 → 重写 HystrixConcurrencyStrategy

**Token 最佳实践：**

- 设置较短（合理）的过期时间
- 注销的 Token 及时清除（放 Redis 做一层过滤）
- 监控 Token 使用频率（防爬虫）
- 核心功能敏感操作加动态验证（验证码）
- 网络环境和浏览器信息识别（银行 APP 级安全）
- 加密密钥支持动态修改（存配置中心，低峰更换）

**一句话兜底：** JWT=RSA生成Token(Header.Payload.Signature)+请求头传递+服务内ThreadLocal；密钥低峰动态更换；Hystrix线程池丢ThreadLocal需重写ConcurrencyStrategy。

### 灰度发布

**核心思路：** 非全量发布，逐步放量验证。部署 1-2 个新版本实例 → 灰度标记请求路由到新实例 → 观察运行 → 逐步扩大比例 → 出问题只回滚灰度节点

**实现方式：**

- Ribbon 自定义规则：根据 X-Gray-Tag 请求头分流，灰度用户→v2 实例，普通用户→v1 实例
- Discovery 组件：基于 Eureka/Ribbon 的高层灰度抽象，内置灰度策略管理
- Eureka Zone：灰度实例注册到专用 Zone，利用就近路由特性隔离流量

**一句话兜底：** 灰度发布=逐步放量+按标记分流(Ribbon自定义规则/Zone隔离)；出问题只回滚灰度节点，不影响全量用户。

### 多版本隔离

**与灰度发布区别：** 灰度是临时发布策略（验证完就摘），多版本隔离是长期运行策略（v1/v2 长期共存）

**发布流程：** ①摘流量→②升级节点→③测试验证→④接入正常流量→⑤发布下一个节点，逐台替换

**实现方式：** Eureka 元数据标记 version + Ribbon 按版本号路由 + 请求头 X-API-Version 标识客户端版本

**一句话兜底：** 多版本隔离=Eureka元数据version标记+Ribbon按版本路由+客户端请求头API版本号；解决不兼容接口升级时新旧客户端长期共存。

### 各组件调优（SpringCloud篇附）

**关键参数：**

- Hystrix 信号量默认 maxConcurrentRequests=100，网关场景按需调大
- Feign HttpURLConnection→httpclient（连接池），日志生产用 BASIC 不用 FULL
- Tomcat: accept-count(队列)、max-threads(线程数默认200)、max-connections(最大连接数)
- Gateway/Feign 配合配置中心实现动态路由和动态日志

**一句话兜底：** 调优三件套：Hystrix信号量调大/Fiegn换httpclient/Tomcat线程池参数评估；配合配置中心动态调整。


## plan-09-distributed 分布式篇

### 分布式发展历程

**演进路径：**

- 阶段一：单应用架构 — 应用服务和数据服务分离 → 应用服务集群 → 应用服务中心化 SAAS
- 阶段二：数据库主备读写分离 — 全文搜索引擎加速统计 → 缓存集群缓解读压力 → MQ 缓解写压力 → 水平/垂直拆分适应微服务
- 阶段三：划分上下文拆分微服务 — 服务注册发现（Eureka/Nacos）、配置动态更新（Config/Apollo）、灰度发布（Gateway/Feign）、安全认证（Gateway/Auth）、降级限流（Hystrix/Sentinel）、监控（Actuator/Prometheus）、链路追踪（Sleuth/Zipkin）

**原理：**

- 分布式系统 = 组件分布在不同的网络计算机上，彼此仅通过消息传递通信和协调
- 演进动力：单机容量不足 → 读写分离 → 缓存/MQ 削峰 → 微服务拆分 → 基础设施配套

**一句话兜底：** 分布式演进三阶段：单应用→读写分离+缓存MQ→微服务拆分+注册发现/配置/网关/熔断/链路追踪全套基础设施。

### CAP 理论

**三个指标：**

- C（Consistency）一致性：所有节点同一时刻看到相同数据
- A（Availability）可用性：每次请求都能获得非错响应，不保证是最新数据
- P（Partition tolerance）分区容忍性：网络分区发生时系统仍能继续工作

**原理：**

- CAP 只能同时满足两个，P 在分布式系统中必须选（网络不可靠）
- 选 CP：强一致，牺牲可用性（如 Zookeeper，Leader 挂掉时不可用）
- 选 AP：高可用，牺牲一致性（如 Eureka，peer 同步是异步的，可能读到旧数据）
- BASE 理论是 AP 的工程补充：基本可用（Basically Available）+ 软状态（Soft state）+ 最终一致性（Eventual Consistency）

**坑点：**

- ① P 不是可选项——分布式系统中网络分区一定会发生，P 必须满足
- ② 现实系统不是极端 CP 或 AP，而是在 CP/AP 之间做程度取舍

**一句话兜底：** CAP：一致性/可用性/分区容错三选二；P必须选，实际在CP(ZK强一致)和AP(Eureka高可用)间取舍；BASE是AP的工程折衷。

### 分布式一致性协议（XA / 2PC / 3PC）

**2PC 两阶段提交：**

- 阶段一 Prepare：协调者问所有参与者是否可以提交，参与者写 Undo/Redo 日志后回复 Yes/No
- 阶段二 Commit：全 Yes → 执行 Redo 提交；有 No/超时 → 执行 Undo 回滚

**3PC 三阶段提交：**

- CanCommit：询问能否提交，超时或 No 则中断
- PreCommit：全 Yes 后写 Undo/Redo 日志
- DoCommit：执行 Redo 提交或 Undo 回滚
- 相比 2PC 的改进：参与者自身增加了超时机制，失败可及时释放资源

**坑点：**

- ① 2PC 准备阶段资源锁定，严重时造成死锁
- ② 提交阶段出现网络异常，部分节点收到部分未收到，造成数据不一致
- ③ 协调者单点故障 → 整个事务阻塞

**一句话兜底：** 2PC=协调者两阶段(P=预提交写日志/C=提交或回滚)，3PC=三阶段加超时免死锁；两者都有资源锁+单点+网络异常一致性风险。

### Paxos 算法

**角色：**

- Client：发起请求的客户端
- Proposer：提案发起者，根据 Acceptor 返回选最大编号 N 对应的 V，发 [N+1, V]
- Acceptor：决策者，Accept 后拒绝小于 N 的提案，返回自己的 [N, V]
- Learner：最终决策的学习者，负责复制

**核心约束：**

- P1：Acceptor 必须接受收到的第一个提案
- P2a/P2b：某个 v 被 accept 后，所有更高编号的提案值必须也是 v
- 通过半数以上 Acceptor 集合 S 保证：S 接受的提案若小于 Mid，编号最大的值为 Vid

**活性问题（活锁）：**

- 两个 Proposer 交替提递增编号的提案导致死循环
- 解决：选主 Proposer（只有主能提案）+ 提案间隔随机化

**一句话兜底：** Paxos 通过多数派 Acceptor 投票+提案编号递增→值唯一确定；角色：Proposer提案/Acceptor投票/Learner复制；活锁靠选主Proposer解决。

### ZAB 协议

**定位：** Zookeeper 使用的原子广播协议，为 Paxos 的工程简化版，专门为分布式协调设计。

**两种模式：**

- 崩溃恢复模式：选举 Leader，同步各 Follower 数据至一致
- 消息广播模式：Leader 将事务请求转为 Proposal 广播，过半 Ack 后提交

**与 Paxos 的关键区别：**

- Paxos 允许乱序提交，ZAB 要求严格按 zxid 顺序
- ZAB 有明确的 Leader 角色，Paxos 中 Proposer 可以任意发起

**一句话兜底：** ZAB=ZK原子广播协议，分崩溃恢复(选Leader同步)+消息广播(过半ACK提交)；相比Paxos增加严格顺序性和固定Leader。

### Raft 算法

**角色：** Leader（处理所有写请求）、Follower（被动接收）、Candidate（选举中的临时角色）

**选举机制：**

- 心跳触发选举，超时时间 150-300ms 随机
- Follower 超时未收到 Leader 心跳 → 变为 Candidate → 发起选举
- 获得半数以上投票 → 成为 Leader

**异常处理：**

- Leader 异常：Follower 超时选举新 Leader，选出后比较步长同步
- Follower 异常：恢复后直接从 Leader 当前状态同步
- 多个 Candidate 分票：选举失败，超时后重新选举（随机超时避免再次分票）

**一句话兜底：** Raft=强Leader一致性算法，心跳选举(150-300ms超时)、过半投票、Leader异常自动重选；相比Paxos更易理解和实现。

### 数据库与缓存一致性（Canal + Binlog）

**方案：订阅 Binlog 同步缓存**

- 通过 Canal 等工具模拟 MySQL 从库，读取主库 Binlog
- 解析变更数据后直接写入 Redis 缓存
- Binlog 基于 ACK 机制：同步失败不确认，下次重复消费保证最终一致

**优点：**

- 读取速度大幅提升、延迟降低
- Binlog ACK 机制解决了分布式事务问题

**缺点：**

- 增加系统复杂度、消耗缓存资源
- 需要筛选压缩数据、极端情况可能丢失

**一句话兜底：** Canal伪装MySQL从库→消费Binlog→同步Redis；ACK保证最终一致，适合读多写少高并发场景。

### 可用性保障

**心跳检测：**

- 周期检测：以固定频率向其他节点汇报状态，携带元数据
- 累计失效检测：超时未返回 → 重试 → 超次数判定故障

**多机房实时热备：**

- 两套缓存集群部署在不同城市机房
- 读服务就近访问同机房缓存
- 优点：性能提升（就近访问）+ 可用性提升（秒级切流）
- 代价：资源成本翻倍

**一句话兜底：** 可用性三板斧：心跳检测(周期+累计失效)、多机房热备(就近访问+秒级切流)、限流降级兜底。

### 分区容错性

**定义：** 分布式系统对网络分区（节点间通信中断）的包容能力。

**应对手段：**

- 限流：控制进入系统的请求量，防止过载
- 降级：关停非核心功能，保证核心链路
- 兜底：返回默认值或缓存数据而非报错
- 重试：失败自动重试，配合幂等
- 负载均衡：剔除故障节点，路由到健康节点

**日志复制：**

- Leader 发 RPC 给 Follower 复制日志条目
- Leader 不断重试直到所有 Follower 返回 ACK
- 通知所有 Follower 提交，返回客户端

**一句话兜底：** 分区容错=网络分区下系统仍可用，靠限流/降级/兜底/重试/负载均衡五件套；日志复制=Leader发RPC+重试至全ACK+提交。

### 高可用架构模式

**主备（Master-Slave）：**

- 主机宕机 → 备机接管；主机恢复 → 热备自动或冷备手动切回
- 常见于 MySQL 主从复制（Binlog → I/O线程 → 中继日志 → SQL线程回放）

**互备（Active-Active / MM）：**

- 两台主机同时运行、相互监测
- 每个 master 都有读写能力，按时间戳或业务逻辑合并版本

**集群（Cluster）模式：**

- 多节点运行，主控节点分担请求（如 Zookeeper）
- 主控节点本身的高可用靠主备模式解决

**一句话兜底：** 三种高可用模式：主备(热备/冷备)、互备(双活MM按时间戳合并)、集群(多节点+主控节点主备)。

### TCC 分布式事务

**三个阶段：**

- Try：对各服务资源做检测，锁定或预留资源
- Confirm：各服务执行实际操作
- Cancel：任一服务出错，执行补偿/回滚

**适用场景：** 短事务、严格资金要求场景（如转账、扣库存）

**一句话兜底：** TCC=Try预留资源+Confirm执行+Cancel回滚，适合严格资金一致性短事务；缺点是实现复杂需要写补偿逻辑。

### Saga 分布式事务

**定义：** 事务性补偿方案，将长事务拆成多个本地事务，每个本地事务有对应的补偿操作。

**适用场景：** 流程长、环节多、调用第三方业务

**一句话兜底：** Saga=长事务拆短事务链+每步配补偿；流程长/调用第三方时用；失败则逆序调用补偿回滚。

### 本地消息表方案

**流程：**

- 本地事务执行时，将消息写入本地消息表（同一事务内）
- 定时任务轮询消息表，发送到 MQ
- 消费方处理成功后回调或更新消息状态

**一句话兜底：** 本地消息表=业务数据+消息同库同事务写入→定时轮询发MQ→消费方幂等处理；eBay经典方案，简单可靠。

### MQ 最终一致性方案（RocketMQ 事务消息）

**流程：**

1. A 系统发 prepared 消息到 MQ，失败则取消操作
2. 发送成功 → 执行本地事务，成功发确认、失败发回滚
3. B 系统收到确认消息 → 执行本地事务
4. MQ 定时轮询 prepared 消息的回调接口，确认事务状态
5. B 失败则自动重试至成功，达上限后人工介入

**选型指南：**

- 严格资金不能错：TCC
- 一般分布式事务（如积分）：可靠消息最终一致性
- 允许不一致：最大努力通知

**一句话兜底：** RocketMQ事务消息=prepared消息+本地事务+确认/回滚+定时回调轮询+消费方重试幂等；核心是双端确认+重试幂等。

### 分布式 Session

**三种方案：**

- JWT Token：无状态，数据从 cache 或 DB 获取，适合微服务
- Tomcat + Redis：配置 conf 文件，Session 集中存 Redis
- Spring Session + Redis：支持 SpringCloud/SpringBoot，开箱即用

**一句话兜底：** 分布式Session三方案：JWT无状态Token(推荐)、Tomcat Redis集中存储、Spring Session Redis(Spring生态首选)。


## plan-10-es ES篇

### ES 概述

**定位：** 基于 Lucene 的分布式海量数据近实时搜索引擎。

**特点：**

- 安装方便：无其他依赖，几行配置可搭集群
- JSON 输入输出：不需要定义 Schema
- RESTful：索引/查询/配置全走 HTTP
- 分布式：节点对外对等，自动负载均衡
- 多租户：不同用途分不同索引
- 超大数据：可扩展到 PB 级，近实时处理

**功能：**

- 分布式搜索引擎：自动将海量数据分散到多台服务器
- 全文检索：模糊搜索 + 相关性排名 + 高亮
- 数据分析引擎：分组聚合（group by）、指标聚合（max/min/avg）
- 近实时处理：秒级延迟

**一句话兜底：** ES=基于Lucene的分布式近实时搜索引擎，JSON+RESTful+PB级扩展；三功能：全文检索+数据分析+近实时。

### ES 竞品对比

**Lucene：** Java 写的搜索工具包（Jar），只是框架，直接使用复杂。

**Solr：** 基于 Lucene 的 HTTP 查询服务器，封装了 Lucene 细节。

**Elasticsearch vs Solr：**

| 维度 | ES | Solr |
|------|-----|------|
| 分布式管理 | 自带协调管理 | 依赖 Zookeeper |
| 功能范围 | 聚焦核心，高级功能靠第三方插件 | 实现更全面 |
| 实时搜索 | 更强 | 传统搜索更优 |

**现状：** 主流仍是 ES 7.x，最新 7.8；升级亮点：默认集成 JDK、Lucene8 提升 TopK、引入熔断防 OOM。

**一句话兜底：** ES vs Solr：ES自带分布式管理实时搜索更强，Solr功能全面传统搜索更优依赖ZK；当前主流ES 7.x。

### IK 分词器

**定位：** 开源的轻量级中文分词工具包，独立于 Lucene。

**特性：**

- "正向迭代最细粒度切分算法"，60 万字/秒处理能力
- 多子处理器：英文（IP/Email/URL）、数字（日期/数量词/科学计数法）、中文（姓名/地名）
- 支持个人词条优化词典，更小内存占用
- 三种词典：扩展词典 `ext_dict`、停用词典 `stop_dict`、同义词典 `same_dict`

**一句话兜底：** IK分词器=中文分词工具包，正向迭代最细粒度切分+多子处理器+三词典(扩展/停用/同义)；核心解决中文分词语义问题。

### ES 索引与映射

**索引（Index）：** 类比数据库的 database，定义 settings（分片数、副本数等）

**映射（Mapping）：** 类比表结构设计，定义：
- 字段的数据类型（text/keyword/long/date/geo_point）
- 分词器类型
- 是否存储、是否需要创建索引

**文档（Document）：** 类比一行数据
- 全量更新用 PUT
- 局部更新用 POST

**一句话兜底：** 索引=数据库(settings分片副本)、映射=表结构(字段类型+分词器)、文档=数据行(PUT全量/POST局部)。

### DSL 查询

**查询分类：**

- 查询所有：`match_all`
- 全文搜索：`match`（匹配）/ `match_phrase`（短语）/ `query_string` / `multi_match`（多字段）
- 词条级搜索：`term`（精确）/ `terms`（集合）/ `range`（范围）/ `prefix`（前缀）/ `wildcard`（通配符）/ `regexp`（正则）/ `fuzzy`（模糊）
- 复合搜索：`bool` 组合 `must`/`should`/`must_not`/`filter`
- 辅助：`sort` 排序、`size` 分页、`highlight` 高亮、`bulk` 批量

**一句话兜底：** ES DSL=JSON查询语言；全文(match/phrase/multi_match)+词条(term/range/fuzzy)+复合(bool组合)+辅助(sort/size/highlight/bulk)。

### 聚合分析

**指标聚合（Metric）：** 对数据集求 max/min/sum/avg 等

**桶聚合（Bucketing）：** 对数据分桶 group by，再在桶上做指标聚合

**一句话兜底：** ES聚合=指标聚合(max/min/avg)+桶聚合(group by后再指标)；等同于SQL的聚合函数+GROUP BY。

### 智能搜索建议（Suggester）

**四种 Suggester：**

- Term Suggester：基于编辑距离的纠错建议
- Phrase Suggester：基于短语级别的纠错
- Completion Suggester：基于前缀匹配的自动补全（最快）
- Context Suggester：带上下文过滤的补全

**精度与召回：**
- 精准度：Completion > Phrase > Term
- 召回率：Term > Phrase > Completion
- 性能：Completion 最快，Phrase/Term 需走倒排索引

**一句话兜底：** ES搜索建议：Completion(前缀补全最快最准)、Phrase(短语纠错)、Term(单词语义纠错召回最高)；精度↔召回互为倒数。

### 写优化

**策略：**

- 副本数置 0：首次导入数据时先关副本，写完再开
- 自动生成 ID：避免写前判断文档是否已存在
- 合理使用分词器：binary 类型不做分词，title/text 用不同分词器
- 禁用评分 + 延长索引刷新间隔（`refresh_interval`）
- 批量操作：多个索引操作放入 bulk 一起提交

**一句话兜底：** ES写优化五招：关副本+自生成ID+合理分词+禁用评分延长刷新+bulk批量写入。

### 读优化

**策略：**

- 用 Filter 代替 Query：跳过打分环节，利用缓存
- bool 组合 query 和 filter：需要打分的走 query，精确匹配走 filter
- 按时间维度分组索引：查询集中在局部 index，减少扫描范围

**一句话兜底：** ES读优化：Filter替代Query免打分、bool组合query+filter、按日/月分组索引缩小扫描范围。

### 零停机索引重建

**三种方案：**

| 方案 | 做法 | 优缺点 |
|------|------|--------|
| 自研 MQ 导入 | MQ触发→微服务消费→分页查DB→bulk写ES | 参与度灵活性最高，稳定性低 |
| Scroll + bulk + 别名 | 建新索引→scroll批量读出→bulk批量写入→切换别名 | 中等 |
| Reindex API（v6.3.1+） | 一行 API 调用，内置 scroll+bulk 封装 | 稳定性最高，灵活性最低 |

**参与度 & 灵活性：** 自研 > scroll+bulk > reindex
**稳定性 & 可靠性：** 自研 < scroll+bulk < reindex

**一句话兜底：** 零停机重建=建新索引+数据迁移+切别名；三个方案：自研MQ导入(灵活)、scroll+bulk+别名(均衡)、Reindex API(稳定)。

### Deep Paging 性能

**问题：** `from + size` 深分页（如 from=10000, size=10）会导致每个分片检索 10010 条数据，coordinator 汇总排序，内存和耗时线性增长。

**解决方案：**

- 业务层限制深度，不允许跳过大页
- 使用 `search_after`：基于上次结果的排序值继续查，避免 from 开销
- 使用 `scroll`：适合全量导出场景

**一句话兜底：** ES深分页from+size=每个分片取from+size条汇总排序→开销线增；用search_after(实时)或scroll(全量导出)替代。


## plan-11-docker-k8s Docker & K8S 篇

### Why Docker

**三个核心场景：**

- 开发：本地写代码，Docker 容器与同事共享成果
- 测试：Docker 推送应用到测试环境，执行自动化+手动测试
- 生产：修复后推送更新镜像，部署到生产

**核心价值：**

- 快速一致交付：镜像打包环境，避免环境不一致
- 简化开发生命周期：适合快速迭代、敏捷开发

**一句话兜底：** Docker=镜像打包环境→一致交付→简化开发/测试/生产三环境流转；解决"在我机器上能跑"问题。

### Docker vs 虚拟机

**虚拟机架构：**

- 物理机 → Hypervisor 虚拟化层 → 多个 Guest OS → 各自的 App
- 优点：提升 IT 效率、更快部署、提高可用性
- 缺点：占用资源多、性能较差、扩展迁移能力差

**Docker 架构：**

- 物理机 → Host OS → Docker Engine → 多个容器（共享内核，各自隔离）
- 无 Hypervisor 层，无 Guest OS，直接复用 Host 内核

**核心差异：**

| 维度 | 虚拟机 | Docker |
|------|--------|--------|
| 启动速度 | 分钟级 | 秒级 |
| 资源占用 | GB 级 | MB 级 |
| 隔离级别 | 完全隔离（OS级） | 进程级隔离（共享内核） |
| 镜像大小 | 几 GB | 几十 MB ~ 几百 MB |

**一句话兜底：** VM=物理机+Hypervisor+独立GuestOS→重慢全隔离；Docker=共享Host内核+进程隔离→轻快秒启动；Docker不是轻量级VM而是隔离加强的进程。

### Docker 引擎

**过程：**

- Client 发指令（`docker run`、`docker build`），Daemon（dockerd）执行
- 通信：本机走 Unix Socket（`/var/run/docker.sock`），远程走 REST API

**原理：**

- C/S 架构，引擎负责三件事：构建（build）、运行（run）、分发（push/pull）
- 引擎调度内核 namespace + cgroups 实现隔离与资源限制

**坑点：**

- ① 把用户加入 docker 组等于给 root 权限——拥有 docker.sock 读写权即可提权
- ② 生产环境不要暴露 dockerd 的 REST API 到公网，未授权访问可控制宿主机

**一句话兜底：** Docker引擎=C/S架构(dockerd守护进程)，Client通过Socket/REST发指令，引擎负责build/run/push三件事。

### Docker 镜像

**过程：**

- Dockerfile 每条指令生成一层：`FROM` → `COPY` → `RUN` → `CMD`
- 容器启动 = 只读镜像层之上叠加一个可写容器层（Container Layer）

**原理：**

- 基于 UnionFS，各层叠加形成最终文件系统视图
- 写时复制（Copy-on-Write）：修改下层文件时先拷贝到可写层再改
- 同层共享：多个镜像用同一基础镜像（如 ubuntu:20.04），底层只存一份
- 拉取镜像时只传本地缺少的层

**坑点：**

- ① 镜像层数越多越臃肿，Dockerfile 指令合理合并（`RUN apt-get update && apt-get install -y pkg1 pkg2 && rm -rf /var/lib/apt/lists/*`）
- ② `COPY` / `ADD` 前后依赖不变的内容放前面，利用构建缓存

**一句话兜底：** 镜像=UnionFS分层只读模板，一层一条指令；容器启动加可写层；同底共享省空间、只传缺失层。

### Docker 仓库

**过程：**

- `docker pull mysql:5.7.30`：从仓库拉镜像到本地
- `docker tag mysql:5.7.30 myrepo/mysql5`：打标签
- `docker push myrepo/mysql5`：推送到仓库

**原理：**

- Registry = 仓库服务器（Docker Hub / Harbor）
- Repository = 具体镜像名（`library/mysql`），可存多个 TAG 版本
- 公开仓库：Docker Hub；私有仓库：企业自建 Harbor

**坑点：**

- ① `latest` tag 只是默认值，不代表最新版本，生产环境必须指定精确版本号（如 `mysql:8.0.33`）
- ② 镜像名不带 Registry 前缀默认走 Docker Hub，自建 Harbor 需写全 `harbor.company.com/project/image:tag`

**一句话兜底：** 仓库=Registry(服务器)+Repository(镜像名)+TAG(版本)；公开DockerHub，私有Harbor；pull/push分发。

### 容器基本操作

**镜像操作：**

- `docker pull mysql:5.7.30`：从仓库拉镜像，不指定 TAG 默认 `:latest`
- `docker images`：列出本地镜像，关注 REPOSITORY / TAG / SIZE
- `docker tag mysql:5.7.30 mysql5`：打标签，不是复制，同一 IMAGE ID 多个别名
- `docker inspect mysql:5.7.30`：查看镜像/容器 JSON 元数据（分层、环境变量等）
- `docker search mysql`：搜索 Docker Hub
- `docker rmi mysql:5.7.30`：删除镜像，有容器引用时删不掉
- `docker push myimage:v1`：推送到远程仓库

**容器生命周期：**

- `docker create` → 创建但不启动
- `docker start` → 启动已创建的容器
- `docker run` = create + start（组合）
- `docker stop` → 优雅停止（SIGTERM → 超时 → SIGKILL）
- `docker rm` → 删除已停止的容器
- `docker ps` → 运行中的容器，`-a` 显示全部（含已停止）

**核心参数：**

- `-d`：后台运行
- `-it`：`-i` stdin 打开 + `-t` 伪终端，交互场景用
- `--rm`：退出后自动删除容器
- `--name`：容器命名
- `-p 8080:8080`：端口映射（宿主机:容器）
- `-v /data:/app`：卷挂载
- `--network host`：使用宿主机网络栈

**坑点：**

- ① `docker tag` 不复制数据，删标签不是删镜像——IMAGE ID 没其他 tag 时才算真正删除
- ② `docker run` 不加 `--rm` 会留下已停止容器，日积月累占磁盘
- ③ `--network host` 端口直接绑定宿主机，注意端口冲突
- ④ 清理命令：`docker system prune -a` 清理未使用的镜像/容器/网络/缓存

**一句话兜底：** 镜像操作=CRUD+tag加别名，容器生命周期=create/start/run/stop/rm，docker run=create+start，核心参数-d/-it/--rm/-p/-v。

### Docker 数据卷（Volume）

**两种挂载方式：**

| 方式 | 命令 | 数据位置 | 场景 |
|------|------|---------|------|
| Volume | `-v my_vol:/data` | `/var/lib/docker/volumes/` | 持久化、共享 |
| Bind mount | `-v /host/path:/data` | 宿主机指定路径 | 开发热更新、配置注入 |

**过程：**

- `docker volume create test_volume`：创建命名卷
- `docker run -v test_volume:/test nginx`：挂载卷到容器
- 多个容器挂同一 volume → 共享同一份数据
- `docker volume ls` / `docker volume rm` / `docker volume prune`

**原理：**

- Volume 独立于容器生命周期，容器删了 volume 还在
- Volume 由 Docker 管理存储路径，bind mount 由用户指定宿主机路径
- 命名 volume 永不会被 `docker rm` 自动删除，需手动清理

**坑点：**

- ① Bind mount 宿主机路径不存在时 Docker 自动创建，但权限可能是 root
- ② Bind mount 一个不存在文件 → Docker 创建同名目录，是个经典坑
- ③ `-v` 不支持容器内相对路径，必须绝对路径
- ④ 复用已有 volume 名不报错，自动复用旧 volume

**一句话兜底：** Volume=容器删数据在，两种挂载：命名卷(Docker管理)和bind mount(宿主机直通)；多容器同卷=数据共享。

### Docker 网络

**四种模式：**

| 模式 | 命令 | 原理 | 场景 |
|------|------|------|------|
| bridge | `--network bridge`（默认） | docker0 网桥 + NAT，每容器独立 IP | 一般单机 |
| host | `--network host` | 直接复用宿主机网络栈，`-p` 失效 | 高性能 |
| none | `--network none` | 仅 lo 回环，完全隔离 | 安全敏感 |
| container | `--network container:<name>` | 共享另一容器网络栈 | sidecar |

**bridge 模式原理：**

- `docker0` 虚拟网桥，容器分配 172.17.0.0/16 私有 IP
- 容器出外网：iptables SNAT 替换源 IP 为宿主机 IP
- 外网入容器：`-p` 端口映射 = iptables DNAT 转发

**容器间通信：**

- 同自定义 bridge：直接 `ping <容器名>`，内置 DNS
- 同默认 bridge：只能用 `--link`（已废弃）或 IP
- 跨 bridge：默认隔离，需 `docker network connect`

**坑点：**

- ① host 模式 `-p` 参数失效且不报错，端口冲突难排查
- ② 默认 bridge 不支持 DNS 容器名解析，生产一律用自定义 bridge
- ③ `docker network create` 创建的 bridge 才有 DNS，默认 bridge 没有

**一句话兜底：** Docker四种网络：bridge(NAT+独立IP)、host(宿主机直通)、none(纯隔离)、container(sidecar共享)；端口映射=iptables DNAT；自定义bridge支持DNS。


## plan-12-netty Netty 篇

### Netty 整体架构分层

**整体三层结构：**

| 层 | 职责 | 一句话 |
|------|------|------|
| Transport Service | TCP/UDP/Socket 传输抽象 | 怎么传 |
| Core | 事件模型、通用API、零拷贝ByteBuf | 怎么处理 |
| Protocol Support | HTTP/Protobuf/WebSocket 编解码 | 怎么翻译 |

**Core 层三个能力：**

- 事件模型：所有网络行为统一为事件（channelActive/Read/Inactive/exceptionCaught），通过 Pipeline Handler 链传播
- 通用 API：Channel 接口统一抽象（register/bind/connect/read/write/flush），底层切换 NIO/epoll/kqueue 上层代码不变
- ByteBuf：读写指针分离（不需 flip）、自动扩容、引用计数+对象池、零拷贝 CompositeByteBuf

**逻辑架构三层组件：**

- 网络通信层：Bootstrap（客户端启动器）/ ServerBootstrap（服务端，区分 boss 接连接 + worker 干读写）/ Channel（I/O 操作抽象）
- 事件调度层：EventLoop（单线程绑定一个 Channel，串行执行保证无锁）/ EventLoopGroup（线程池，支持单线程/多线程/主从多线程三种 Reactor 模式）
- 服务编排层：Pipeline（双向链表组装 Handler，入站 Head→Tail，出站 Tail→Head）/ Handler（Inbound/Outbound 两类）/ HandlerContext（Handler 与 Pipeline 桥梁，fireXxx 方法传播事件）

**坑点：**

- ① Handler 里不调 `ctx.fireChannelRead(msg)` → 事件传播断链，后续 Handler 收不到
- ② 入站出站方向搞反：Read 入站走 Head→Tail，Write 出站走 Tail→Head
- ③ bossGroup 线程太多浪费：boss 只负责 accept，1-2 个线程足够

**一句话兜底：** Netty整体三层：传输层(怎么传)→Core(事件+API+ByteBuf)→协议层(编解码)；逻辑三层：通信层(Bootstrap/Channel)→调度层(EventLoop/Pool)→编排层(Pipeline双向链表+Handler)。

