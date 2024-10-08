# 汇编语言 —— 寄存器


CPU 由运算器、控制器、寄存器等器件构成，这些器件靠内部总线相连。内部总线实现 CPU 内部各个器件之间的联系，外部总线实现 CPU 和主板上其他器件的联系。

在 CPU 中：

- 运算器进行信息处理；
- 寄存器进行信息存储；
- 控制器控制各种器件进行工作；
- 内部总线连接各种器件，在它们之间进行数据的传送。

---

## 通用寄存器

8086CPU 的所有寄存器都是 16 位的，可以存放两个字节。AX、BX、CX、DX 这 4 个寄存器通常用来存放一般性的数据，被称为通用寄存器。

8086CPU 的 AX、BX、CX 、DX 这 4 个寄存器都可分为两个可独立使用的 8 位寄存器来使用。

---

## 字在寄存器中的存储

字节：记为 byte，一个字节由 8 个 bit 组成，可以存在 8 位寄存器中。

字：记为 word，一个字由两个字节组成，这两个字节分别称为这个字的高位字节和低位字节。

``` text
AX 寄存器：
   高位字节 AH       低位字节 AL
0 1 0 0 1 1 1 0 | 0 0 1 0 0 0 0 0
```

---

## 几条汇编指令

| 汇编指令 | 控制 CP U 完成的操作 | 用高级语言的语法描述 |
| :--: | :--: | :--: |
| mov ax,18 | 将 18 送入寄存器 AX | AX = 18 |
| mov ah,78 | 将 78 送入寄存器 AH | AH = 78 |
| add ax,8 | 将寄存器 AX 中的数值加上 8 | AX = AX + 8 |
| mov ax,bx | 将寄存器 BX 中的数据送入寄存器 AX | AX = BX |
| add ax,bx | 将 AX 和 BX 中的数值相加，结果存在 AX 中 | AX = AX + BX |

---

## 8086CPU 给出物理地址的方法

### 物理地址的概念

CPU 访问内存单元时，要给出内存单元的地址。所有的内存单元构成的存储空间是一个一维的线性空间，每一个内存单元在这个空间中都有唯一的地址，我们将这个唯一的地址称为物理地址。

CPU 通过地址总线送入存储器的，必须是一个内存单元的物理地址。在 CPU 向地址总线上发出物理地址之前，必须要在内部先形成这个物理地址。不同的 CPU 可以有不同的形成物理地址的方式。

### 8086CPU 的结构

8086CPU 是 16 位结构的 CPU：

- 运算器一次最多可以处理 16 位的数据；
- 寄存器的最大宽度为 16 位；
- 寄存器和运算器之间的通路为 16 位。

内存单元的地址在送上地址总线之前，必须在 CPU 中处理、传输、暂时存放，对于 16 位 CPU，能一次性处理、传输、暂时存储 16 位的地址。

8086CPU 有 20 位地址总线，可以传送 20 位地址，达到 1MB 寻址能力。

### 8086CPU 生成物理地址的过程

8086CPU 采用一种在内部用两个 16 位地址合成的方法来形成一个 20 位的物理地址:

1. CPU 中的相关部件提供两个 16 位的地址，一个称为段地址，另一个称为偏移地址；
2. 段地址和偏移地址通过内部总线送入一个称为地址加法器的部件；
3. 地址加法器将两个 16 位地址合成为一个 20 位的物理地址；
4. 地址加法器通过内部总线将 20 位物理地址送入输入输出控制电路；
5. 输入输出控制电路将 20 位物理地址送上地址总线；
6. 20 位物理地址被地址总线传送到存储器。

{{< image src="/images/assembly_language/02_01.jpg" caption="8086CPU 相关部件的逻辑结构" title="8086CPU 相关部件的逻辑结构" >}}

地址加法器采用 **物理地址 = 段地址 × 16 + 偏移地址** 的方法用段地址和偏移地址合成物理地址。

“段地址 × 16 + 偏移地址 = 物理地址” 的本质含义是：CPU 在访问内存时，用一个基础地址(段地址 × 16)和一个相对于基础地址的偏移地址相加，给出内存单元的物理地址。

内存并没有分段，段的划分来自于 CPU，由于 8086CPU 用 “基础地址(段地址 x 16) + 偏移地址 = 物理地址” 的方式给出内存单元的物理地址，使得我们可以用分段的方式来管理内存。

“基础地址 = 段地址 x 16”，因此段的起始地址也一定 16 的倍数；偏移地址为 16 位 ，16 位地址的寻址能力为 64KB ，所以一个段的长度最大为 64KB。

---

## CS 和 IP

段地址在 8086CPU 的段寄存器中存放。8086CPU 有 4 个段寄存器：CS、DS、SS、ES。

CS 和 IP 是8086CPU 中两个最关键的寄存器，它们指示了 CPU 当前要读取指令的地址。CS 为代码段寄存器，IP 为指令指针寄存器。

8086 机中，任意时刻，CPU 将 CS:IP 指向的内容当作指令执行。

{{< image src="/images/assembly_language/02_02.jpg" caption="8086PC 读取和执行指令的相关部件" title="8086PC 读取和执行指令的相关部件" >}}

8086CPU 读取、执行一条指令的详细过程：

1. CS、IP 中的内容送入地址加法器，地址加法器计算物理地址；
2. 地址加法器将物理地址送入输入输出控制电路；
3. 输入输出控制电路将物理地址送上地址总线；
4. 从物理地址对应的内存单元中获取机器指令，将机器指令通过数据总线被送入 CPU；
5. 输入输出控制电路将机器指令送入指令缓冲器；
6. 读取一条指令后，IP 中的值自动増加，值为所读取指令的长度，以使 CPU 可以读取下一条指令；
7. 执行控制器执行指令。转到步骤 1，重复这个过程。

8086CPU 读取、执行一条指令的简要过程描述：

1. 从 CS:IP 指向的内存单元读取指令，读取的指令进入指令缓冲器；
2. IP = IP + 所读取指令的长度，从而指向下一条指令；
3. 操作执行器执行指令。转到步骤 1，重复这个过程。

在 8086CPU 加电启动或复位后(即 CPU 刚开始工作时) CS 和 IP 被设置为 CS=FFFFH，IP=0000H，即在 8086PC 机刚启动时，CPU 从内存 FFFFOH 单元中读取指令执行，FFFFOH 单元中的指令是 8086PC 机开机后执行的第一条指令。

在内存中，指令和数据没有任何区别，都是二进制信息，CPU 在工作的时候把有的信息看作指令，有的信息看作数据。CPU 将 CS、IP 中的内容当作指令的段地址和偏移地址，用它们合成指令的物理地址，到内存中读取指令码，执行。

---

## 修改 CS、IP 的指令

程序员可以通过改变 CS、IP 中的内容来控制 CPU 执行目标指令。

mov 指令不能用于设置 CS、IP 的值，mov 指令被称为传送指令。

jmp 指令可以用于设置 CS、IP 的值。能够改变 CS、IP 的 内容的指令被统称为转移指令。

| jmp 指令用法 | jmp 指令案例 | jmp 指令功能 |
| :--: | :--: | :--: |
| jmp 段地址:偏移地址 | `jmp 2AE3:3` 执行后：CS=2AE3H，IP=0003H | 同时修改 CS、IP 的内容 |
| jmp 某一合法奇存器 | `jmp ax` 执行后：IP = AX | 用寄存器中的值修改 IP |

---

## 代码段

对于 8086PC 机，在编程时，可以根据需要，将一组内存单元定义为一个段。我们可以将长度为 N（N ≤ 64KB）的一组代码，存在一组地址连续、起始地址为 16 的 倍数的内存单元中。我们可以认为，这段内存是用来存放代码的，从而定义了一个代码段。

将一段内存当作代码段，仅仅是我们在编程时的一种安排，CPU 并不会由于这种安排，就自动地将我们定义的代码段中的指令当作指令来执行。

---

