# 汇编语言 —— 包含多个段的程序


在操作系统中，合法地通过操作系统取得的空间都是安全的。程序取得所需空间的方法有两种：

- 在加载程序的时候为程序分配；
- 在执行的过程中向系统申请（不讨论）；

我们通过在源程序中定义段来进行内存空间的获取。

---

## 将数据、代码、栈放入不同的段

为何要将数据、代码、栈放入不同的段？

1. 把它们放到一个段中使程序显得混乱；
2. 如果数据、栈和代码需要的空间超过 64KB，就不能放在一个段中(8086 模式的限制，一个段的容量不能大于 64KB)。

如下面的程序所示，这个程序将数据段定义的数据逆序存放。

``` text
assume cs:code, ds:data, ss:stack

data segment
    dw 0123h, 0456h, 0789h, Oabch, Odefh, Ofedh, Ocbah, 0987h
data ends

stack segment
    dw 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0
stack ends

code segment

start:  mov ax,stack
        mov ss,ax
        mov sp,20h      ; 设置栈顶 ss:sp 指向 stack:20

        mov ax,data
        mov ds,ax       ; ds 指向 data 段

        mov bx, O       ; ds:bx 指向 data 段中的第一个单元

        mov cx,8
    s:  push [bx]
        add bx,2
        loop s

        mov bx,O
        mov cx,8

    sO: pop [bx]
        add bx,2
        loop sO

        mov ax,4c00h
        int 21h

code ends

end start               ; 指明程序入口为 start，程序加载进内存后，CS:IP 指向该入口
```

### 定义多个段的方法

从代码中可看出，定义多个段只需要对每个段定义不同的段名即可。

### 对段地址的引用

代码片段：`mov ax,stack` 和 `mov ax,data` 都是将名称为 stack 或 data 的段地址送入寄存器 ax。程序中对段名的引用，会被编译器处理为一个表示段地址的数值。

### “代码段”、“数据段”、“栈段”完全是我们的安排

在上述程序中，定义了三个段，“code”、“data”和“stack”，我们分别安排它们存放代码、数据和栈。这是我们人为的安排：

1. 我们在源程序中为这三个段起了具有含义的名称，数据段“data” ，代码段“code” ，栈段"stack"。但如此命名后 CPU 并不会去执行“code”段中的内容，处理“data”段中的数据，将“stack”当做栈。

2. 伪指令：`assume cs:code, ds:data, ss:stack` 将cs、ds 和 ss 分别和 code、data、stack 段相连。但 CPU 不会将 cs 指向 code，ds 指向 data, ss 指向 stack，从而按照我们的意图来处理这些段，因为 assume 是伪指令，是由编译器执行的，CPU 并不知道这些伪指令。

3. 若要让 CPU 按照我们的安排行事，需要用机器指令控制它，源程序中只有汇编指令是 CPU 要执行的内容。因此为了将 stack 段作为栈使用，需要设置 ss:sp 指向 stack:20。为了将 data 作为数据段，需要设置 ds 指向 data，用 bx 存放 data 段的偏移地址。

总之，CPU 到底如何处理我们定义的段中的内容，是当作指令执行，当作数据访问，还是当作栈空间，完全是靠程序中具体的汇编指令，和汇编指令对 CS:IP、SS:SP、DS 等寄存器的设置来决定的。

---

