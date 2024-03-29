# 汇编语言 —— 寄存器(内存访问)


---

## 内存中字的存储

CPU 中，用 16 位寄存器来存储一个字。高 8 位存放高位字节，低 8 位存放低位字节。

字单元：即存放一个字型数据(16 位)的内存单元，由两个地址连续的内存单元组成。高地址内存单元中存放字型数据的高位字节，低地址内存单元中存放字型数据的低位字节。

---

## DS 和 [address]

8086CPU 中有一个 DS 寄存器，通常用来存放要访问数据的段地址。

比如我们需要读取 10000H 单元的内容，并将数据读到 al 中。代码如下：

``` text
; 8086CPU 不支持将数据直接送入段寄存器的操作（硬件设计问题）
; 需要使用一个寄存器进行中转
mov bx,1000H
mov ds,bx

; 完成数据从 1000:0 单元到 al 的传送
; 指令执行时，8086CPU 自动取 ds 中的数据为内存单元的段地址
mov al,[0]
```

---

## 字的传送

只要在 mov 指令中给出 16 位的寄存器就可以进行 16 位数据的传送：

``` text
mov bx,1000H
mov ds,bx
mov ax,[0]      ; 1000:0 处的字型数据送入 ax
mov [0],cx      ; cx 中的 16 位数据送到 1000:0 处
```

---

## mov、add、sub 指令

mov、add、sub 指令可以有以下几种形式（以 mov 指令举例）：

| 指令形式 | 指令案例 |
| :--: | :--: |
| mov 寄存器，数据 | `mov ax,8` |
| mov 寄存器，寄存器 | `mov ax,bx` |
| mov 寄存器，内存单元 | `mov ax,[0]` |
| mov 内存单元，寄存器 | `mov [0],ax` |
| mov 段寄存器，寄存器 | `mov ds,ax` |
| mov 寄存器，段寄存器 | `mov ax,ds` |
| mov 段寄存器，内存单元 | `mov cs,[0]` |
| mov 内存单元，段寄存器 | `mov [0],cs` |

---

## 数据段

我们可以将一组长度为 N( ≤ 64 KB )、地址连续、起始地址为 16 的倍数的内存单元当作专门存储数据的内存空间，从而定义了一个数据段。

将一段内存当作数据段，也是我们在编程时的一种安排，与代码段类似。

---

## CPU 提供的栈机制

栈是一种具有特殊的访问方式的存储空间。最后进入这个空间的数据，最先出去。

8086CPU 提供相关的指令来以栈的方式访问内存空间。在基于 8086CPU 编程的时候，可以将一段内存当作栈来使用。

8086CPU 提供入栈和出栈指令，最基本的两个是 PUSH(入栈) 和 POP(出栈)。8086CPU 的入栈和出栈操作以字为单位进行的。

8086CPU 中，有两个寄存器，段寄存器 SS 和寄存器 SP，栈顶的段地址存放 在 SS 中，偏移地址存放在 SP 中。任意时刻，SS:SP 指向栈顶元素。push 指令和 pop 指令执行时，CPU 从 SS 和 SP 中得到栈顶的地址。

### PUSH 操作

push ax 的执行，由以下两步完成：

1. SP = SP - 2，SS:SP指向当前栈顶前面的单元，以当前栈顶前面的单元为新的栈顶；
2. 将 ax 中的内容送入 SS:SP 指向的内存单元处，SS:SP 此时指向新栈顶。

{{< image src="/images/assembly_language/03_01.jpg" caption="PUSH 指令执行过程" title="PUSH 指令执行过程" >}}

### POP 操作

pop ax 的执行过程和 push ax 相反，由以下两步完成：

1. 将 SS:SP 指向的内存单元处的数据送入 ax 中；
2. SP = SP + 2，SS:SP 指向当前栈顶下面的单元，以当前栈顶下面的单元为新的栈顶。

{{< image src="/images/assembly_language/03_02.jpg" caption="POP 指令执行过程" title="POP 指令执行过程" >}}

### 栈空时 SS:SP 的指向

任意时刻，SS:SP 指向栈顶元素，当栈为空的时候，栈中没有元素，也就不存在栈顶元素，所以 SS:SP 只能指向栈的最底部单元下面的单元，该单元的偏移地为栈最底部的字单元的偏移地址 + 2。

{{< image src="/images/assembly_language/03_03.jpg" caption="栈空的状态" title="栈空的状态" >}}

### 栈顶超界问题

当栈满的时候 再使用 push 指令入栈，或栈空的时候再使用 pop 指令出栈，都将发生栈顶超界问题。

8086CPU 不保证我们对栈的操作不会超界，在编程的时候要自己操心栈顶超界的问题。

---

## PUSH、POP 指令

| 指令形式 | 指令案例 | 指令说明 |
| :--: | :--: | :--: |
| push 寄存器 | `push ax` | 将一个奇存器中的数据入栈 |
| pop 寄存器 | `pop ax` | 出栈，用一个寄存器接收出栈的数据 |
| push 段寄存器 | `push ds` | 将一个段寄存器中的数据入栈 |
| pop 段寄存器 | `pop cs` | 出栈，用一个段寄存器接收出栈的数据 |
| push 内存单元 | `push [0]` | 将一个内存字单元处的字入栈(注意:栈操作都是以字为单位) |
| pop 内存单元 | `pop [0]` | 出栈，用 一个内存字单元按收出栈的数据 |

push、pop 实质上就是一种内存传送指令，可以在寄存器和内存之间或内存和内存之间传送数据，与 mov 指令不同的是，push 和 pop 指令访问的内存单元的地址不是在指令中给出的，而是由 SS:SP 指出的。

CPU 执行 mov 指令只需一步操作，就是传送；而执行 push、pop 指令需要两步操作：

- 执行push时，CPU 的两步操作是：先改变 SP，后向 SS:SP 处传送
- 执行 pop 时，CPU 的两步操作是：先读取 SS:SP 处的数据，后改变SP

提供：SS、SP 指示栈顶；改变 SP 后写内存的入栈指令；读内存后改变 SP 的出栈指令。这就是 8086CPU 提供的栈操作机制。

---

## 段的综述

我们可以将一段内存定义为一个段，用一个段地址指示段，用偏移地址访问段内的单元。这完全是我们自己的安排：

1. 我们可以用一个段存放数据，将它定义为“数据段”；
2. 我们可以用一个段存放代码，将它定义为“代码段”；
3. 我们可以用一个段当作栈，将它定义为“栈段”。

对于数据段，将它的段地址放在 DS 中，用 mov、add、sub 等访问内存单元的指令时，CPU 就将我们定义的数据段中的内容当作数据来访问。

对于代码段，将它的段地址放在 CS 中，将段中第一条指令的偏移地址放在 IP 中，这样 CPU 就将执行我们定义的代码段中的指令。

对于栈段，将它的段地址放在 SS 中，将栈顶单元的偏移地址放在 SP 中，这样 CPU 在需要进行栈操作的时候，比如执行 push、pop 指令等，就将我们定义的栈段当作栈空间来用。

**一段内存，可以既是代码的存储空间，又是数据的存储空间，还可以是栈空间，也可以什么也不是。关键在于CPU中寄存器的设置，即 CS、IP，SS、SP，DS 的指向。**

---

