# 计算机网络自顶向下方法 —— 网络层：数据平面


---

{{< image src="/images/computer_networking/04_01.jpg" caption="网络层" title="网络层" >}}

网络层能够被分解为两个相互作用的部分，即数据平面和控制平面。

网络层数据平面负责实现 “每台路由器” 的转发功能，即数据包如何通过单个网络设备。

## 网络层概述

### 网络层的两大核心功能

- 转发（数据平面）： 将单个数据包从路由器的输入链路移动到适当的输出链路。

    - 动作：查表、交换

    - 时间尺度：纳秒级

    - 实现：通常是硬件（ASIC）

- 路由（控制平面）： 确定数据包从源到目的端所经过的端到端路径。

    - 动作：计算、决策

    - 时间尺度：秒级

    - 实现：通常是软件（路由协议）

### 数据平面与控制平面的合作方式

- 传统方式： 路由器和控制平面功能在同一设备中紧密耦合。路由选择算法决定转发表中的值。

    {{< image src="/images/computer_networking/04_02.jpg" caption="路由选择算法决定转发表中的值" title="路由选择算法决定转发表中的值" >}}

- SDN（软件定义网络）方式： 控制平面功能从网络设备中分离出来，由一个远程的控制器（如SDN控制器）负责。数据平面仍然在路由器中。

    {{< image src="/images/computer_networking/04_03.jpg" caption="远程控制器确定并分发转发表中的值" title="远程控制器确定并分发转发表中的值" >}}

### 网络服务模型

因特网的网络层提供了单一的服务，称为**尽力而为服务**（best-effort service）。使用尽力而为服务，传送的分组既不能保证以它们发送的顺序被接收，也不能保证它们最终交付；既不能保证端到端时延，也不能保证有最小的带宽。

## 路由器工作原理

### 路由器四个核心组件

{{< image src="/images/computer_networking/04_04.jpg" caption="路由器体系结构" title="路由器体系结构" >}}

- 输入端口

    {{< image src="/images/computer_networking/04_05.jpg" caption="输入端口处理" title="输入端口处理" >}}

    - 功能：线路端接、数据链路处理（协议，拆封）、查找，转发，排队。
    - 关键：**转发表**，根据数据包目的地址查找输出端口。

- 交换结构

    {{< image src="/images/computer_networking/04_06.jpg" caption="三种交换技术" title="三种交换技术" >}}

    - 连接输入和输出端口，有三种类型：经内存交换、经总线交换、经互联网络交换

- 输出端口

    {{< image src="/images/computer_networking/04_07.jpg" caption="输出端口处理" title="输出端口处理" >}}

    - 功能：从交换结构接收数据包，数据链路处理（协议，拆封）、线路端接。
    - 包含**队列管理**。

- 路由选择处理器

    - 执行控制平面功能（路由协议、维护路由表、计算转发表）。
    - 在SDN中，它与远程控制器通信。

### 分组排队与丢包

在输入端口和输出端口处都可以形成分组队列，当队列满时，新到达的数据包将被丢弃（丢包）。

- 输入排队：如果交换结构速度慢于输入端口总和，会在输入端口产生排队。

    - 线路前部阻塞（Head-Of-the-Line，HOL）：一个队列头部的数据包阻塞了队列中后续数据包的转发。

        {{< image src="/images/computer_networking/04_08.jpg" caption="在一个输入排队交换机中的HOL阻塞" title="在一个输入排队交换机中的HOL阻塞" >}}

- 输出排队：当交换结构速度大于单个输出端口速度时，会在输出端口产生排队。

    {{< image src="/images/computer_networking/04_09.jpg" caption="输出端口排队" title="输出端口排队" >}}

### 分组调度策略

分组调度策略决定输出端口排队的分组如何经输出链路传输。

- FIFO（先进先出）

    {{< image src="/images/computer_networking/04_10.jpg" caption="FIFO排队抽象" title="FIFO排队抽象" >}}

- 优先级排队

    {{< image src="/images/computer_networking/04_11.jpg" caption="优先权排队模型" title="优先权排队模型" >}}

- 加权公平排队（Weighted Fair Queueing，WFQ）

    {{< image src="/images/computer_networking/04_12.jpg" caption="加权公平排队" title="加权公平排队" >}}

---

## 网际协议（IP）

### IPv4数据报格式

{{< image src="/images/computer_networking/04_13.jpg" caption="IPv4数据报格式" title="IPv4数据报格式" >}}

- 版本、首部长度、服务类型

- 数据报长度：IP数据报的总长度（首部加上数据）

- 标识、标志、片偏移：与IP分片有关

- 寿命：（Time-To-Live, TTL）防止数据包无限循环。每当一台路由器处理数据报时，该字段的值减1，若TTL字段减为0,则该数据报必须丢弃。

- 协议：指示应将数据部分交给哪个运输层协议（如TCP=6，UDP=17）。

- 首部检验和

- 源和目的IP地址

- 选项（如果有）

- 数据（有效载荷）

### IPv4编址

- IP地址：32位标识符，与接口相关联。主机与物理链路之间的边界叫作接口（interface）。

    {{< image src="/images/computer_networking/04_14.jpg" caption="接口地址和子网" title="接口地址和子网" >}}

- 子网（Subnet）：可以从物理上**不经路由器**而直接彼此联系的一组接口。

    - 子网掩码（Subnet Mask）： 用来区分IP地址中的网络（子网）部分和主机部分。

    - CIDR（无类别域间路由）： a.b.c.d/x，其中x是地址中网络部分的位数，。

    {{< image src="/images/computer_networking/04_15.jpg" caption="子网地址" title="子网地址" >}}

    {{< image src="/images/computer_networking/04_16.jpg" caption="3台路由器互联6个子网" title="3台路由器互联6个子网" >}}

    - IP广播地址255.255.255.255，当一台主机发出一个目的地址为IP广播地址的数据报时，该报文会交付给同一个网络中的所有主机。

- 获取IP地址：动态主机配置协议（DHCP）

    - 功能：动态分配IP地址、子网掩码、第一跳路由器地址（默认网关）、DNS服务器。
    
    {{< image src="/images/computer_networking/04_17.jpg" caption="DHCP客户和服务器" title="DHCP客户和服务器" >}}

    - 工作过程（四步）：DHCP服务器发现、DHCP服务器提供、DHCP请求、DHCP ACK。

    {{< image src="/images/computer_networking/04_18.jpg" caption="DHCP客户-服务器交互" title="DHCP客户-服务器交互" >}}

    - DHCP缺陷：当一个移动节点在子网之间移动时，不能维持与远程应用之间的TCP连接。**移动IP**是一种对IP基础设施的扩展,允许移动节点在网络之间移动时使用其单一永久的地址。

### 网络地址转换（NAT）

- 动机：解决IPv4地址短缺问题，提高隐私和安全性。

- 工作原理：

    - 所有离开本地网络的数据包，将其（源IP，端口）替换为（NAT IP，新端口）。

    - 在NAT转换表中记录该映射。

    - 将进入的数据包根据转换表进行反向转换。

    {{< image src="/images/computer_networking/04_19.jpg" caption="网络地址转换" title="网络地址转换" >}}

### IPv6

- 动机：IPv4地址耗尽、简化首部以加快处理。

- IPv6数据报格式

    - 重要变化
    
        - 扩大的地址容量IP地址长度从32比特增加到128比特

        - 固定40字节首部，字段更少

        - 去除分片、首部检验和等字段，增加流标签字段

    {{< image src="/images/computer_networking/04_20.jpg" caption="IPv6数据报格式" title="IPv6数据报格式" >}}

    - 版本、流量类型、流标签

    - 有效载荷长度、下一个首部、跳限制

    - 源地址和目的地址

    - 数据

- 从IPv4向IPv6迁移

    建隧道，在IPv4路由器之间传输IPv6数据报。

    {{< image src="/images/computer_networking/04_21.jpg" caption="建隧道" title="建隧道" >}}

---

## 通用转发和SDN


