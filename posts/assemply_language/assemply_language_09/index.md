# 汇编语言 —— 转移指令的原理


可以修改 IP，或同时修改 CS 和 IP 的指令统称为转移指令。

8086CPU 的转移行为有以下几类。

- 只修改 IP 时，称为段内转移，比如： `jmp ax`。
- 同时修改 CS 和 IP 时，称为段间转移，比如： `jmp 1000:0`。

由于转移指令对 IP 的修改范围不同，段内转移又分为：短转移和近转移。

- 短转移 IP 的修改范围为 -128~127。
- 近转移 IP 的修改范围为 -32768~32767。

8086CPU 的转移指令分为以下几类。

- 无条件转移指令(如:jmp)
- 条件转移指令
- 循环指令(如: loop)
- 过程
- 中断

我们主要通过深入学习无条件转移指令 jmp 来理解 CPU 执行转移指令的基本原理。

---

## 操作符 offset

操作符 offset 在汇编语言中是由编译器处理的符号，它的功能是取得标号的偏移地址。

{{< admonition type=example open=true >}}

在如下的程序中 offset 操作符取得了标号 start 和 s 的偏移地址 0 和 3。

``` text
assume cs:codesg

codesg segment
    start:  mov ax, offset start    ; 相当于 mov ax, 0
        s:  mov ax, offset s        ; 相当于 mov ax, 3 (第一条指令长度为 3 字节)
codesg ends

end start
```

{{< /admonition >}}

---

## jmp 指令

jmp 为无条件转移指令，可以只修改 IP，也可以同时修改 CS 和 IP。

jmp 指令要给出两种信息:

1. 转移的目的地址
2. 转移的距离(段间转移、段内短转移、段内近转移)

### 依据位移进行转移的 jmp 指令

`jmp short 标号` 这种格式的 jmp 指令实现的是段内短转移。它对 IP 的修改范围为 -128~127。

``` text
assume cs:codesg

codesg segment

    start:  mov ax, O
            jmp short s         ; 段内短转移
            add ax, 1
        s:  inc ax

codesg ends

end start
```

上述代码在 Debug 中翻译为机器码，可以看到机器码中不包含转移目的地址而是 “EB 03”。

{{< image src="/images/assembly_language/09_01.jpg" caption="段内短转移程序机器码" title="段内短转移程序机器码" >}}

这是因为 **CPU在执行 jmp 指令的时候并不需要转移的目的地址**。在 “jmp short 标号〞指令所对应的机器码中，并不包含转移的目的地址，而包含的是转移的位移。这个位移，是编译器根据汇编指令中的 “标号” 计算出来的，计算方法如下图所示：

{{< image src="/images/assembly_language/09_02.jpg" caption="转移位移的计算方法" title="转移位移的计算方法" >}}

“jmp short 标号” 的功能为:(IP)=(IP)+8位位移。

1. 8 位位移 = 标号处的地址-jmp 指令后的第一个字节的地址；
2. short 指明此处的位移为 8 位位移；
3. 8 位位移的范围为 -128~127，用补码表示；
4. 8 位位移由编译程序在编译时算出。

“jmp near ptr 标号” 格式的命令，它实现的是段内近转移。功能为：(IP)=(IP)+16位位移。

1. 16 位位移 = 标号处的地址-jmp指令后的第一个字节的地址；
2. near ptr 指明此处的位移为 16 位位移，迸行的是段内近转移；
3. 16 位位移的范围为 -32768~32767，用补码表示;
4. 16 位位移由编译程序在编译时算出。

### 转移的目的地址在指令中的 jmp 指令

“jmp far ptr 标号” 实现的是段间转移，又称为远转移。

``` text
assume cs:codesg

codesg segment

    start:  mov ax, 0
            mov bx, 0
            jmp far ptr s           ; 段间转移
            db 256 dup (0)
        s:  add ax,1
            inc ax
codesg ends

end start
```

在 Debug 中将上述代码翻译成机器码，结果如下图所示：

{{< image src="/images/assembly_language/09_03.jpg" caption="段间转移程序机器码" title="段间转移程序机器码" >}}

代码 “jmp far ptr s” 所对应的机器码: “EA 0B 01 BD 0B”，其中包含转移的目的地址。“0B 01 BD 0B” 是目的地址在指令中的存储顺序，高地址的 “BD 0B” 是转移的段地址：“0BBDH”，低地址的 “0B 01” 是偏移地址: “010BH”。

### 转移地址在寄存器中的 jmp 指令

指令格式：jmp 16位 reg
功能:(IP)=(16 位 reg)

### 转移地址在内存中的jmp 指令

转移地址在内存中的 jmp 指令有两种格式：

1. jmp word ptr 内存单元地址(段内转移)

    - 功能：从内存单元地址处开始存放着一个字，是转移的目的偏移地址。

    ``` text
        mov ax, 0123H
        mov ds:[0],ax
        jmp word ptr ds:[0]         ; 执行后 IP = 0123H
    ```

2. jmp dword ptr 内存单元地址(段间转移)

    - 功能：从内存单元地址处开始存放着两个字，高地址处的字是转移的目的段地址，低地址处是转移的目的偏移地址。

    ``` text
        mov ax, 0123H
        mov ds:[0],ax
        mov word ptr ds:[2], 0
        jmp dword ptr ds:[0]        ; 执行后 CS = 0000H，IP = 0123H
    ```

---

## jcxz 指令

jcxz 指令为有条件转移指令，所有的有条件转移指令都是短转移，在对应的机器码中包含转移的位移，而不是目的地址。对 IP 的修改范围都为：-128~127。

- 格式：jcxz 标号。
- 操作：当 (cx)=0 时，IP=(IP)+8位位移。当 (cx)!=0 时，什么也不做 (程序向下执行)。

{{< admonition type=tip title="jcxz 指令的 C语言描述" open=true >}}

可以用以下 C语言代码来描述 jcxz 指令的功能：

`if ((cx)==0) jmp short 标号`

{{< /admonition >}}

---

## loop 指令

loop 指令为循环指令，所有的循环指令都是短转移，在对应的机器码中包含转移的位移，而不是目的地址。对 IP 的修改范围都为： -128~127。

{{< admonition type=tip title="loop 指令的 C语言描述" open=true >}}

可以用以下 C语言代码来描述 loop 指令的功能：

``` text
(cx)--
if((cx) != 0) jmp short 标号
```

{{< /admonition >}}

---

## 实验 9

``` text
assume cs:code, es:data, ss:stack

data segment
    db 'welcome to masm!'
    db 02H, 24H, 71H            ; 颜色
data ends

stack segment
    db 10 dup (0)
stack ends

code segment

    start:
            mov ax, data
            mov es, ax
            mov di, 0           ; es:di -> data
            mov si, 16          ; es:si -> 颜色

            mov ax, 0B800H
            mov ds, ax
            mov bx, 0720H       ; ds:bx -> 缓冲区

            mov ax, stack
            mov ss, ax
            mov sp, 0

            mov cx, 3           ; 三行

      row:  
            push cx
            mov cx, 16          ; 每行 16 个字符

      col:
            mov al, es:[di]
            mov ds:[bx], al

            mov ah, es:[si]
            mov ds:[bx+1], ah

            inc di
            add bx, 2
            loop col

            add bx, 128         ; bx 指向下一行的起始地址
            mov di, 0
            inc si              ; 下一个颜色

            pop cx
            loop row

            mov ax, 4c00H
            int 21H

code ends

end start
```

---

