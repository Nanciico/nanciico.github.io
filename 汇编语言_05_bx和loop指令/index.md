# 汇编语言 —— [BX]和loop指令


---

## [BX]

指令 `mov ax, [bx]` 的功能：bx 中存放的数据作为一个偏移地址 EA，段地址 SA 默认在 ds 中，将 SA:EA 处的数据送入 ax 中。即: (ax)=((ds)*16+(bx))。

指令 `mov [bx], ax` 的功能：bx 中存放的数据作为一个偏移地址 EA，段地址 SA 默认在 ds 中，将 ax 中的 数据送入内存 SA:EA 处。即: ((ds)*16+(6x))=(ax)。

---

## Loop 指令

CPU 执行 loop 指令的时候，要进行两步操作：

1. (cx) = (cx) - 1；
2. 判断 cx 中的值，不为零则转至标号执行程序，如果为零则向下执行。

通常我们用 loop 指令来实现循环功能，cx 中存放循环次数。

使用 cx 和 loop 指令配合实现循环功能的要点：

1. 在 cx 中存放循环次数；
2. loop 指令中的标号所标识的地址在前面；
3. 要循环执行的程序段，要写在标号和 loop 指令的中间。

用 cx 和 loop 指令相配合实现循环功能的程序框架如下：

``` text
    mov cx, 循环次数

s:
    循环执行的程序段
    loop s
```

标号 "s" 实际是一个地址，如果在执行 "loop s" 时，cx 减 1 后不为 0，指令 "loop s" 就会把 IP 设置为 "s" 指向的地址，从而使 CS:IP 指向循环执行的程序段。

---

## Loop 和 [bx] 的联合应用

{{< admonition type=question title="问题" open=true >}}

编程目标：计算 ffff:0~ffff:b 单元中的数据的和，结果存储在 dx 中。

{{< /admonition >}}

分析问题：

1. 运算后的结果是否会超出 dx 所能存储的范围?
2. 能否将 ffff:0~ffff:b 中的数据直接累加到 dx 中?
3. 能否将 ffff:0~ffff:b 中的数据累加到 dl 中，并设置 (dh)=0，从而实现累加到 dx 中?

问题解答：

1. ffff:0~ffff:b 内存単元中的数据是字节型数据，范围在 0~255 之间，12 个这样的数据相加，结果不会大于 65535，可以在dx中存放下；
2. ffff:0~ffff:b 中的数据是 8 位的，不能直接加到 16 位寄存器 dx 中；
3. 不行，因为 dl 是 8 位寄存器，能容纳的数据的范围在 0~ 255 之间，ffff:0~ffff:b 中的数据也都是 8 位，如果仅向 dl 中累加 12 个 8 位数据，很有可能造成进位丢失。

编程思路：用一个 16 位寄存器来做中介。将内存单元中的 8 位数据赋值到一个 16 位寄存器 ax 中，再将 ax 中的数据加到 dx 上，从而使两个运算对象的类型匹配并且结果不会超界。

``` asm
assume cs:code
code segment

    mov ax, 0ffffh
    mov ds, ax
    mov bx, 0               ; 初始化 ds:bx 指向 ffff:0

    mov dx, 0               ; 初始化累加寄存器 dx， (dx)=0

    mov cx, 12              ; 初始化循环计数寄存器 cx, (cx)=12

s:  mov al, [bx]
    mov ah, 0
    add dx, ax              ; 间接向 dx 中加上 ((ds)*16+(bx)) 单元的数值
    inc bx                  ; ds:bx 指向下一个单元
    loop s

    mov ax, 4c00h
    int 21h

code ends

end
```

---

## 段前缀

指令 `mov ax, [bx]` 中，内存单元的偏移地址由 bx 给出，而段地址默认在 ds 中。我们可以在访问内存单元的指令中显式地给出内存单元的段地址所在的段寄存器。比如：

1. `mov ax,ds:[bx]`
2. `mov ax,cs:[bx]`
3. `mov ax,ss:[bx]`
4. `mov ax,es:[bx]`
5. `mov ax,ss:[0]`
6. `mov ax,cs:[0]`
7. ...

这些出现在访问内存单元的指令中，用于显式地指明内存单元的段地址的 “ds:” “cs:” “ss:” “es:〞，在汇编语言中称为段前缀。

### 一段安全的空间

在 8086 模式中，随意向一段内存空间写入内容是很危险的，因为这段空间中可能存放着重要的系统数据或代码。

DOS 方式下， 一般情况，0:200~0:2ff 空间中没有系统或其他程序的数据或代码，以后，我们需要直接向一段内存中写入内容时，就使用 0:200~0:2ff 这段空间。

### 段前缀的使用

编程目标：将内存 ffff:0~ffff:b 单元中的数据复制到 0:200~0:20b 单元中。

问题：如何指定 0:200~0:20b 的段地址？

使用 0020:0~0020:b 来描述 0:200~0:20b，它们描述的是同一段内存空间。 因为源始单元 ffff:X 和目标单元 0020:X 的偏移地址 X 是变量，将 0:200~0:20b 用 0020:0~0020:b 描述，就是为了使目标单元的偏移地址和源始单元的偏移地址从同一数值又开始。

``` asm
assume cs:code
code segment

    mov ax, 0ffffh
    mov ds, ax                  ; (ds)=0ffffh

    mov ax, 20h
    mov es, ax                  ; (es)=0020h

    mov bx, 0                   ; (bx)=0，此时 ds:bx 指向 ffff:0，es:bx 指向0020:0

    mov cx, 12                  ; (cx)=12，循环12次

s:  mov dl, [bx]                ; (dl)=((ds)*16+(bx))，将 ffff:bx 中的数据送入 dl
    mov es:[bx], dl             ; ((es)*16+(bx))=(dl)，将 dl 中的数据送入 0020:bx
    inc bx                      ; (bx)=(bx)+1
    loop s

    mov ax, 4c00h
    int 21h

code ends

end
```

因源始单元 ffff:X 和目标单元 0020:X 相距大于64KB，在不同的 64KB 段里。因此使用 es 存放目标空间 0020:0~0020:b 的段地址，用 ds 存放源始空间 ffff:0~ffff:b 的段地址。

---

