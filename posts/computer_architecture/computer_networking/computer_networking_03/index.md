# 计算机网络自顶向下方法 —— 运输层


---

运输层协议为运行在不同主机上的应用进程之间提供了**逻辑通信**（logic communication）功能。

{{< image src="/images/computer_networking/03_01.jpg" caption="运输层在应用程序进程间提供逻辑的而非物理的通信" title="运输层在应用程序进程间提供逻辑的而非物理的通信" >}}

运输层协议是在端系统中而不是在路由器中实现的。运输层分组称为运输层**报文段**（segment）。

---

## 运输层和网络层的关系

网络层提供了**主机**之间的逻辑通信，而运输层为运行在不同主机上的**进程**之间提供了逻辑通信。

因特网网络层协议有一个名字叫 IP,即网际协议。IP 为主机之间提供了逻辑通信。IP 的服务模型是尽力而为交付服务(best-effort delivery service)。这意味着 IP 尽它“最大的努力”在通信的主机之间交付报文段，但它并不做任何确保。它不确保报文段的交付，不保证报文段的按序交付，不保证报文段中数据的完整性。由于这些原因，IP 被称为**不可靠服务**(unreliable service)。

UDP 和 TCP 最基本的责任是，将两个端系统间 IP 的交付服务扩展为运行在端系统上的两个进程之间的交付服务。将主机间交付扩展到进程间交付被称为**运输层的多路复用**(transport-layer multiplexing)与**多路分解**(demultiplexing)。

UDP 和 TCP 还可以通过在其报文段首部中包括差错检查字段而提供完整性检查。程到进程的数据交付和差错检查是两种最低限度的运输层服务，也是 UDP 所能提供的仅有的两种服务。UDP 也是一种不可靠的服务。

TCP 为应用程序提供了几种附加服务。首先，它提供**可靠数据传输**(reliable data transfer)。通过使用流量控制、序号、确认和定时器，TCP 确保正确地、按序地将数据从发送进程交付给接收进程。TCP就将两个端系统间的不可靠 IP 服务转换成了一种进程间的可靠数据传输服务。

TCP 还**提供拥塞控制**(congestion control)。TCP 拥塞控制防止任何一条 TCP 连接用过多流量来淹没通信主机之间的链路和交换设备。

---

## 多路复用与多路分解

运输层的多路复用与多路分解，也就是将由网络层提供的主机到主机交付服务延伸到为运行在主机上的应用程序提供进程到进程的交付服务。

在接收端，运输层检查这些字段，标识出接收套接字，进而将报文段定向到该套接字。将运输层报文段中的数据交付到正确的套接字的工作称为**多路分解**(demultiplexing)。

在源主机从不同套接字中收集数据块，并为每个数据块封装上首部信息(这将在以后用于分解)从而生成报文段，然后将报文段传递到网络层，所有这些工作称为**多路复用**(nmhiplexing)。

{{< image src="/images/computer_networking/03_02.jpg" caption="多路复用与多路复用分解" title="多路复用与多路复用分解" >}}

运输层多路复用要求：①套接字有唯一标识符；②每个报文段有特殊字段来指示该报文段所要交付到的套接字。特殊字段是**源端口号字段**(source port number field)和**目的端口号字段**(destination port number field)。

运输层是怎样能够实现分解服务的了：在主机上的每个套接字能够分配一个端口号，当报文段到达主机时，运输层检査报文段中的目的端口号，并将其定向到相应的套接字。然后报文段中的数据通过套接字进入其所连接的进程。

---

## 无连接运输：UDP

有许多应用更适合用 UDP,原因主要以下几点：

- 关于发送什么数据以及何时发送的应用层控制更为精细
- 无须连接建立
- 无连接状态
- 分组首部开销小

### UDP 报文段结构

{{< image src="/images/computer_networking/03_03.jpg" caption="UDP 报文段结构" title="UDP 报文段结构" >}}

### UDP 检验和

UDP 检验和提供了差错检测功能。假定我们有下面 3 个 16 比特的字，计算原理如下：

{{< image src="/images/computer_networking/03_04.jpg" caption="UDP 检验和" title="UDP 检验和" >}}

和 0100101011000010 的反码运算结果是 1011010100111101，这就是检验和。

在接收方，全部的 4 个 16 比特字（包括检验和）加在一起。和将是 1111111111111111，表明未出现差错。

---

## 面向连接的运输：TCP

TCP 被称为是**面向连接**的（connection-oriented）,这是因为在一个应用进程可以开始向另一个应用进程发送数据之前，这两个进程必须先相互“握手”，即它们必须相互发送某些预备报文段，以建立确保数据传输的参数。

连接建立过程常被称为**三次握手**（three-way handshake）。一旦建立起一条 TCP 连接，两个应用进程之间就可以相互发送数据了。

TCP 连接的组成包括：一台主机上的缓存、变量和与进程连接的套接字，以及另一台主机上的另一组缓存、变量和与进程连接的套接字。

{{< image src="/images/computer_networking/03_05.jpg" caption="TCP 发送缓存和接收缓存" title="TCP 发送缓存和接收缓存" >}}

### TCP 报文段结构

{{< image src="/images/computer_networking/03_06.jpg" caption="TCP 报文段结构" title="TCP 报文段结构" >}}

TCP 报文段由首部字段和一个数据字段组成。数据字段包含一块应用数据。

TCP 报文段首部包含下列字段:

- **源端口号**和**目的端口号**：被用于多路复用/分解来自或送到上层应用的数据

- 序号和确认号：32 比特的**序号字段**(sequence number field)和 32 比特的**确认号字段**(acknowledgment number field)。这些字段被 TCP 发送方和接收方用来实现可靠数据传输服务

- 首部长度字段：4 比特的首部长度字段(header length field)，该字段指示了以 32 比特的字为单位的 TCP 首部长度。由于 TCP 选项字段的原因，TCP 首部的长度是可变的。(通常，选项字段为空，所以 TCP 首部的典型长度是 20 字节。)

- 标志位字段（URG, ACK, PSH, RST, SYN, FIN）：6 比特的**标志字段**(flag field)。**ACK 比特**用于指示确认字段中的值是有效的，即该报文段包括一个对已被成功接收报文段的确认。**RST、SYN和FIN比特**用于连接建立和拆除，在明确拥塞通告中使用了 **CWR 和 ECE 比特**，当 **PSH 比特**被置位时，就指示接收方应立即将数据交给上层，URG 比特用来指示报文段里存在着被发送端的上层实体置为“紧急”的数据。

- 接收窗口字段：16 比特的接收窗口字段(receive window field),该字段用于流量控制。用于指示接收方愿意接受的字节数量。

- 检验和字段：同 UDP 一样，TCP 数据首部也包括**检验和字段**(checksum field)。

- 紧急数据指针字段：紧急数据的最后一个字节由 16 比特的**紧急数据指针字段**(urgent data pointer field)指出。

- 可选与变长的**选项字段**(options field)：该字段用于发送方与接收方协商最大报文段长度(MSS)时，或在高速网络环境下用作窗口调节因子时使用。

#### 序号和确认号

序号字段和确认号字段是 TCP 可靠传输服务的关键部分。

TCP 把数据看成一个无结构的、有序的字节流。一个**报文段的序号**(sequence number for a segment)是该报文段首字节的字节流编号。

{{< image src="/images/computer_networking/03_07.jpg" caption="文件数据划分成 TCP 报文段" title="文件数据划分成 TCP 报文段" >}}

主机 A 填充进报文段的**确认号**是主机 A 期望从主机 B 收到的下一字节的序号。

{{< image src="/images/computer_networking/03_08.jpg" caption="TCP 确认号和序号" title="TCP 确认号和序号" >}}

### 可靠数据传输

TCP 在 IP 不可靠的尽力而为服务之上创建了一种**可靠数据传输服务**（reliable data transfer service）。

- 基于回退 N 步 (GBN) 和选择重传 (SR) 的混合机制

- 超时重传/快速重传

#### 超时重传

TCP 采用超时/重传机制来处理报文段的丢失问题。

定时器超时后，重传具有最小序号但仍未应答的报文段。

##### 往返时间的估计与超时时间

在任意时刻，TCP 仅为一个已发送的但目前尚未被确认的报文段估计 SampleRTT (样本RTT)

- 计算 EstimatedRTT：一旦获得一个新 SampleRTT 时，TCP 就会根据下列公式来更新 EstimatedRTT (指数加权移动平均 RTT)，在［RFC 6298］中 α 推荐值是 `α = 0.125`
    $$
    EstimatedRTT = (1 - α) * EstimatedRTT + α * SampleRTT
    $$


- 计算 DevRTT：DevRTT 用于估算 SampleRTT 一般会偏离 EstimatedRTT 的程度，β 的推荐值为 0.25
    $$
    DevRTT = (1 - β) * DevRTT + β * | SampleRTT - EstimatedRTT |
    $$


- 计算 TimeoutInterval：TimeoutInterval (超时间隔) 应该大于等于 EstimatedRTT,否则，将造成不必要的重传。也不应该比 EstimatedRTT 大太多，否则当报文段丢失时，TCP 不能很快地重传该报文段，导致数据传输时延大
    $$
    TimeoutInterval = EstimatedRTT + 4 * DevRTT
    $$

{{< image src="/images/computer_networking/03_09.jpg" caption="RTT 样本和 RTT 估计" title="RTT 样本和 RTT 估计" >}}

##### 定时器管理策略

使用以下伪代码描述定时器管理策略。

``` TCP发送方高度简化的描述
/* 假设发送方不受 TCP 流量和拥塞控制的限制，来自上层数据的长度小于 MSS,且数据传送只在一个方向进行。 */

NextSeqNum = InitialSeqNumber
SendBase = InitialSeqNumber

loop (forever) {
    switch (event) {
        event: 从上层应用程序接收到数据
            生成序号 NextSeqNum 的 TCP 报文段
            if (定时器当前没有运行) {
                启动定时器
            }
            向 IP 传递报文段
            NextSeqNum = NextSeqNum + length(data)
            break

        event: 定时器超时
            重传具有最小序号但仍未应答的报文段
            启动定时器
            break

        event: 收到 ACK，具有 ACK 字段值 y
            if (y > SendBase) {
                SendBase = y
                if (当前有未被确认的报文段) {
                    启动定时器
                }
            }
            else { /* 对已经确认的报文段的一个冗余 ACK */
                对 y 收到的冗余 ACK数加 1
                if (对 y 收到的冗余 ACK 数 == 3) {
                    重新发送具有序号 y 的报文段     /* TCP快速重传 */
                }
            }
            break
    }
}
```

##### 超时重传的几种情况

###### 由于确认丢失而重传

- 主机 A 向主机 B 发送一个报文段

- 主机 B 发往主机 A 的确认报文丢失

- 超时事件发生，主机 A 重传相同的报文段

{{< image src="/images/computer_networking/03_10.jpg" caption="由于确认丢失而重传" title="由于确认丢失而重传" >}}

###### 超时重传第一个报文段

- 主机 A 连续发两个报文段

- 主机 B 为每一个报文段分别发送一个确认。在超时之前这两个报文段中没有一个确认报文到达主机 A

- 超时事件发生，主机 A 重传第一个报文段，并重启定时器

- 只要第二个报文段的 ACK 在新的超时发生以前到达，则第二个报文段将不会被重传

{{< image src="/images/computer_networking/03_11.jpg" caption="超时重传第一个报文段" title="超时重传第一个报文段" >}}

###### 累积确认避免第一个报文段重传

- 主机 A 连续发两个报文段

- 主机 B 为每一个报文段分别发送一个确认。第一个报文段的确认报文丢失，在超时事件发生之前主机 A 收到第二个报文段的确认报文

- 主机A不会重传这两个报文段中的任何一个

{{< image src="/images/computer_networking/03_12.jpg" caption="累积确认避免第一个报文段重传" title="累积确认避免第一个报文段重传" >}}

#### 快速重传

超时触发重传存在的问题之一是超时周期可能相对较长。当一个报文段丢失时，会增加端到端时延。

发送方通常可在超时事件发生之前通过注意冗余 ACK 来较好地检测到丢包情况。**冗余 ACK**（duplicate ACK）就是再次确认某个报文段的 ACK，而发送方先前已经收到对该报文段的确认。

如果 TCP 发送方接收到对相同数据的3个冗余 ACK，它把这当作一种指示，说明跟在这个已被确认过 3 次的报文段之后的报文段已经丢失。一旦收到 3 个冗余 ACK, TCP 就执行快速重传（fast retransmit），即在该报文段的定时器过期之前重传丢失的报文段。

{{< image src="/images/computer_networking/03_13.jpg" caption="快速重传：在某报文段的定时器过期之前重传丢失的报文段" title="快速重传：在某报文段的定时器过期之前重传丢失的报文段" >}}

### 流量控制

TCP 为它的应用程序提供了**流量控制服务**(flow control service)以消除发送方使接收方缓存溢岀的可能性。

流量控制是一个速度匹配服务，即发送方的发送速率与接收方应用程序的读取速率相匹配。

TCP 通过让发送方维护一个称为**接收窗口**(receive window)的变量来提供流量控制。

假设主机 A 通过一条 TCP 连接向主机 B 发送一个大文件，定义以下变量：

- RcvBuffer：主机 B 接收缓存的大小

- LastByteRead：主机 B 上的应用进程从缓存读出的数据流的最后一个字节的编号

- LastByteRcvd：从网络中到达的并且已放入主机 B 接收缓存中的数据流的最后一个字节的编号

由于 TCP 不允许已分配的缓存溢出，下式必须成立:

$$
    LastByteRcvd - LastByteRead <= RcvBuffer
$$

接收窗口用 rwnd 表示，根据缓存可用空间的数量来设置：

$$
    rwnd = RcvBuffer - ( LastByteRcvd - LastByteRead )
$$

{{< image src="/images/computer_networking/03_14.jpg" caption="接收窗口（rwnd）和接收缓存（RcvBuffer）" title="接收窗口（rwnd）和接收缓存（RcvBuffer）" >}}

主机 B 通过把当前的 rwnd 值放入它发给主机 A 的报文段接收窗口字段中，通知主机 A 它在该连接的缓存中还有多少可用空间。

使用变量 rwnd 来提供流量控制服务的过程：

- 开始时，主机 B 设定 `rwnd = RcvBuffer`

- 主机 A 轮流跟踪两个变量，LastByteSent 和 LastByteAcked，LastByteSent - LastByteAcked 就是主机 A 发送到连接中但未被确认的数据量

- 通过将未确认的数据量控制在值 rwnd 以内，就可以保证主机 A 不会使主机B的接收缓存溢出，即主机 A 在该连接的整个生命周期须保证：`LastByteSent - LastByteAcked <= rwnd`

为了防止主机 A 接收到 rwnd 为 0 的报文被阻塞而不能再发送数据，TCP规范中要求：当主机 B 的接收窗口为 0 时，主机 A 继续发送只有一个字节数据的报文段。这些报文段将会被接收方确认。最终缓存将开始清空，并且确认报文里将包含一个非 0 的 rwnd 值。

### 连接管理

#### 连接建立：三次握手

目标：客户端与服务器相互通知并协商初始序列号，创建连接并分配缓存和变量。

步骤如下：

- 步骤 1：客户端发送 SYN 报文段 (随机选择序列号为 client_isn)

- 步骤 2：服务器端接收到 SYN 报文段，为 TCP 连接分配 TCP 缓存和变量。服务器端发送 SYNACK 报文段 (随机生成序列号为 server_isn，确认号为 client_isn+1)

- 步骤 3：客户端接收到 SYNACK 报文段，为 TCP 连接分配 TCP 缓存和变量。客户端发送 ACK 报文段 (确认号为 server_isn+1，可携带应用层数据)

{{< image src="/images/computer_networking/03_15.jpg" caption="TCP 三次握手" title="TCP 三次握手" >}}


#### 连接终止：四次挥手

{{< image src="/images/computer_networking/03_16.jpg" caption="TCP 四次挥手" title="TCP 四次挥手" >}}

客户端 TIME_WAIT 状态的作用：

- 确保最后一个 ACK 可靠到达。如果最后一个 ACK 丢失，被动关闭方会重传 FIN，TIME_WAIT 状态允许重新发送最后一个 ACK，防止被动关闭方一直处于 LAST_ACK 状态

- 让旧的重复报文段在网络中消失。防止旧连接数据混淆新连接，2MSL 时间确保所有旧的报文段都在网络中消失，避免相同四元组（源IP、源端口、目的IP、目的端口）的新连接收到旧连接的延迟报文

#### TCP 状态

{{< image src="/images/computer_networking/03_17.jpg" caption="客户端 TCP 状态" title="客户端 TCP 状态" >}}

{{< image src="/images/computer_networking/03_18.jpg" caption="服务端 TCP 状态" title="服务端 TCP 状态" >}}

### 拥塞控制

#### 拥塞控制方法

1. 端到端拥塞控制。网络层没有为运输层拥塞控制提供显式支持。
    - TCP 报文段的丢失（通过超时或3次冗余确认而得知）。


2. 网络辅助的拥塞控制。路由器向发送方提供关于网络中拥塞状态的显式反馈信息。
    - 直接网络反馈。直接反馈信息可以由网络路由器发给发送方。这种方式的通知通常采用了一种阻塞分组(choke packet)的形式。
    - 路由器标记或更新从发送方流向接收方的分组中的某个字段来指示拥塞的产生。一旦收到一个标记的分组后，接收方就会向发送方通知该网络拥塞指示。

{{< image src="/images/computer_networking/03_19.jpg" caption="网络指示拥塞信息的两种反馈路径" title="网络指示拥塞信息的两种反馈路径" >}}

#### TCP 的拥塞控制

TCP 必须使用端到端拥塞控制而不是使用网络辅助的拥塞控制，因为 IP 层不向端系统提供显式的网络拥塞反馈。

TCP 的指导性原则：

- 一个丢失的报文段表意味着拥塞，因此当丢失报文段时应当降低 TCP 发送方的速率。

- 一个确认报文段指示该网络正在向接收方交付发送方的报文段，因此，当对先前未确认报文段的确认到达时，能够增加发送方的速率。

- 带宽探测。给定 ACK 指示源到目的地路径无拥塞，而丢包事件指示路径拥塞，TCP 调节其传输速率的策略是增加其速率以响应到达的 ACK,除非岀现丢包事件，此时才减小传输速率。

三个主要部分：慢启动、拥塞避免、快速恢复

##### 慢启动

- cwnd 初始值为 1 MSS（Maximum Segment Size）

- 每收到一个 ACK，cwnd 增加 1 MSS（指数级增长）

- 结束条件：发生超时 或 cwnd 达到慢启动阈值 (ssthresh)

{{< image src="/images/computer_networking/03_20.jpg" caption="TCP 慢启动" title="TCP 慢启动" >}}

##### 拥塞避免

- 进入条件：cwnd >= ssthresh，一旦进入拥塞避免状态，cwnd 的值大约是上次遇到拥塞时的值的一半。因此，TCP 无法每过一个 RTT 再将 cwnd 的值翻番。

- 每经过一个 RTT，cwnd 增加约 1 MSS（加法增长）

##### 快速恢复

快速恢复是 TCP 推荐的而非必需的构件。

- 进入条件：收到 3 个冗余 ACK

- ssthresh 设置为 cwnd/2，cwnd 设置为 ssthresh + 3 MSS

- 每收到一个冗余 ACK，cwnd 增加 1 MSS

- 收到对新数据的 ACK 时，将 cwnd 设置为 ssthresh，进入拥塞避免阶段

{{< image src="/images/computer_networking/03_21.jpg" caption="TCP拥塞控制的FSM描述" title="TCP 拥塞控制的 FSM 描述" >}}

---

