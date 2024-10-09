# TCP/IP协议系列

[TCP/IP协议系列之网络层](https://blog.npex.top/post/21.html)

# ICMP是什么
ICMP全称是**互联网控制消息协议**（Internet Control Message Protocol）ICMP协议工作在网络层，是IP的辅助协议。由于IP协议的设计是“尽力而为”，IP数据报是否有成功送达接收端，发送端是完全不可知的。所以ICMP就是在IP数据报由于某些原因被节点丢弃时，会回送一个ICMP通知给发送端，告知发送端发送失败，通知附带了发送失败的原因。

值得注意的是，ICMP并不是为了补齐IP协议的短板或者缺陷，IP协议设计如此，并不是缺陷。ICMP提供了一个类似callback的机制来告知发送端主机发送失败，而且这个callback消息并不是IP协议处理，而是直接交给上层协议跟进实际情况处理。


# ICMP如何工作

ICMP协议是基于IP协议的，IP协议的设计思路是“尽力而为”，所以ICMP协议