<!--
 * @Description:
 * @Author: Hongyang_Yang
 * @Date: 2020-08-08 21:10:29
 * @LastEditors: Hongyang_Yang
 * @LastEditTime: 2020-08-08 21:20:12
-->

# 启动顺序

x86 启动顺序 - 第一条指令
CS=F000H, EIP=0000FFF0H

实际地址是：
`Base（基址）+EIP=FFFF0000H + 0000FFF0H = FFFFFFF0H` 。这是 BIOS 的 EPROM 所在地。

当 CS 被新值加载，则地址转换规则讲开始起作用。

通常第一条指令是一条长跳转指令（这样 CS 和 EIP 都会更新）到 BIOS 代码中执行。
