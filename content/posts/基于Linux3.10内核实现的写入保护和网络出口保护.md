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

### 写入拦截的实现的两种方式

#### 一：通过内核kprobe探测技术对 `sys_write` 函数插入探针拦截所有写入调用

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

#### 二：通过替换系统调用表，替换sys_write的函数指针指向自定的sys_write函数

```c
asmlinkage long (*original_write)(unsigned int, const char __user *, size_t); // 存储原始的 sys_write 地址
asmlinkage long handler_sys_write(unsigned int fd, const char __user *buf, size_t count) {
    char path[MAX_PATH_LEN];
    char* filepath = get_file_path(fd, path, MAX_PATH_LEN);
    int action = 0;

    // 文件路径获取失败则不处理
    if (NULL == filepath) return original_write(fd, buf, count);
    // 排除一些系统本身需要使用的路径，加快处理速度减少竞争增加稳定性
    if (1 == is_start_with(filepath, "/var/log/")
    || 1 == is_start_with(filepath, "/tmp/")
    ) return original_write(fd, buf, count);
    // 执行规则拦截逻辑
    spin_lock(&config_lock);
    action = protect_rules_filter(filepath, current->comm);
    spin_unlock(&config_lock);
    if (PROCESS_ACTION_DENY == action) return -EPERM;
    return original_write(fd, buf, count);
}
```


### 网络拦截的实现的几种方式

关于网络拦截部分一开始是想通过 `netfilter`， `hook` 到 **NF_INET_POST_ROUTING** 来实现.

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
接着同样的方式通过 `get_task_comm` 获取当前进程名进行规则比对, 如果不在允许列表则直接 `return NF_DROP;`。

实际上这种方式不太合适，遇到了一些问题，比如在 `netfilter` 的钩子函数中， 获取 `current->comm` 当前的进程有时候并不是实际发送网络数据包的进程，经常能获取到内核进程和 `ps` 、`tail` 等一些并不可能发起数据连接的程序，推测在网络数据包处理的上下文中使用 current 宏获取的进程信息可能对应的并非是发送数据的用户空间进程，因为网络包的处理可能在软中断或 tasklet 上下文中进行。在这种情况下，current 可能表示触发中断的进程，或者是空闲进程.所以我更换了处理方式，改为通过 `kprobe` 拦截 `sys_connect` 函数进行处理，将所有不合规的connect请求全部拒绝.

```c
static struct kprobe connect_kprobe = {
        .symbol_name = "sys_connect",
};

```

大概流程如下：

```c
// 规则未加载则直接放行
if (NULL == net_protect_list) return 0;
// 如果读取寄存器失败则直接返回，不继续处理了
if (0 != copy_from_user(&addr_in, (struct sockaddr __user *) regs->si, min((unsigned long) regs->dx, (unsigned long) sizeof(addr_in)))) return 0;
// 跳过已经被重置的请求
if (addr_in.sin_addr.s_addr == htonl(INADDR_LOOPBACK) && addr_in.sin_port == htons(RESET_PORT)) return 0;
// 只处理TCP并且跳过DNS请求
if (addr_in.sin_family != AF_INET || 53 == ntohs(addr_in.sin_port)) return 0;
// 获取进程名
get_process_name(process_name);
// 执行拦截规则
spin_lock(&config_spinlock);
entry = net_protect_list->head;
while (NULL != entry) {
rule = entry->value;
if (NULL == rule) {
    entry = entry->next;
    continue;
}
if (NET_RULE_CHECK(process_name, addr_in.sin_addr.s_addr, ntohs(addr_in.sin_port), rule)) {
    spin_unlock(&config_spinlock);
    return 0;
}
entry = entry->next;
}
spin_unlock(&config_spinlock);
LOG_INFO("[BLOCK-NET|Mode:%d,[%s|%d->%pI4:%u]]", f_mode.netfilter, process_name, current->pid, &addr_in.sin_addr, ntohs(addr_in.sin_port));
reset_tcp_dst(regs); // 重置TCP目的地和端口，禁止连接
```

关于重置函数 `reset_tcp_dst` 的实现是通过改变目的地IP和端口来实现的

```c
void reset_tcp_dst(struct pt_regs *regs)
{
    struct sockaddr_in addr_in;
    addr_in.sin_family = AF_INET;
    addr_in.sin_addr.s_addr = htonl(INADDR_LOOPBACK);
    addr_in.sin_port = htons(RESET_PORT);
    if (0 != copy_to_user((struct sockaddr __user *) regs->si, &addr_in, min((unsigned long) regs->dx, (unsigned long)sizeof(addr_in)))) {
        LOG_ERR("重置TCP目标失败: %s->%d", current->comm, current->pid);
    }
}
```

但是实际上这种方式感觉也有一点不安全，因为用户空间的程序多种多样，修改了用户空间的数据包不确定是否会对应用程序造成什么影响，所以最终我依然是通过替换系统调用表的方式将`sys_connect`的函数指针替换为自定义的函数.

```c
asmlinkage long (*original_connect)(int, struct sockaddr __user *, int); // 存储原始的 sys_connect 地址
asmlinkage long handler_sys_connect(int sockfd, struct sockaddr __user *vaddr, int addrlen) {
    struct sockaddr_in addr;
    struct network_trust_table *rule;
    struct list_node* element = NULL;

    // 规则未加载则直接放行
    if (list_empty(&net_protect_list)) return original_connect(sockfd, vaddr, addrlen);
    // 如果读取寄存器失败则直接返回，不继续处理了
    if (0 != copy_from_user(&addr, vaddr, min_t(unsigned long, sizeof(addr), addrlen))) return original_connect(sockfd, vaddr, addrlen);
    // 只处理TCP并且跳过DNS请求
    if (addr.sin_family != AF_INET || 53 == ntohs(addr.sin_port)) return original_connect(sockfd, vaddr, addrlen);
    // 执行拦截规则
    spin_lock(&config_lock);
    list_for_each_entry(element, &net_protect_list, list) {
        rule = element->value;
        if (NULL == rule) {
            spin_unlock(&config_lock);
            return original_connect(sockfd, vaddr, addrlen);
        }
        if (NET_RULE_CHECK(current->comm, addr.sin_addr.s_addr, ntohs(addr.sin_port), rule)) {
            spin_unlock(&config_lock);
            return original_connect(sockfd, vaddr, addrlen);
        }
    }
    spin_unlock(&config_lock);
    LOG_INFO("[BLOCK-NET|Mode:%d,[%s|%d->%pI4:%u]]", f_mode.netfilter, current->comm, current->pid, &addr.sin_addr, ntohs(addr.sin_port));
    return -ECONNREFUSED;
}
```

### 遇到的一些问题

最开始实现写入保护的时候是想获取到完整的进程二进制路径和进程的参数, 但是在 **Intel CPU** 下从 `current->mm` 中读取参数一直会导致内核崩溃. 这个问题我将内核模块在同样3.10内核版本,同样的虚拟化环境的 **AMD CPU** 中测试一切正常运行, 目前尚未解决该问题, 仅能通过其它方式达到想要功能上的效果.

其次是网络拦截部分, 最开始通过netfilter实现的方式每次将握手包 **DROP** 掉之后, 都会被 **swapper/** 这个进程重发, 导致即便执行了 **DROP** 操作, 数据包依然能发出, 只是感知上会稍微慢一点, 最后只能通过将未经允许的出口流量对应的进程直接 **kill** 掉来达到想要的效果, 为什么 **swapper** 进程会重发, 这个问题目前看可能是由于Linux内核本身的协议栈工作方式引起的。当网络包经过 `netfilter` 钩子时，可能已经在发送队列中，并且已经安排好了在稍后通过网络接口进行发送(仅仅是猜测).

有一个困扰了很久的问题是测试中经常会遇到一段时间后内核崩溃的问题，最终找到问题应该和自旋锁有关系，用户空间通过netlink与内核通信写入规则，在写入规则的时候通过自旋锁保证规则表是绝对正确的，这个时候如果写入规则的函数因为某些原因触发了`sys_write`函数(例如某些系统日志的写入操作)，则会导致sys_write函数也需要获得自旋锁，产生了死锁。

