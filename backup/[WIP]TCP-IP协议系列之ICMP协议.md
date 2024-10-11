# TCP/IP协议系列

[TCP/IP协议系列之网络层](https://blog.npex.top/post/21.html)

# ICMP是什么
ICMP全称是**互联网控制报文协议**（Internet Control Message Protocol）ICMP协议工作在网络层，是IP的辅助协议，也是控制协议。

> [!TIP]
> ICMP是Control Message Protocol，我个人理解**控制**的点在于ICMP并不是设计用来承载业务数据传输的，不携带任何业务数据，而是为了实现网络控制，所以他是一个控制协议。

# ICMP是怎么工作的

## ICMP的报文格式

ICMP的报文格式如下：

![image](https://github.com/user-attachments/assets/fb0d56fd-4c07-492e-bc74-da64b23aef26)

ICMP的报文格式想对IP协议来说简单太多，总共就4个字段：类型、代码、校验和、Data。

其中最重要的就是**类型**和**代码**，这两个字段确定了ICMP消息的类型。校验和无需多说了。

> [!TIP]
> 需要注意的是，这里的**Data**和其他协议不一样，ICMP消息的Data携带的是对应**类型**和**代码**的补充数据，而不是上层协议传递过来的数据。

## 类型和代码

ICMP消息的类型如下：

![Image00192](https://github.com/user-attachments/assets/1dfce6c6-2f08-47cf-941f-95611efc2382)

ICMP消息的**类型**表示的消息的大类，例如**目标不可达**表示IP数据报无法送达目标地址，但是具体什么原因并不在类型字段里面给出，而是在**代码**字段里面做了具体的分类，例如类型**目标不可达**的**代码**如下：

![Image00193](https://github.com/user-attachments/assets/85448aa6-ea80-49bf-a9b7-f510d13934c0)

可以看出来，不可达的原因有可能是网络不可达或者是主机不可达等等。

## ICMP消息的传输

ICMP使用IP协议发送，由于IP协议的设计是“尽力而为”，IP数据报是否有成功送达接收端，发送端是完全不可知的。ICMP协议自己本身也没有可靠传输的设计，所以ICMP消息也同样不保证一定可以送达。


# ICMP协议的应用

## IP数据报发送失败的通知

这是其中一个很重要的应用，当某个IP数据报在某个网络节点要被丢弃时，这个网络节点就会发送一条ICMP消息给发送端主机。节点会在类型和代码字段中填写IP数据报无法送达的原因，例如：不可达、超时等等。发送端主机收到这条ICMP消息以后会上报给上层协议，上层协议就会采取相应的处理。

 