---
title: "基于Linux3.10内核实现的写入保护和网络出口保护"
date: 2023-06-10T22:52:12+08:00
draft: false
---

## 概述

针对于Linux系统中用户权限分配混乱,以及不明原因的黑客提取root权限,危害到部分重要用户的数据, 实现了几种功能:

 - 用户绑定写入目录
 - 进程绑定写入目录
 - 写入目录白名单
 - 进程网络出口IP和端口控制

此文档仅作为备注,记录一下实现过程及部分细节和问题.

### 写入拦截的实现

通过内核kprobe探测技术对 `sys_write` 函数插入探针拦截所有写入调用. 

```c
static struct kprobe write_kprobe = {  
	.symbol_name = "sys_write",  
};
```
在探针函数内部实现以下流程:

 1. 获取 `pt_regs *regs` 寄存器的 `di` 也就是 `sys_write` 函数的第一个参数 `fd` 文件描述符, 通过文件描述符读取要写入的路径
 2. 通过 `get_task_comm` 函数从 `current` 中获取当前进程名进行规则比对. 仅允许 ** 绑定好关系的用户 / 白名单内的目录 / 进程绑定的目录 ** 执行写入操作, 其它操作全部拒绝
 3. 需要注意的是, 写入操作的拒绝不要通过在探针函数内 **return** 错误码来实现, 基本上内核是会炸的. 替代实现的方式可以将 `sys_write` 函数的 `buffer` 和 `size` 重置, 这样可以做到拒绝写入操作, 同时兼顾了内核的稳定性.
 
 ```c
 void reset_buf_to_empty(struct pt_regs *regs)  
{  
	regs->si = EMPTY_BUFF_PTR;  
	regs->dx = 0;  
}
 ```

### 网络拦截的实现

关于网络拦截部分可以通过 `netfilter` 来 `hook` 到 **NF_INET_POST_ROUTING** 来实现.

首先拿到 `sk_buff *skb` 数据后我们可以增加一个校验实现仅处理握手包, 这样我们只需要在需要的时候拒绝掉握手包即可, 减少处理的数据量.
```c
// 忽略非TCP包  
ip_header = ip_hdr(skb);  
if (!ip_header || ip_header->protocol != IPPROTO_TCP) {  
	return NF_ACCEPT;  
}  
// 只关注TCP握手包，如果不是握手包则直接放行，提高处理速度  
tcp_header = tcp_hdr(skb);  
if (skb->len != ip_header->ihl * 4 + tcp_header->doff * 4) {  
	return NF_ACCEPT;  
}
```
接着同样的方式通过 `get_task_comm` 获取当前进程名进行规则比对, 如果不在允许列表则直接 `return NF_DROP;`

### 遇到的一些问题

最开始实现写入保护的时候是想获取到完整的进程二进制路径和进程的参数, 但是在 **Intel CPU** 下从 `current->mm` 中读取参数一直会导致内核崩溃. 这个问题我将内核模块在同样3.10内核版本,同样的虚拟化环境的 **AMD CPU** 中测试一切正常运行, 目前尚未解决该问题, 仅能通过其它方式达到想要功能上的效果.

其次是网络拦截部分, 每次将握手包 **DROP** 掉之后, 都会被 **swapper/** 这个进程重发, 导致即便执行了 **DROP** 操作, 数据包依然能发出, 只是感知上会稍微慢一点, 最后只能通过将未经允许的出口流量对应的进程直接 **kill** 掉来达到想要的效果, 为什么 **swapper** 进程会重发, 这一点目前也未解决.


