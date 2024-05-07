# 汇编语言 —— 使用BIOS进行键盘输入和磁盘读写


---

## INT 9 中断例程对键盘输入的处理

键盘输入将引发 9 号中断，BIOS 提供了 int 9 中断例程 。CPU 在 9 号中断发生后，执行 int 9 中断例程，从 60h 端又读出扫描码，并将其转化为相应的 ASCII 码或状态信息，存储在内存的指定空间(键盘缓冲区或状态字节)中。

一般的键盘输入，在 CPU 执行完 int 9 中断例程后，都放到了键盘缓冲区中。键盘缓冲区中有 16 个字单元，可以存储 15 个按键的扫描码和对应的 ASCII 码。

{{< admonition type=warning open=true >}}

在这里，仅在逻辑结构的基础上，讨论 BIOS 键盘缓冲区的读写问题。其实键盘缓冲区是用环形队列结构管理的内存区。

{{< /admonition >}}

下面通过 A、B、C、D、E、Shift_A、A 的输入过程，简要地看一下 int 9 中断例程对键盘输入的处理方法。

1. 初始状态下，没有键盘输入，键盘缓冲区空，此时没有任何元素。

    {{< image src="/images/assembly_language/17_01.jpg" caption="键盘缓冲区状态 1" title="键盘缓冲区状态 1" >}}

2. 按下 A 键，引发键盘中断：CPU 执行 int 9 中断例程，从 60h 端又读出 A 键的通码：然后检测状态字节，看看是否有 Shift、Ctrl 等切换键按下；发现没有切换键按下，则将 A 键的扫描码 1Eh 和对应的 ASCII 码 61h，写入键盘缓冲区。缓冲区的字单元中，高位字节存储扫描码，低位字节存储 ASCII 码。此时缓冲区中的内容如下：

    {{< image src="/images/assembly_language/17_02.jpg" caption="键盘缓冲区状态 2" title="键盘缓冲区状态 2" >}}

3. 按下 B、C、D、E 键后，缓冲区中的内容如下：

    {{< image src="/images/assembly_language/17_03.jpg" caption="键盘缓冲区状态 3" title="键盘缓冲区状态 3" >}}

4. 按下左 Shift 键，引发键盘中断：int 9 中断例程接收左 Shift 键的通码，设置 0040:17 处的状态字节的第 1 位为 1，表示左 Shift 键按下。

5. 按下 A 键，引发键盘中断：CPU 执行 int9 中断例程，从 60h 端又读出 A 键的通码；检测状态字节，看看是否有切换键按下;发现左 Shift 键被按下，则将 A 键的扫描码 1Eh 和 Shift_A 对应的 ASCII 码，即字母 “A” 的 ASCII 码 41h，写入键盘缓冲区。此时缓冲区中的内容如下：

    {{< image src="/images/assembly_language/17_04.jpg" caption="键盘缓冲区状态 5" title="键盘缓冲区状态 5" >}}

6. 松开左 Shift 键，引发键盘中断：int 9 中断例程接收左 Shift 键的断码，设置 0040:17 处的状态字节的第 1 位为 0，表示左 Shift 键松开。

7. 按下 A 键，引发键盘中断：CPU 执行 int 9 中断例程，从 60h 端又读出 A 键的通码：然后检测状态字节，看看是否有 Shift、Ctrl 等切换键按下；发现没有切换键按下，则将 A 键的扫描码 1Eh 和对应的 ASCII 码 61h，写入键盘缓冲区。缓冲区的字单元中，高位字节存储扫描码，低位字节存储 ASCII 码。此时缓冲区中的内容如下：

    {{< image src="/images/assembly_language/17_05.jpg" caption="键盘缓冲区状态 7" title="键盘缓冲区状态 7" >}}

---

## 使用 INT 16H 中断例程读取键盘缓冲区

BIOS 提供了 int 16h 中断例程供程序员调用。int 16h 中断例程中包含的一个最重要的功能是从键盘缓冲区中读取一个键盘输入，该功能的编号为 0。

下面的指令从键盘缓冲区中读取一个键盘输入，并且将其从缓冲区中删除：

``` text
mov ah, 0
int 16h
```

结果：(ah)=扫描码，(al)=ASCII码。

下面我们接着上一节中的键盘输入过程，看一下 int 16h 如何读取键盘缓冲区：

1. 执行 `mov ah, 0; int 16h` 后，缓冲区的内容如下：

    {{< image src="/images/assembly_language/17_06.jpg" caption="键盘缓冲区状态 8" title="键盘缓冲区状态 8" >}}

    ah 中的内容为 1Eh，al 中的内容为 61h。

2. 执行 6 次 `mov ah, 0; int 16h` 后，缓冲区空：

    {{< image src="/images/assembly_language/17_01.jpg" caption="键盘缓冲区状态 9" title="键盘缓冲区状态 9" >}}

    ah 中的内容为 1Eh，al 中的内容为 61h。

3. 执行 `mov ah, 0; int 16h` 后，int 16h 中断例程检测键盘缓冲区，发现缓冲区空，则循环等待，直到缓冲区中有数据。

4. 按下 A 键后，缓冲区中的内容如下：

    {{< image src="/images/assembly_language/17_02.jpg" caption="键盘缓冲区状态 10" title="键盘缓冲区状态 10" >}}

5. 循环等待的 int 16h 中断例程检测到键盘缓冲区中有数据，将其读出，缓冲区又为空：

    {{< image src="/images/assembly_language/17_01.jpg" caption="键盘缓冲区状态 11" title="键盘缓冲区状态 11" >}}

    ah 中的内容为 1Eh，al 中的内容为 61h。

从上面我们可以看出， int 16h 中断例程的 0 号功能，进行如下的工作：

1. 检测键盘缓冲区中是否有数据；
2. 没有则继续做第 1 步；
3. 读取缓冲区第一个字单元中的键盘输入；
4. 将读取的扫描码送入 ah，ASCII 码送入 al；
5. 将已读取的键盘输入从缓冲区中删除。

BIOS 的 int 9 中断例程和 int 16h 中断例程是一对相互配合的程序，int 9 中断例程向键盘缓冲区中写入，int 16h 中断例程从缓冲区中读出。它们写入和读出的时机不同，int 9 中断例程是在有键按下的时候向键盘缓冲区中写入数据；而 int 16h 中断例程是在应用程序对其进行调用的时候，将数据从键盘缓冲区中读出。

---

## 应用 INT 13H 中断例程对磁盘进行读写

以 3.5 英寸软盘为例：软盘分为上下两面，每面有 80 个磁道，每个磁道又分为 18 个扇区，每个扇区的大小为 512 个字节。

磁盘的实际访问由磁盘控制器进行，我们可以通过控制磁盘控制器来访问磁盘。只能以扇区为单位对磁盘进行读写。在读写扇区的时候，要给出面号、磁道号和扇区号。面号和磁道号从 0 开始，而扇区号从 1 开始。

### 从磁盘读取数据

BIOS 提供的访问磁盘的中断例程为 int 13h。读取 0 面 0 道 1 扇区的内容到 0:200 的程序如下所示：

``` text
mov ax, 0
mov es, ax
mov bx, 200h

mov al, 1
mov ch, 0
mov cl, 1
mov dl, 0
mov dh, 0
mov ah, 2
int 13h
```

入口参数：

- (ah)=int 13h 的功能号(2 表示读扇区)
- (al)=读取的扇区数
- (ch)=磁道号
- (cl)=扇区号
- (dh)=磁头号(对于软盘即面号，因为一个面用一个磁头来读写)
- (dl)=驱动器号（软驱从 0 开始，0：软驱A，1：软驱B；硬盘从 80h 开始，80h：硬盘C，81h：硬盘D）
- es:bx 指向接收从扇区读入数据的内存区

返回参数：

- 操作成功：(ah)=0，(al)=读入的扇区数
- 操作失败：(ah)=出错代码

### 从磁盘写入数据

将 0:200 中的内容写入 0 面 0 道 1 扇区的程序如下所示：

``` text
mov ax, 0
mov es, ax
mov bx, 200h

mov al, 1
mov ch, 0
mov cl, 1
mov dl, 0
mov dh, 0

mov ah, 3
int 13h
```

入口参数：

- (ah)=int 13h 的功能号(3 表示写扇区)
- (al)=写入的扇区数
- (ch)=磁道号
- (cl)=扇区号
- (dh)=磁头号(面)
- (dl)=驱动器号
- es:bx 指向将写入磁盘的数据

返回参数：

- 操作成功：(ah)=0，(al)=写入的扇区数
- 操作失败：(ah)=出错代码

---
