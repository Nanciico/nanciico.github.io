# 汇编语言 —— 端口


在 PC 机系统中，和 CPU 通过总线相连的芯片除各种存储器外，还有以下 3 种芯片：

1. 各种接口卡(比如，网卡、显卡)上的接口芯片，它们控制接口卡进行工作；
2. 主板上的接口芯片，CPU 通过它们对部分外设进行访问；
3. 其他芯片，用来存储相关的系统信息，或进行相关的输入输出处理。

在这些芯片中，都有一组可以由 CPU 读写的寄存器。这些寄存器，它们在物理上可能处于不同的芯片中，但是它们在以下两点上相同：

1. 都和 CPU 的总线相连，这种连接是通过它们所在的芯片进行的；
2. CPU 对它们进行读或写的时候都通过控制线向它们所在的芯片发出端口读写命令。

从 CPU 的角度，将这些寄存器都当作端口，对它们进行统一编址，从而建立了一个统一的端口地址空间。每一个端口在地址空间中都有一个地址。

CPU 可以直接读写以下 3 个地方的数据：

1. CPU 内部的寄存器；
2. 内存单元；
3. 端口。

---

## 端口的读写

在访问端口的时候，CPU 通过端口地址来定位端口。因为端口所在的芯片和 CPU 通过总线相连，所以，端口地址和内存地址一样，通过地址总线来传送。在 PC 系统中， CPU 最多可以定位 64KB 个不同的端口。则端口地址的范围为 O~65535。

端口的读写指令只有两条: in 和 out，分别用于从端口读取数据和往端口写入数据。

指令 `in al, 60h    ; 从 60h 号端口读入一个字节` 执行时与总线相关的操作如下：

1. CPU 通过地址线将地址信息 60h 发出；
2. CPU 通过控制线发出端口读命令，选中端口所在的芯片，并通知它，将要从中读取数据；
3. 端口所在的芯片将 60h 端口中的数据通过数据线送入 CPU。

{{< admonition type=warning open=true >}}

在 in 和 out 指令中，只能使用 ax 或 al 来存放从端口中读入的数据或要发送到端口中的数据。访问 8 位端口时用 al，访问 16 位端口时用 ax。

对 0~255 以内的端口进行读写时：

``` text
in al, 20h                      ; 从 20h 端口读入一个字节
out 20h, al                     ; 往 20h 端口写入一个字节
```

对 256~65535 的端口进行读写时，端口号放在 dx 中：

``` text
mov dx, 3f8h                    ; 将端口号 3f8h 送入 dx
in al, dx                       ; 从 3f8h 端口读入一个字节
out dx, al                      ; 向 3f8h 端口写入一个字节
```

{{< /admonition >}}

---

## CMOS RAM 芯片

PC 机中，有一个 CMOS RAM 芯片，一般简称为 CMOS。此芯片的特征如下：

1. 包含一个实时钟和一个有 128 个存储单元的 RAM 存储器；
2. 该芯片靠电池供电。所以，关机后其内部的实时钟仍可正常工作，RAM 中的信息不丢失。
3. 128 个字节的 RAM 中，内部实时钟占用 0～0dh 单元来保存时间信息，其余大部分单元用于保存系统配置信息，供系统启动时 BIOS 程序读取。BIOS 也提供了相关的程序，使我们可以在开机的时候配置CMOS RAM 中的系统信息。
4. 该芯片内部有两个端口，端口地址为 70h 和 71h 。CPU 通过这两个端口来读写 CMOS RAM。
5. 70h 为地址端口，存放要访问的 CMOS RAM 单元的地址；71h 为数据端口，存放从选定的 CMOS RAM 单元中读取的数据，或要写入到其中的数据。

可见，CPU 对 CMOS RAM 的读写分两步进行，比如，读 CMOS RAM 的 2 号单元:

1. 将 2 送入端口 70h；
2. 从端口 71h 读出 2 号单元的内容。

{{< admonition type=example open=true >}}

读取 CMOS RAM 的 2 号单元内容：

``` text
assume cs:code

code segment

    start:
            mov al, 2h
            out 70h, al                     ; 将 2 送入端口 70h

            in  al, 71h                     ; 从 71h 读出 2 号单元内容

            mov ax, 4c00h
            int 21h

code ends

end start
```

向 CMOS RAM 的 2 号单元写入 0：

``` text
assume cs:code

code segment

    start:
            mov al, 2h
            out 70h, al                     ; 将 2 送入端口 70h

            mov al, 0
            out 71h, al                     ; 将 0 写入 2 号单元

            mov ax, 4c00h
            int 21h

code ends

end start
```

{{< /admonition >}}

---

## SHL 和 SHR 指令

shl 是逻辑左移指令，它的功能为：

1. 将一个寄存器或内存单元中的数据向左移位；
2. 将最后移出的一位写入 CF 中；
3. 最低位用 0 补充。

{{< admonition type=example open=true >}}

``` text
mov al, 01010001b
mov cl,3                        ; 移动位数大于 1 时，必须将移动位数放在 cl 中
shl al, cl
```

执行后 (al)=10001000b，因为最后移出的一位是 0，所以 CF=0。

将 X 逻辑左移一位，相当于执行 X=X*2。

{{< /admonition >}}

shr 是逻辑右移指令，它和 shl 所进行的操作刚好相反：

1. 将一个寄存器或内存单元中的数据向右移位；
2. 将最后移出的一位写入 CF 中；
3. 最高位用 0 补充。

{{< admonition type=example open=true >}}

``` text
mov al, 01010001b
mov cl,3                        ; 移动位数大于 1 时，必须将移动位数放在 cl 中
shr al, cl
```

执行后 (al)=00001010b，因为最后移出的一位是 0，所以 CF=0。

将 X 逻辑右移一位，相当于执行 X=X/2。

{{< /admonition >}}

---
