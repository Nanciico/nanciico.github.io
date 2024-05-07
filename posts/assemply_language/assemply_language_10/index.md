# 汇编语言 —— CALL和RET指令


call 和 ret 指令都是转移指令，它们都修改 IP，或同时修改CS 和 IP。它们经常被共同用来实现子程序的设计。

---

## ret 和 retf

ret 指令用栈中的数据，修改 IP 的内容，从而实现近转移；
retf 指令用栈中的数据，修改 CS 和 IP 的内容，从而实现远转移。

{{< admonition type=tip open=true >}}

CPU 执行 ret 指令时，相当于进行：

``` text
pop IP
```

CPU 执行 retf 指令时，相当于进行：

``` text
pop IP
pop CS
```

{{< /admonition >}}

---

## call 指令

CPU 执行 call 指令时，进行两步操作:

1. 将当前的 IP 或 CS和IP 压入栈中；
2. 转移。

### 依据位移进行转移的 call 指令

格式：`call 标号` (将当前的 IP 压栈后，转到标号处执行指令)

位移 = 标号处的地址 - call指令后的第一个字节的地址，由编译程序在编译时算出。

{{< admonition type=tip open=true >}}

CPU 执行 “call 标号” 指令时，相当于进行：

``` text
push IP
jmp near ptr 标号
```

{{< /admonition >}}

### 转移的目的地址在指令中的 call 指令

格式：`call far ptr 标号`

{{< admonition type=tip open=true >}}

CPU 执行 “call far ptr 标号” 指令时，相当于进行：

``` text
push CS
push IP
jmp far ptr 标号
```

{{< /admonition >}}

### 转移地址在寄存器中的 call 指令

格式：`call 16位reg`

{{< admonition type=tip open=true >}}

CPU 执行 “call 16位reg” 指令时，相当于进行：

``` text
push IP
jmp 16位reg
```

{{< /admonition >}}

### 转移地址在内存中的 call 指令

格式：

1. `call word ptr 内存单元地址`
2. `call dword ptr 内存单元地址`

{{< admonition type=tip open=true >}}

CPU 执行 “call word ptr 内存单元地址” 指令时，相当于进行：

``` text
push IP
jmp word ptr 内存单元地址
```

CPU 执行 “call dword ptr 内存单元地址” 指令时，相当于进行：

``` text
push CS
push IP
jmp dword ptr 内存单元地址
```

{{< /admonition >}}

---

## call 和 ret 的配合使用

可以利用 call 和 ret 来实现子程序的机制。子程序的框架如下：

``` text
assume cs:code

code segment

    main:   
            ...
            call sub1               ; 调用子程序 sub1
            ...
            ...
            mov ax, 4c00H
            int 21H

    sub1:                           ; 子程序 sub1 开始
            ...
            call sub2               ; 调用子程序 sub2
            ...
            ...
            ret                     ; 子程序 sub1 返回

    sub2:                           ; 子程序 sub2 开始
            ...
            ...
            ret                     ; 子程序 sub2 返回

code ends

end main
```

---

## mul 指令

格式：`mul reg` 或 `mul 内存单元`

- 8位乘法：两个乘数一个默认在 AL 中，另一个放在 8 位 reg 或内存字节单元中，结果默认放在 AX 中；
- 16位乘法：两个乘数一个默认在 AX 中，另一个放在 16 位 reg 或内存字单元中，结果高位默认在 DX 中存放，低位在 AX 中存放。

---

## 子程序设计中的相关问题和解决方法

### 参数和结果传递的问题

此处讨论的问题是：**应该如何存储子程序需要的参数和产生的返回值？**

用寄存器来存储参数和结果是最常使用的方法，对于存放参数的寄存器和存放结果的寄存器，调用者和子程序的读写操作恰恰相反：

- 调用者将参数送入参数寄存器，从结果寄存器中取到返回值；
- 子程序从参数寄存器中取到参数，将返回值送入结果寄存器。

{{< admonition type=example open=true >}}

编程，计算 data 段中第一组数据的 3 次方，结果保存在后面一组 dword 单元中。

``` text
assume cs:code

data segment
    dw 1,2,3,4,5,6,7,8
    dd 0,0,0,0,0,0,0,0
data ends

code segment

    start:
            mov ax, data
            mov ds, ax

            mov si, 0               ; ds:si 指向第一组 word 单元
            mov di, 16              ; ds:di 指向第二组 dword 单元

            mov cx, 8
        s:
            mov bx, [si]            ; 将子程序 cube 参数存入 bx 中
            call cube

            mov [di], ax            ; 取出 ax 中的值
            mov [di].2, dx          ; 取出 dx 中的值
            add si, 2               ; ds:si 指向下一个 word 单元
            add di, 4               ; ds:di 指向下一个 dword 单元
            loop s


            mov ax, 4c00h
            int 21h

     cube:
            mov ax, bx
            mul bx
            mul bx
            ret                     ; 计算结果存放在 dx 和 ax 中

code ends

end start
```

{{< /admonition >}}

### 批量数据的传递

#### 使用内存空间传递参数

前面的例程中，子程序 cube 只有一个参数，放在 bx 中。如果有两个参数，那么可以用两个寄存器来放，可是如果需要传递的数据有 N 个，该怎样存放呢？

在这种时候，我们将批量数据放到内存中，然后将它们所在内存空间的首地址放在寄存器中，传递给需要的子程序。

{{< admonition type=example open=true >}}

编程，设计一个子程序，将一个全是字母的字符串转化为大写。

这个子程序需要知道两件事，字符串的内容和字符串的长度。

由于字符串中的字母可能很多，不便将整个字符串中的所有字母传递给子程序，我们可以将字符串在内存中的首地址存放在寄存器中传递给子程序。

因为子程序中需要循环，循环的次数是字符串的长度，我们可以将字符串的长度放到 cx 中传递给子程序。

``` text

assume cs:code

data segment
    db 'conversation'
data ends

code segment

    start:
            mov ax, data
            mov ds, ax

            mov si, 0                       ; ds:si 指向数据所在空间首地址
            mov cx, 12                      ; cx 存放字符串长度

            call capital

            mov ax, 4c00h
            int 21h

  capital:
            and byte ptr [si], 11011111b
            inc si
            loop capital
            ret

code ends

end start
```

{{< /admonition >}}

#### 使用栈传递参数

除了用寄存器传递参数外，还有一种通用的方法是用栈来传递参数。

这种技术和高级语言编译器的工作原理密切相关。用栈传递参数的原理十分简单，就是由调用者将需要传递给子程序的参数压入栈中，子程序从栈中取得参数。

{{< admonition type=example open=true >}}

编程，编写一个子程序，计算 (a-b)^3，a、b 为字型数据。

因为用栈传递参数，所以调用者在调用程序的时候要向栈中压入参数，子程序在返回的时候可以用 ret n 指令将栈顶指针修改为调用前的值。

``` text
main:
        mov ax, 1
        push ax                 ; 压入 b
        mov ax, 3
        push ax                 ; 压入 a
        call cube               ; 压入 IP，调用 cube 子程序

        mov ax, 4c00h
        int 21h

cube:
        push bp                 ; 栈中暂存 bp 的值
        mov bp, sp              ; 子程序中使用 bp 作为栈顶指针
        mov ax, [bp+4]          ; 栈中 a 的值送入 ax
        sub ax, [bp+6]          ; 减栈中 b 的值
        mov bp, ax
        mul bp
        mul bp
        pop bp                  ; 还原 bp 值
        ret 4                   ; 调用子程序前压入了两个参数，所以用 ret 4 返回
```

{{< /admonition >}}

{{< admonition type=tip open=true >}}

指令 “ret n” 的含义用汇编语法描述为：

``` text
pop ip
add sp, n
```

{{< /admonition >}}

既然用栈传递参数和高级语言编译器的工作原理密切相关，我们通过一个 C 语言程序编译后的汇编语言程序，看一下栈在参数传递中的应用。

{{< admonition type=info open=true >}}

注意：在 C 语言中，局部变量也在栈中存储。

``` C
void add(int, int, int);

main()
{
    int a = 1;
    int b = 2;
    int c = 0;

    add(a, b, c);

    c++;
}

void add(int a, int b, int c)
{
    c = a + b;
}
```

编译后的汇编程序

``` text
main:
        mov bp, sp
        sub sp, 6
        mov word ptr [bp-6], 0001       ; int a = 1
        mov word ptr [bp-4], 0002       ; int b = 2
        mov word prt [bp-2], 0003       ; int c = 0

        push [bp-2]
        push [bp-4]
        push [bp-6]                     ; 传入参数

        call add

add:
        push bp
        mov bp, sp
        mov ax, [bp+4]
        add ax, [bp+6]                  ; 计算 a + b
        mov [bp+8], ax                  ; c = a + b
        mov sp, bp
        pop bp
        ret 6
```

可以观察到，子程序并未改变主程序中的 c 局部变量的值。

{{< /admonition >}}

### 寄存器冲突的问题

子程序中使用的寄存器，很可能在主程序中也要使用，造成了寄存器使用上的冲突。那么如何来避免这种冲突呢？

解决这个问题的简洁方法是，**在子程序的开始将子程序中所有用到的寄存器中的内容都保存起来，在子程序返回前再恢复。可以用栈来保存寄存器中的内容。**

以后我们编写子程序的标准框架如下：

``` text
子程序开始：
            子程序中使用的寄存器入栈

            子程序内容

            子程序中使用的寄存器出栈

            返回（ret、retf）
```

{{< admonition type=example open=true >}}

编程，将 data 段中的字符串全部转化为大写。

程序要处理的字符串以 0 作为结尾符，子程序可以一次读取每个字符串进行检测，如果不是 0，就进行大写转化；如果是 0，就结束处理。由于可通过检测 0 而知道是否已经处理完字符串，所以子程序可以不需要字符串的长度作为参数。可以用 jcxz 来检测 0。

``` text
assume cs:code

data segment
    db 'word', 0
    db 'unix', 0
    db 'wind', 0
    db 'good', 0
data ends

code segment

    start:
            mov ax, data
            mov ds, ax
            mov bx, 0                       ; ds:bx 指向每个字符串首地址

            mov cx, 4
        s:
            mov si, bx                      ; ds:si 指向单个字符
            call capital
            add bx, 5
            loop s

            mov ax, 4c00h
            int 21h

  capital:
            push cx                         ; 子程序中使用的寄存器入栈
            push si
   change:                                  ; 子程序内容
            mov cl, [si]
            mov ch, 0
            jcxz ok
            and byte ptr [si], 11011111b
            inc si
            jmp short change
       ok:
            pop si                          ; 子程序中使用的寄存器出栈
            pop cx                          ; 注意寄存器入栈和出栈的顺序
            ret
```

{{< /admonition >}}

---

