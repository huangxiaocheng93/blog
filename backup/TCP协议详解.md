# TCP是什么？

TCP是TCP/IP协议簇中的**传输层**协议，TCP是面向**有连接的**，**高可靠性**的传输层协议。

为了通过IP数据报实现可靠性传输，需要考虑很多事情，例如数据的破坏、丢包、重复以及分片顺序混乱等问题。如不能解决这些问题，也就无从谈起可靠传输。

TCP通过检验和、序列号、确认应答、重发控制、连接管理以及窗口控制等机制实现可靠性传输。

既然有可靠的传输层协议，那是不是还有不可靠的传输层协议？答案是有的，就是UDP。

# 连接管理

要做到可靠的发送数据，第一步就是建立连接，在建立连接的过程中，通信两端就会做好发送和接收数据的准备。

## 连接
<html><table frame=void style="margin-left: auto; margin-right: auto;"><tr><td>
<!--左侧内容-->
 TCP建立连接需要三次握手，也就是发3个包才能建立连接。

为什么一定要三次握手，举个例子：
开始一个视频通话：

A：你好，请问你能听到我的声音吗？（第一次握手）；

B：可以听到，你能听到我的声音吗？（第二次握手，此时A确定B能听到他的声音，但是B不能确定A能听到他的声音）；

A：可以听到你的声音（现在B可以确定A能听到他的声音）；

其他原因：避免历史重复连接；避免重复创建连接；同步双方的序列号；
</td><td>
<!--右侧内容-->
 ![image](https://github.com/user-attachments/assets/cf3d54fc-c461-4348-8ffe-f467ed7168f8)
</td></tr></table></html>
<html><table frame=void style="margin-left: auto; margin-right: auto;"><tr><td>
<!--左侧内容-->
### 一

client → server发送SYN包：

- SYN标志位置为1；

 - 生成序列号（Sequence Number）：序列号不会从0或1开始，而是在建立连接时由计算机生成的随机数作为其初始值，序列号是指发送数据的位置，每发送一次数据，就累加一次该数据字节数的大小。

这个包不携带待发送的业务层数据。
</td><td>
<!--右侧内容-->
![1423484-20230919222751995-69249474](https://github.com/user-attachments/assets/ab899b76-6d21-4e49-877b-4fd450544b90)
</td></tr></table></html>
<html><table frame=void style="margin-left: auto; margin-right: auto;"><tr><td>
<!--左侧内容-->
### 二

server → client发送ACK + SYN包：

- ACK和SYN标志都置为1；
- server生成自己的序列号；
- 把client生成的序列号+1作为确认应答号，意思是把SYN包当作一个字节接收了；

这个包不携带待发送的业务层数据。
</td><td>
<!--右侧内容-->
![1423484-20230919222756526-1163296370](https://github.com/user-attachments/assets/b3925253-1dc4-41d2-9e5b-2f35a683e7cb)
</td></tr></table></html>
<html><table frame=void style="margin-left: auto; margin-right: auto;"><tr><td>
<!--左侧内容-->
### 三

client → server发送ACK包，作为服务端SYN包的响应：

- ACK标志位置为1；
- 把收到的服务端序列号+1作为确认应答号；

**这个包可以携带业务数据。**
</td><td>
<!--右侧内容-->
![1423484-20230919222802441-1256757742](https://github.com/user-attachments/assets/37540054-97e4-409b-9229-aaf296edd26e)
</td></tr></table></html>
