# 汇编语言 —— 外中断


CPU 除了有运算能力外，还要有 I/O 能力。

要及时处理外设的输入，需要解决两个问题：

1. 外设的输入随时可能发生，CPU 如何得知？
2. CPU 从何处得到外设的输入？

---

## 接口芯片和端口

PC 系统的接口卡和主板上，装有各种接口芯片。这些外设接口芯片的内部有若干寄存器，CPU 将这些寄存器当作端口来访问。

外设的输入不直接送入内存和 CPU，而是送入相关的接口芯片的端口中；CPU 向外设的输出也不是直接送入外设，而是先送入端口中，再由相关的芯片送到外设。CPU 还可以向外设输出控制命令，而这些控制命令也是先送到相关芯片的端口中，然后再由相关的芯片根据命令对外设实施控制。

**CPU 通过端口和外部设备进行联系。**

---

## 外中断信息

外设随时都可能发生需要 CPU 及时处理的事件，CPU 如何及时得知并进行处理？

CPU 提供中断机制来满足这种需要。

当 CPU 外部有需要处理的事情发生的时候比如外设的输入到达，相关芯片将向 CPU 发出相应的中断信息。CPU 在执行完当前指令后，可以检测到发送过来的中断信息，引发中断过程，处理外设的输入。

在 PC 系统中，外中断源一共有以下两类：

1. 可屏蔽中断
2. 不可屏蔽中断

### 可屏蔽中断

可屏蔽中断是 CPU 可以不响应的外中断。CPU 是否响应可屏蔽中断，要看标志寄存器的 IF 位的设置。当 CPU 检测到可屏蔽中断信息时，如果 IF=1，则 CPU 在执行完当前指令后响应中断，引发中断过程：如果 IF=0，则不响应可屏蔽中断。

内中断将 IF 置为 0 的原因是：在进入中断处理程序后，禁止其他的可屏蔽中断。

8086CPU 提供的设置 IF 的指令如下：

1. sti，设置 IF=1；
2. cli，设置 IF=0。

几乎所有由外设引发的外中断，都是可屏蔽中断。当外设有需要处理的事件（比如说键盘输入）发生时，相关芯片向 CPU 发出可屏蔽中断信息。

### 不可屏蔽中断

不可屏蔽中断是 CPU 必须响应的外中断。当 CPU 检测到不可屏蔽中断信息时，则在执行完当前指令后，立即响应，引发中断过程。

对于 8086CPU，不可屏蔽中断的中断类型码固定为 2，所以中断过程中，不需要取中断类型码。则不可屏蔽中断的中断过程为：

1. 标志寄存器入栈，IF=0，TF=0；
2. CS、IP 入栈；
3. (IP)=(8), (CS)=(0AH)。

不可屏蔽中断是在系统中有必须处理的紧急情况发生时用来通知 CPU 的中断信息。

---

## 指令系统总结

8086CPU提供以下几大类指令。

### 数据传送指令

比如，mov、push、pop、pushf、popf、xchg 等都是数据传送指令，这些指令实现寄存器和内存、寄存器和寄存器之间的单个数据传送。

### 算术运算指令

比如，add、sub、adc、sbb、inc、dec、cmp、imul、idiv、aaa 等都是算术运算指令，这些指令实现寄存器和内存中的数据的算数运算。它们的执行结果影响标志寄存器的 sf、zf、of、cf、pf、af 位。

### 逻辑指令

比如，and、or、not、xor、test、shl、shr、sal、sar、rol、ror、rcl、rcr 等都是逻辑指令。除了 not 指令外，它们的执行结果都影响标志寄存器的相关标志位。

### 转移指令

可以修改 IP，或同时修改 CS 和 IP 的指令统称为转移指令。转移指令分为以下几类：

1. 无条件转移指令，比如，jmp；
2. 条件转移指令，比如，jcxz、je、jb、ja、jnb、jna 等；
3. 循环指令，比如，loop;
4. 过程，比如，call、ret、retf；
5. 中断，比如，int、iret。

### 处理机控制指令

这些指令对标志寄存器或其他处理机状态进行设置，比如，cld、std、cli、sti、nop、clc、cmc、stc、hlt、wait、ese、lock 等都是处理机控制指令。

### 串处理指令

这些指令对内存中的批量数据进行处理，比如，movsb、movsw、cmps、scas、lods、stos 等。若要使用这些指令方便地进行批量数据的处理，则需要和 rep、repe、repne 等前缀指令配合使用。

---

