# 汇编语言 —— INT指令


中断信息可以来自 CPU 的内部和外部，当 CPU 的内部有需要处理的事情发生的时候，将产生需要马上处理的中断信息，引发中断过程。

本章，讲解由 int 指令引发的内中断。

---

## INT 指令

int 指令的格式为：int n，n 为中断类型码，**它的功能是引发中断过程**。

CPU 执行 int n 指令，相当于引发一个 n 号中断的中断过程，执行过程如下：

1. 取中断类型码n；
2. 标志寄存器入栈，IF=0，TF=0；
3. CS、IP 入栈；
4. (IP)=(nx4)，(CS)=(nx4+2)。

从此处转去执行 n 号中断的中断处理程序。

一般情况下，系统将一些具有一定功能的子程序，以中断处理程序的方式提供给应用程序调用。我们在编程的时候，可以用 int 指令调用这些子程序。也可以自己编写一些中断处理程序供别人使用。

中断处理程序又称中断例程。

---

## 编写供应用程序调用的中断例程

这里通过两个案例来讨论可以供应用程序调用的中断例程的编写方法。

### 案例1：编写、安装中断 7ch 的中断例程

功能：求 word 型数据的平方。
参数：(ax) = 要计算的数据。
返回值：dx、ax 中存放结果的高 16 位和低 16 位。

应用举例：求 2*3456^2 的值

``` text
assume cs:code

code segment

    start:
            mov ax, 3456                            ; (ax)=3456
            int 7ch                                 ; 计算 ax 中数据的平方
            add ax, ax
            adc dx, dx                              ; dx:ax 存放结果，将结果乘以2

            mov ax, 4c00h
            int 21h

code ends

end start
```

我们要做以下 3 部分工作：

1. 编写实现求平方功能的程序；
2. 安装程序，将其安装在 0:200 处；
3. 设置中断向量表，将程序的入口地址保存在 7ch 表项中，使其成为中断 7ch 的中断例程。

安装程序如下：

``` text
assume cs:code

code segment

    start:
            mov ax, cs
            mov ds, ax
            mov si, offset sqr                      ; 设置 ds:si 指向源地址

            mov ax, 0
            mov es, ax
            mov di, 200                             ; 设置 es:di 指向目的地址

            mov cx, offset sqrend-offset sqr        ; 设置 cx 为传输长度

            cld                                     ; 设置传输方向为正

            rep movsb

            mov ax, 0
            mov es, ax
            mov word ptr es:[7ch*4], 200h
            mov word ptr es:[7ch*4+2], 0

            mov ax, 4c00h
            int 21h

      sqr:
            mul ax
            iret                                    ; 在中断例程最后使用 iret 指令

   sqrend:
            nop

code ends

end start
```

CPU 执行 int 7ch 指令进入中断例程之前，标志寄存器、当前的 CS 和 IP 被压入栈中，在执行完中断例程后，应该用 iret 指令恢复 int 7ch 执行前的标志寄存器和 CS、IP 的值，从而接着执行应用程序。

int 指令和 iret 指令的配合使用与 call 指令和 ret 指令的配合使用具有相似的思路。

### 案例2：编写、安装中断 7ch 的中断例程

功能：将一个全是字母，以 0 结尾的字符串，转化为大写。
参数：ds:si 指向字符串的首地址。
应用举例：将 data 段中的字符串转化为大写。

``` text
assume cs:code

data segment
    db 'conversation',0
data ends

code segment
    start:
            mov ax, data
            mov ds, ax
            mov si, 0
            int 7ch

            mov ax, 4c00h
            int 21h
code ends

end start
```

安装程序如下：

``` text
assume cs:code

code segment

    start:
            mov ax, cs
            mov ds, ax
            mov si, offset capital

            mov ax, 0
            mov es, ax
            mov di, 200h

            mov cx, offset capitalend-offset capital

            cld

            rep movsb

            mov ax, 0
            mov es, ax
            mov word ptr es:[7ch*4], 200h
            mov word ptr es:[7ch*4+2], 0

            mov ax, 4c00h
            int 21h

  capital:
            push cx
            push si

   change:
            mov cl, [si]
            mov ch, 0
            jcxz ok
            and byte ptr [si], 11011111b
            inc si
            jmp short change

       ok:
            pop si
            pop cx
            iret

capitalend:
            nop

code ends

end start
```

在中断例程 capital 中用到了寄存器 si 和 cx，编写中断例程和编写子程序的时候具有同样的问题，就是要避免寄存器的冲突。应该注意例程中用到的寄存器的值的保存和恢复。

---

## 对 int、iret 和栈的深入理解

问题：用 7ch 中断例程完成 loop 指令的功能。

loop s 的执行需要两个信息，循环次数和到 s 的位移，所以，7ch 中断例程要完成 loop 指令的功能，也需要这两个信息作为参数。我们用 cx 存放循环次数，用 bx 存放位移。

应用举例：在屏幕中间显示 80 个 “!”。应用代码如下：

``` text
assume cs:code

code segment

    start:
            mov ax, 0b800h
            mov es, ax
            mov di, 160*12

            mov bx, offset s-offset se          ; 设置从标号 se 到标号 s 的转移位移
            mov cx, 80

        s:
            mov byte ptr es:[di], '!'
            add di, 2
            int 7ch                             ; 如果 (cx)!=0，转移到标号 s 处

       se:
            nop

            mov ax, 4c00h
            int 21h

code ends

end start
```

分析：为了模拟 loop 指令，7ch 中断例程应具备下面的功能：

1. dec cx；
2. 如果 (cx)!=0，转到标号 s 处执行，否则向下执行。

int 7ch 引发中断过程后，进入 7ch 中断例程，在中断过程中，当前的标志寄存器、CS 和 IP 都要压栈，此时压入的 CS 和 IP 中的内容，分别是调用程序的段地址(可以认为是标号 s 的段地址)和 int 7ch 后一条指令的偏移地址(即标号 se 的偏移地址)。在中断例程中，可以从栈里取得标号 s 的段地址和标号 se 的偏移地址，用标号 se 的偏移地址加上 bx 中存放的转移位移就可以得到标号 s 的偏移地址。利用 iret 指令，我们将栈中的 se 的偏移地址加上 bx 中的转移位移，则栈中的 se 的偏移地址就变为了 s 的偏移地址。我们再使用 iret 指令，用栈中的内容设置 CS、IP，从而实现转移到标号 s 处。

7ch 中断例程如下：

``` text
lp:
    push bp
    mov bp, sp
    dec cx
    jcxz lpret
    add [bp+2], bx                      ; [bp+2] 存储的是 IP 的值

lpret:
    pop bp
    iret
```

因为要访问栈，使用了 bp，在程序开始处将 bp 入栈保存，结束时出栈恢复。当要修改栈中 se 的偏移地址的时候，栈中的情况为：栈顶处是 bp 原来的数值，下面是 se 的偏移地址，再下面是 s 的段地址，再下面是标志寄存器的值。而此时，bp 中栈顶的偏移地址，所以 ((ss)*16+(bp)+2) 处为 se 的偏移地址，将它加上 bx 中的转移位移就变为 s 的偏移地址。最后用 iret 出栈返回，CS:IP 即从标号 s 处开始执行指令。

{{< admonition type=tip open=true >}}

bp 为基址寄存器，一般在函数中用来保存进入函数时的 sp 的栈顶基址。

每次子函数调用时，系统在开始时都会保存这个两个指针并在函数结束时恢复 sp 和 bp 的值。

``` text
sub:
    push bp                             ; 保存父函数 bp 指针
    mov bp, sp                          ; bp 指向 sp 的基地址

    ; 此时，如果该函数有参数，则
    ;    [bp+4] 是子函数的第一个参数，
    ;    [bp+6] 是子函数的第二个参数。

    mov sp, bp                          ; 将原 sp 指针传回 sp
    pop bp                              ; 恢复原 bp 的值
    ret                                 ; 退出子函数
```

{{< /admonition >}}

---

## BIOS 和 DOS 所提供的中断例程

在系统板的 ROM 中存放着一套程序，称为 BIOS (基本输入输出系统)，BIOS 中主要包含以下几部分内容：

1. 硬件系统的检测和初始化程序；
2. 外部中断和内部中断的中断例程；
3. 用于对硬件设备进行 IO 操作的中断例程；
4. 其他和硬件系统相关的中断例程。

操作系统 DOS 也提供了中断例程，从操作系统的角度来看，DOS 的中断例程就是操作系统向程序员提供的编程资源。

程序员在编程的时候，可以用 int 指令直接调用 BIOS 和 DOS 提供的中断例程，来完成某些工作。

---

## BIOS 和 DOS 中断例程的安装过程

BIOS 和 DOS 提供的中断例程的安装过程如下：

1. 开机后，CPU 一加电，初始化 (CS)=OFFFFH，(IP)=0，自动从 FFFF:0 单元开始执行程序。FFFF:0 处有一条转跳指令，CPU 执行该指令后，转去执行 BIOS 中的硬件系统检测和初始化程序；
2. 初始化程序将建立 BIOS 所支持的中断向量，即将 BIOS 提供的中断例程的入口地址登记在中断向量表中。对于 BIOS 所提供的中断例程，只需将入口地址登记在中断向量表中即可，因为它们是固化到 ROM 中的程序，一直在内存中存在；
3. 硬件系统检测和初始化完成后，调用 int 19h 进行操作系统的引导。从此将计算机交由操作系统控制；
4. DOS 启动后，除完成其他工作外，还将它所提供的中断例程装入内存，并建立相应的中断向量。

---

## 21h 号中断例程

int 21h 中断例程是 DOS 提供的中断例程，其中包含了 DOS 提供给程序员在编程时调用的子程序。

``` text
mov ah, 4ch                     ; 程序返回
mov al, 0                       ; 返回值
int 21h
```

(ah)=4ch 表示调用第 21h 号中断例程的 4ch 号子程序，功能为程序返回，可以提供返回值作为参数。

---

