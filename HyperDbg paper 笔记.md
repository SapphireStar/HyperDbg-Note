# HyperDbg paper 笔记

## 1. 介绍HyperDbg

Debuggers 是软件开发中不可或缺的工具，开发者使用它们来提高软件效率和安全性。但是现在大部分的Debuggers尽管使用了最新的技术，使得自身的隐蔽度较高，但却没有丰富的调试功能。

但HyperDbg很好地解决了这个问题，它避免使用操作系统API和软件调试机制，取而代之的是使用第二层页表(Second Layer Page Table)，也称扩展页表(Extended Page Tables)，来监控kernel和user executions

由于HyperDbg避免使用操作系统自带的Debugging API，因此它的隐蔽性非常好。使用13个 packers and protectors来检测HyperDbg，然而没有任何一个能够检测到HyperDbg，而其他的Debuggers平均有44%的概率被检测出来，且没有任何一个debugger能够只被三个packers and protectors检测出来。实验表明，hyperdbg能够比WinDbg和x64Dbg多检测22%和26%的恶意软件。

为了高性能的调试，hyperdbg使用了兼容VMX-root的高性能脚本引擎，性能远高于其他调试器



## 2. Debugger技术背景

用户模式调试器和内核模式调试器：

- 用户模式调试器

  用户模式调试器主要有x64dbg，Ollydbg和Immunity Debugger。

  用户模式调试器为用户提供了一个方便和隔离的环境。

- 内核模式调试器

  主要有WinDbg，GDB

  我们都知道，在操作系统中，内核模式比用户模式拥有更高的权限，能够修改更多信息，因此，内核模式调试器在访问寄存器和内存方面有更高的权限

  广泛应用于逆向工程和恶意软件分析

随着恶意软件逃逸技术（malware evasion techniques）技术的进步，开发者对于更加透明的调试器更感兴趣，它们不会对执行过程有过大的更改。

### intel虚拟化技术（VT-x)

向ISA，Instruction Set Architecture，引入了新的数据结构和，该技术使得多核CPU的核像几个独立的处理器一样运行，使得同一个CPU上能有多个操作系统同时运行。

### intel扩展页表技术（Extended Page Table——EPT）

Intel VT-x技术带有硬件辅助的内存管理单元(MMU)和第二级地址转换(SLAT)的实现，即扩展页表(EPT)。

过在CPU级别上将来宾物理地址(Guest Physical Address——GPA)转换为主机物理地址(Host Physical Address——HPA) ， EPT消除了与软件管理的影子页表相关的开销。在英特尔的设计中，每个CPU核心可以使用一个单独的EPT Table，这允许来自不同操作系统的多个独立访问并发。



## 3. HyperDbg的设计和构建模块

### HyperDbg调用流程

- 第一部分：HyperDbg在主机上，通过HyperDbg-CLI接受输入，通过Script Engine进行转译，并由Serial Interface，串口，来与客户机进行交互

- 第二部分：事件触发流程根据图中各个子系统来决定

- 第三部分：子系统利用EPT来执行操作

- 第四部分：子系统接受到主机的命令，后端脚本引擎进行转译，然后在kernel mode或user mode中运行

  ![](https://github.com/SapphireStar/HyperDbg-Note/blob/main/pics/Screenshot%202022-12-01%20095732.png?raw=true)

### 事件触发接口

Event：HyperDbg可以监听系统中特定事件的发生，例如某个Syscall发生，某个内存地址内容被更改

Action：当事件被触发后，HyperDbg可以进行一系列的操作，例如Break，Script，Code。Break就是暂停处理器核心，Script允许在事件触发后，调用脚本来进行一系列操作。

Condition：Condition就是使用逻辑表达式来控制事件的触发。

### 操作模式

- Virtual Machine Introspection (VMI) Mode

  该模式允许虚拟机在本地对自己进行自我调试，操作较为方便，

- Debugger Mode

  调试器模式是一种功能强大的操作模式，它允许连接到内核，并暂停系统，以便通过内核和用户指令进入和跨越。调试连接是通过串行电缆或虚拟串行设备进行的。

- Transparent Mode

  以上两种模式都可以在透明模式下运行，透明模式能够增加调试器的隐蔽度，因为某些恶意软件有反调试功能，因此需要增加隐蔽度来绕开反调试功能。

  

## 4. 后端架构（Back-End Architecture）

### Stepping Subsystem 步进子系统

- Step-in: Step-in机制我们应该非常熟悉，在传统的stepin机制中，它们通过设置RFLAGS trap flag来让系统在执行某个特定操作之后停止，从而方便让调试器来读写在这个特定操作之后，寄存器和内存的内容。这非常类似于IDE中的调试功能

- 问题：但传统的stepin机制并不能保证逐行步进，因为其他的CPU核心和进程都可能会执行他们的流程，而调试操作可能会极大地改变这个流程

  如图所示，push rbx操作被设置了flag，在该操作被执行后，进程将暂停，但其他进程或线程依然在运行，其他线程调用了xor rdi rdi之后，rdi的内容被改变，最终输出的结果为错误结果。

  ![](https://github.com/SapphireStar/HyperDbg-Note/blob/main/pics/Screenshot%202022-12-01%20104355.png?raw=true)

  

- 解决方案：HyperDbg使用Monitor Trap Flag (MTF)来实现step-in机制，该方法属于Non-Maskable Interrupts (NMIs)，使得操作必定在一个核上完成，而其他核心会被暂停，从而解决了之前提到的问题，HyperDbg是**第一个**具有这种功能的调试器。

  如下图所示，MTF被提前设置，然后VM exit被触发，HyperDbg通过只在一个核心上继续，并禁用核心的中断来提供细粒度(fine-grained)的步进，来确保，在调试程序时，只执行VM exit之后的指令。

  并且，HyperDbg能够做到从user-mode根据指令（例如SYSCALL指令）进入kernel-mode，在kernel-mode中执行下一条指令，由kernel-mode到user-mode的迁移也由Hyperdbg完成，例如，执行SYSRET或IRET将调试器从内核模式返回到用户模式。

  ![](https://github.com/SapphireStar/HyperDbg-Note/blob/main/pics/Screenshot%202022-12-01%20151824.png?raw=true)

- step-over：HyperDbg的step-over机制与传统的step-in类似，不同的是，传统step-in设置Trap flag，而HyperDbg则将调用指令的长度发送给被调试程序。当调用完成后，触发硬件调试寄存器，并通知调试器下一条指令。由于其他线程或核心也可能触发硬件调试寄存器，因此HyperDbg会忽略来自其他进程id或线程id的Debug Breakpoint Exception。如下图所示：

  ![](https://github.com/SapphireStar/HyperDbg-Note/blob/main/pics/Screenshot%202022-12-01%20153924.png?raw=true)

  

### Hooking Subsystem 劫持子系统

- Hooking：劫持(Hooking)是一种拦截特定事件的行为，在拦截到某个特定事件后，执行一些命令，并将执行流程回滚到entry point。

- 问题：现有的debuggers的劫持系统通常使用DMA(Direct Memory Access)，其行为可以轻易被处于user-mode的软件检测到，因此，恶意软件可以检测到这些debugger。并且，用于记录调试内容的Hardware Debug Register的数量和大小是固定的，这会限制劫持的性能。
- 解决方案：
  -  SYSCALL and SYSRET Hooks：HyperDbg通过触发一个未定义的操作码异常(undefined opcode exception——#UD)，并检验异常的原始原因来实现劫持功能。用户可以对任何system call(SYSCALL)或system call的返回操作(SYSRET)设置劫持，执行脚本。在user-to-kernel或kernel-to-user的模拟过程中，调试器可以在实际执行指令之前监视，执行或修改上下文。

- 实现方式：HyperDbg允许用户监视和操作内存访问，通过同时提供两个扩展页表(EPT)来实现，将其中一个未被修改的EPT展示给运行的程序，保证了劫持的透明性，不易被发现。此外，HyperDbg模拟调试寄存器来增加地址可跟踪性，超过了以前的限制。



- EPT Hidden Hook：HyperDbg有两种EPT Hook

  - 第一种， Classic EPT Hooks：通过向目标机器的内存中注入BreakPoint(0xcc)来实现的，当用户想要访问目标内存地址时触发trap。

  - 第二种，DetoursStyle Hooks：它通过跳转到打了补丁的指令并在回调后将执行流返回到常规例程来更改执行路径。虽然后一种方法有一些灵活性限制(例如，脚本引擎的使用限制、可挂钩地址的范围、页表中的挂钩数量)，但避免昂贵的VM-exit操作可以使挂钩机制大大加快。

    

### Memory Access in VMX-root Mode

对于hypervisor-level 的debugger来说，实现安全的内存访问很有挑战性，因为可能会导致系统的停止或异常，例如，在VMX-root mode中访问超出限制的page。安全的内存访问机制对于检测恶意软件来说非常重要，因此HyperDbg使用了一系列方法来解决VMX memory management的复杂性。

- 解决方案：Hyperdbg为了防止访问无效的page，会先检查page的有效性。hyperdbg使用intel TSX作为解决方案，TSX可以在不进行user-mode和kernel-mode切换的情况下一直异常/错误，hyperdbg利用这种能力来判断对目标地址的操作是否成功执行，来检查page是否有效。使用该方法进行有效性检测比传统的遍历方法要快三个数量级

  使用方法如下图所示

  ![](https://github.com/SapphireStar/HyperDbg-Note/blob/main/pics/Screenshot%202022-12-01%20201606.png?raw=true)



- Reading and Writing Memory：由于从VMX-root模式直接访问用户空间地址的各种安全考虑，HyperDbg被设计成不直接访问内存，而是使用虚拟寻址方法来保留页表项，并将所需的用户模式物理地址映射到内核模式虚拟地址，以实现安全的内存读写访问。此外，PTE中的写启用位消除了对目标地址可写性的检查。
