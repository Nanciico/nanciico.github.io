# 计算机网络自顶向下方法 —— 应用层


---

## 网络应用程序体系结构

{{< image src="/images/computer_networking/02_01.jpg" caption="客户-服务器体系结构及 P2P 体系结构" title="客户-服务器体系结构及 P2P 体系结构" >}}

### 客户-服务器体系结构

在客户-服务器体系结构（client-server architecture）中，有一个总是打开的主机称为服务器，它服务于来自许多其他称为客户的主机的请求。

具有客户-服务器体系结构的非常著名的应用程序包括 Web、FTP、Telnet 和电子邮件。

### 对等（P2P）体系结构

在一个 P2P 体系结构（P2P architecture）中，应用程序在间断连接的主机对之间使用直接通信，这些主机对被称为对等方。

许多目前流行的、流量密集型应用都是 P2P 体系结构的。这些应用包括文件共享（例如BitTorrent）对等方协助下载加速器（例如迅雷）、因特网电话和视频会议（例如Skype）。

某些应用具有混合的体系结构，它结合了客户-服务器和P2P的元素。例如，对于许多即时讯息应用而言，服务器被用于跟踪用户的IP地址，但用户到用户的报文在用户主机之间（无须通过中间服务器）直接发送。

---

## 进程通信

在两个不同端系统上的进程，通过跨越计算机网络交换报文（message）而相互通信。

进程通过一个称为套接字（socket）的软件接口向网络发送报文和从网络接收报文。套接字是建立网络应用程序的可编程接口，因此套接字也称为应用程序和网络之间的应用程序编程接口（Application Programming Interface, API）。

应用程序开发者可以控制套接字在应用层端的一切，但是对该套接字的运输层端几乎没有控制权。应用程序开发者对于运输层的控制仅限于：①选择运输层协议；②也许能设定几个运输层参数，如最大缓存和最大报文段长度等。一旦应用程序开发者选择了一个运输层协议（如果可供选择的话），则应用程序就建立在由该协议提供的运输层服务之上。

{{< image src="/images/computer_networking/02_02.jpg" caption="应用进程、套接字和下面的运输层协议" title="应用进程、套接字和下面的运输层协议" >}}

---

## 应用层协议

**应用层协议**（application-layer protocol）定义了运行在不同端系统上的应用程序进程如何相互传递报文。特别是应用层协议定义了：

- 交换的报文类型，例如请求报文和响应报文。
- 各种报文类型的语法，如报文中的各个字段及这些字段是如何描述的。
- 字段的语义，即这些字段中的信息的含义。
- 确定一个进程何时以及如何发送报文，对报文进行响应的规则。

### Web 和 HTTP

**Web 页面**（Webpage）（也叫文档）是由对象组成的。一个对象（object）只是一个文件，诸如一个 HTML 文件、一个 JPEG 图形、一个 Java 小程序或一个视频片段这样的文件,且它们可通过一个 URL 地址寻址。

Web 的应用层协议是**超文本传输协议**（HyperText Transfer Protocol, HTTP），它是 Web 的核心，HTTP 使用 TCP 作为它的支撑运输协议。因为 HTTP 服务器并不保存关于客户的任何信息，所以我们说 HTTP 是一个无状态协议(stateless protocol)。

#### 非持续连接和持续连接

在许多因特网应用程序中，客户和服务器在一个相当长的时间范围内通信，其中客户发出一系列请求并且服务器对每个请求进行响应。当这种客户-服务器的交互是经 TCP 进行的，应用程序的研制者就需要做一个重要决定，即每个请求/响应对是经一个单独的 TCP 连接发送，还是所有的请求及其响应经相同的 TCP 连接发送呢？采用前一种方法，该应用程序被称为使用**非持续连接**(non-persistent connection)；采用后一种方法，该应用程序被称为使用**持续连接**(persistent connection) 。

HTTP 在其默认方式下使用持续连接，HTTP 客户和服务器也能配置成使用非持续连接。

#### HTTP 请求报文

{{< image src="/images/computer_networking/02_03.jpg" caption="一个典型的 HTTP 请求报文" title="一个典型的 HTTP 请求报文" >}}

HTTP 请求报文的第一行叫作**请求行**（request line），请求行有 3 个字段：方法字段、URL 字段和 HTTP 版本字段。

请求行后继的行叫作**首部行**（header line）。通过包含 Connection: close 首部行，该浏览器告诉服务器不要麻烦地使用持续连接，它要求服务器在发送完被请求的对象后就关闭这条连接。

{{< image src="/images/computer_networking/02_04.jpg" caption="一个 HTTP 请求报文的通用格式" title="一个 HTTP 请求报文的通用格式" >}}

首部行（和附加的回车和换行）后有一个“实体体” （entity body） 使用 GET 方法时实体体为空，而使用 POST 方法时才使用该实体体。

#### HTTP 响应报文

{{< image src="/images/computer_networking/02_05.jpg" caption="一个 HTTP 响应报文" title="一个 HTTP 响应报文" >}}

这个响应报文有三个部分：一个初始**状态行**（status line） , 6 个**首部行**（headerline）,然后是**实体体**（entity body）。实体体部分是报文的主要部分，即它包含了所请求的对象本身。

状态行有 3 个字段：协议版本字段、状态码和相应状态信息。

服务器用 Connection: close 首部行告诉客户，发送完报文后将关闭该 TCP 连接。

Last-Modified: 首部行指示了对象创建或者最后修改的日期和时间。Last-Modified: 首部行对既可能在本地客户也可能在网络缓存服务器（也叫代理服务器）上的对象缓存来说非常重要。

Content-Length: 首部行指示了被发送对象中的字节数。Content-Type: 首部行指示了实体体中的对象是 HTML 文本。

{{< image src="/images/computer_networking/02_06.jpg" caption="一个 HTTP 响应报文的通用格式" title="一个 HTTP 响应报文的通用格式" >}}

#### 用户与服务器的交互：cookie

HTTP 服务器是无状态的。而一个 Web 站点通常希望能够识别用户，为此，HTTP 使用了 cookie。它允许站点对用户进行跟踪。目前大多数商务 Web 站点都使用了 cookie。

cookie 技术有 4 个组件：①在 HTTP 响应报文中的一个 cookie 首部行；②在 HTTP 请求报文中的一个 cookie 首部行；③在用户端系统中保留有一个 cookie 文件，并由用户的浏览器进行管理；④位于 Web 站点的一个后端数据库。

{{< image src="/images/computer_networking/02_07.jpg" caption="用 cookie 跟踪用户状态" title="用 cookie 跟踪用户状态" >}}

cookie 可以用于标识一个用户。用户首次访问一个站点时，可能需要提供一个用户标识（可能是名字）。在后继会话中，浏览器向服务器传递一个 cookie 首部，从而向该服务器标识了用户。因此 cookie 可以在无状态的 HTTP 之上建立一个用户会话层。

#### Web 缓存与条件 GET 方法

**Web 缓存器**（Web cache）也叫**代理服务器**（proxy server）,它是能够代表初始 Web 服务器来满足 HTTP 请求的网络实体。Web缓存器有自己的磁盘存储空间,并在存储空间中保存最近请求过的对象的副本。可以配置用户的浏览器，使得用户的所有 HTTP 请求首先指向 Web 缓存器。一旦某浏览器被配置，每个对某对象的浏览器请求首先被定向到该 Web 缓存器。

{{< image src="/images/computer_networking/02_08.jpg" caption="客户通过 Web 缓存器请求对象" title="客户通过 Web 缓存器请求对象" >}}

{{< image src="/images/computer_networking/02_09.jpg" caption="为机构网络添加一台缓存器" title="为机构网络添加一台缓存器" >}}

通过使用内容分发网络（Content Distribution Network, CDN） , Web 缓存器正在因特网中发挥着越来越重要的作用。CDN公司在因特网上安装了许多地理上分散的缓存器，因而使大量流量实现了本地化。

HTTP协议有一种机制，允许缓存器证实它的对象是最新的。这种机制就是**条件 GET** （conditional GET）方法。

如果：①请求报文使用 GET 方法；并且②请求报文中包含一个 “If-Modified-Since:” 首部行。那么，这个 HTTP 请求报文就是一个条件 GET 请求报文。

条件 GET 方法的操作方式：

1. 一个代理缓存器（proxy cache）代表一个请求浏览器，向某 Web 服务器发送一个请求报文。
2. 该 Web 服务器向缓存器发送具有被请求的对象的响应报文，包含 `Last-Modified: Wed, 9 Sep 2015 09:23:24` 首部行。
3. 缓存器在将对象转发到请求的浏览器的同时，也在本地缓存了该对象和最后修改日期。
4. 一个星期后，另一个用户经过该缓存器请求同一个对象，该对象仍在这个缓存器中。该缓存器通过发送一个条件 GET 执行最新检查。包含首部行：`If-modified-since: Wed, 9 Sep 2015 09:23:24`。
5. 假设该对象没有被修改过。Web 服务器向该缓存器发送一个状态码为 304 且不包含所请求的对象的响应报文，它告诉缓存器可以使用缓存的对象副本。

### 因特网中的电子邮件

因特网电子邮件系统有 3 个主要组成部分：**用户代理**（user agent）、**邮件服务器**（mail server）和**简单邮件传输协议**（Simple Mail Transfer Protocol, SMTP）。

{{< image src="/images/computer_networking/02_10.jpg" caption="因特网电子邮件系统的总体描述" title="因特网电子邮件系统的总体描述" >}}

SMTP 是因特网电子邮件中主要的应用层协议。它使用 TCP 可靠数据传输服务，从发送方的邮件服务器向接收方的邮件服务器发送邮件。

{{< image src="/images/computer_networking/02_11.jpg" caption="电子邮件协议及其通信实体" title="电子邮件协议及其通信实体" >}}

Bob 的用户代理不能使用 SMTP 得到报文，因为取报文是一个拉操作，而 SMTP 协议是一个推协议。目前有一些流行的邮件访问协议，包括**第三版的邮局协议**（Post
Office Protocol—Version 3 , POP3）、**因特网邮件访问协议**（Internet Mail Access Protocol,
IMAP）以及 HTTP。

### DNS：因特网的目录服务

识别主机有两种方式，通过主机名或者 IP 地址。**域名系统**（Domain Name System, DNS）的主要任务是进行主机名到 IP 地址转换的目录服务。

DNS 是：①一个由分层的 DNS 服务器（DNS server）实现的分布式数据库；②一个使得主机能够查询分布式数据库的应用层协议。

DNS 服务器通常是运行 BIND（Berkeley Internet Name Domain）软件的 UNIX 机器。DNS 协议运行在 UDP 之上，使用 53 号端口。

DNS 通常是由其他应用层协议所使用的，包括 HTTP、SMTP 和 FTP,将用户提供的主机名解析为 IP 地址。

大致有 3 种类型的DNS服务器：根 DNS 服务器、顶级域(Top-Level Domain, TLD) DNS 服务器和权威 DNS 服务器。这些服务器下图所示的层次结构组织起来。

{{< image src="/images/computer_networking/02_12.jpg" caption="部分DNS服务器的层次结构" title="部分DNS服务器的层次结构" >}}

- **根 DNS 服务器**。有400多个根名字服务器遍及全世界。根名字服务器提供 TLD 服务器的 IP 地址。
- **顶级域(DNS)服务器**。对于每个顶级域(如 com、org、net、edu 和 gov)和所有国家的顶级域(如 uk、fr、ca 和 jp),都有 TLD 服务器(或服务器集群)。TLD 服务器提供了权威 DNS 服务器的 IP 地址。
- **权威DNS服务器**。在因特网上具有公共可访问主机（如Web服务器和邮件服务器）的每个组织机构必须提供公共可访问的 DNS 记录，这些记录将这些主机的名字映射为 IP 地址。

另一类重要的 DNS 服务器，称为**本地 DNS 服务器**（local DNS server）。当主机发出 DNS 请求时，该请求被发往本地 DNS 服务器，它起着代理的作用，并将该请求转发到 DNS 服务器层次结构中。

{{< image src="/images/computer_networking/02_13.jpg" caption="各种 DNS 服务器的交互" title="各种 DNS 服务器的交互" >}}

### P2P 文件分发

BitTorrent 是一种用于文件分发的流行 P2P 协议。

参与一个特定文件分发的所有对等方的集合被称为一个**洪流**（torrent）。在一个洪流中的对等方彼此下载等长度的**文件块**（chunk），典型的块长度为 256KB。

每个洪流具有一个基础设施节点，称为**追踪器**（tracker）。当一个对等方加入某洪流时，它向追踪器注册自己，并周期性地通知追踪器它仍在该洪流中。以这种方式，追踪器跟踪参与在洪流中的对等方。

{{< image src="/images/computer_networking/02_14.jpg" caption="用 BitTorrent 分发文件" title="用 BitTorrent 分发文件" >}}

### 视频流和内容分发网

#### HTTP 流和 DASH

在 HTTP 流中，视频只是存储在 HTTP 服务器中作为一个普通的文件，每个文件有一个特定的 URL。

HTTP 流具有严重缺陷，即所有客户接收到相同编码的视频，这导致了一种新型基于 HTTP 的流的研发，它常常被称为**经 HTTP 的动态适应性流**(Dynamic Adaptive
Streaming over HTTP, DASH) 。在 DASH 中，视频编码为几个不同的版本，其中每个版本具有不同的比特率，对应于不同的质量水平。DASH 允许客户自由地在不同的质量等级之间切换。

#### 内容分发网

为了应对向分布于全世界的用户分发巨量视频数据的挑战，几乎所有主要的视频流公司都利用内容分发网（Content Distribution Network, CDN）。

CDN 管理分布在多个地理位置上的服务器，在它的服务器中存储视频（和其他类型的 Web 内容，包括文档、图片和音频）的副本，并且所有试图将每个用户请求定向到一个将提供最好的用户体验的 CDN 位置。

{{< image src="/images/computer_networking/02_15.jpg" caption="DNS 将用户的请求重定向到一台 CDN 服务器" title="DNS 将用户的请求重定向到一台 CDN 服务器" >}}

---

## 套接字编程：生成网络应用

在研发阶段，开发者必须最先做的一个决定是，应用程序是运行在 TCP 上还是运行在 UDP 上。

TCP 是面向连接的，并且为两个端系统之间的数据流动提供可靠的字节流通道。

UDP 是无连接的，从一个端系统向另一个端系统发送独立的数据分组，不对交付提供任何保证。

### UDP 套接字编程

{{< image src="/images/computer_networking/02_16.jpg" caption="使用 UDP 的客户 - 服务器应用程序" title="使用 UDP 的客户 - 服务器应用程序" >}}

``` UDPClient.py
from socket import *

server_name = 'localhost'
server_port = 12000

# 创建客户的套接字
# AF_INET 指示了底层网络使用了 IPv4
# SOCK_DGRAM 指示 UDP 套接字
# 操作系统指示端口号
client_socket = socket(AF_INET, SOCK_DGRAM)

message = input('Input lowercase sentence:')

# 首先将报文由字符串类型转换为字节类型
# 再向进程的套接字发送分组
client_socket.sendto(message.encode(), (server_name, server_port))

# 在发送分组之后，客户等待接收来自服务器的数据
# 缓存长度 2048
modified_message, server_address = client_socket.recvfrom(2048)

# 将报文从字节转化为字符串后，在用户显示器上打印
print(modified_message.decode())

# 关闭套接字
client_socket.close()
```

``` UDPServer.py
from socket import *

server_port = 12000
server_socket = socket(AF_INET, SOCK_DGRAM)

# 将端口号 12000 与该服务器的套接字绑定在一起
server_socket.bind(('', server_port))
print ('The server is ready to receive')

while True:
    message, client_address = server_socket.recvfrom(2048)
    modified_message = message.decode().upper()
    server_socket.sendto(modified_message.encode(), client_address)
```

### TCP 套接字编程

{{< image src="/images/computer_networking/02_17.jpg" caption="TCPServer 进程有两个套接字" title="TCPServer 进程有两个套接字" >}}

{{< image src="/images/computer_networking/02_18.jpg" caption="使用 TCP 的客户 - 服务器应用程序" title="使用 TCP 的客户 - 服务器应用程序" >}}

``` TCPClient.py
from socket import *

server_name = 'localhost'
server_port = 12000

# 参数指示该套接字是 SOCK_STREAM 类型。这表明它是一个 TCP 套接字
client_socket = socket(AF_INET, SOCK_STREAM)

# 执行三次握手，并在客户和服务器之间创建起一条 TCP 连接
client_socket.connect((server_name, server_port))

sentence = input('Input lowercase sentence:')
client_socket.send(sentence.encode())

modified_sentence = client_socket.recv(1024)
print('From Server: ', modified_sentence.decode())

# 引起客户中的 TCP 向服务器中的 TCP 发送一条 TCP 报文
client_socket.close()
```

``` TCPServer.py
from socket import *

server_port = 12000

server_socket = socket(AF_INET, SOCK_STREAM)

# 将服务器的端口号与套接字关联
server_socket.bind(('', server_port))

# 欢迎套接字，等待并聆听某个客户敲门
# 参数定义了请求连接的最大数
server_socket.listen(1)

print('The server is ready to receive')

while True:
    # server_socket 调用 accept() 方法
    # 创建 connection_socket 新套接字
    # 客户和服务器则完成了握手
    # 创建了一个 TCP 连接
    connection_socket, addr = server_socket.accept()
    sentence = connection_socket.recv(1024).decode()
    capitalized_sentence = sentence.upper()
    connection_socket.send(capitalized_sentence.encode())
    connection_socket.close()
```

---

