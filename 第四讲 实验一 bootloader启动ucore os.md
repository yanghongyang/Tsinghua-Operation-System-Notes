<!--
 * @Description:
 * @Author: Hongyang_Yang
 * @Date: 2020-08-08 21:10:29
 * @LastEditors: Hongyang_Yang
 * @LastEditTime: 2020-08-09 23:42:36
-->

# 启动顺序

## x86启动顺序 - 寄存器初始值
当x86加电后，所有的寄存器在初始情况下都会有缺省值。

最重要的是启动地址，CS和EIP的结合确定启动地址。

## x86 启动顺序 - 第一条指令
`CS=F000H, EIP=0000FFF0H`

一开始启动的是实模式，这种情况下的寻址，按照**实模式的寻址方式**。
实际地址是：
`Base（基址）+EIP=FFFF0000H + 0000FFF0H = FFFFFFF0H` 。这是 `BIOS` 的 `EPROM` （这是只读的地址）所在地。

当 `CS` 被新值加载，则地址转换规则讲开始起作用。

通常第一条指令是一条长跳转指令（这样 `CS` 和 `EIP` 都会更新）到 `BIOS` 代码中执行（也就是会跳到1M的寻址空间）。

## x86启动顺序 - 处于实模式的段

- 段选择子（Segment Selector）：CS，DS，SS，...
- 偏移量（Offset）：EIP

## x86启动顺序 - 从BIOS到Bootloader
- BIOS进行了底层的初始化操作
- BIOS加载存储设备（比如软盘、硬盘、光盘、USB盘）上的第一个扇区（主引导扇区，Master Boot Record，or MBR）的512字节到内存的0x7c00......
- 然后跳转到 0x7c00处的第一条指令开始执行，使之可以执行扇区中的代码（这个扇区里的代码被叫做bootloader）

## x86启动顺序 - 从bootloader到OS
- bootloader中的主要执行代码实际上不足512字节
- bootloader做的事情
  - 使能（Enable）[保护模式（protection mode）& 段机制（segment-level ]protection）：从实模式切换到保护模式，为后续操作系统的执行做准备（从1M的寻址到4G的寻址）
  - 从硬盘上读取kernel in ELF格式的ucore kernel（跟在MBR后面的扇区）并放到内存中固定位置
  - 跳转到ucore OS的入口点（entry point）执行，这时控制权到了ucore OS中

## x86启动顺序 - 段机制
段寄存器(Segment Registers)起到了指针的作用，指向了段描述符。

段描述符描述了一个**段落的起始地址和它的大小**。

因此，可以通过CS这个寄存器找到ucore代码段的起始地址在什么地方。

由于后面有页机制，存在一定地重叠。因此在此映射关系较为简单。

在一个段寄存器中，会保存一块区域叫做段选择址（Index）,Index会查找段描述符。接着加上EIP就是段落的起始地址（实际上是一个线性地址）。在没有启动页机制的情况下，线性地址等同于物理地址。

全局描述符表（GDT，简称段表）由bootloader建立。可以描述好段描述符表的大小和位置，让CPU找到段表找到起始地址。

通过内部的GDTR寄存器保存地址信息，使CS、DS、SS等寄存器和GDT表建立对应关系。

ucore中，基址都是0，段的长度都是4G。

*如何产生全局描述表的索引值？*

段寄存器大致有16位，其中高13位放的是GDT的index。还有两位存放段当前的优先级的级别（4个特权级）。

ucore中没有使用LDT，只是用了GDT。

## x86启动顺序 - 使能保护模式
建立好映射关系后，需要使能保护模式。
- 使能保护模式（protection mode），bootloader/OS要设置CR0的bit 0(PE)
- 段机制（Segment-level protection）在保护模式下是自动使能的

## x86启动顺序 - 加载ELF格式的ucore OS kernel
ucore中的elf格式文件名为kernel

首先需要解析kernel文件的信息。

elf有elf头（ELFheader）`elfhdr`，指明了`Program Header`（程序段的头）的信息。程序段里包含了代码段、数据段等等。

例如，Program Header的phoff（Offset of the program header tables）、phnum（number of program header tables）。有了这两个信息，就可以进一步查找Program Header这个结构。

在Program Header结构中，发现了va（虚地址）、memsz（代码段多大）。这两个地方可以用来描述存放ucore代码段或数据段的相关信息。offset（偏移量）说明了文件从何处开始**读**代码段到内存中。

**读的是什么呢？**

读的是磁盘扇区。

