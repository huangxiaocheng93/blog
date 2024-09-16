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

 ![image](https://github.com/user-attachments/assets/cf3d54fc-c461-4348-8ffe-f467ed7168f8)
</td></tr></table></html>
### 一
<html><table frame=void style="margin-left: auto; margin-right: auto;"><tr><td>
<!--左侧内容-->

client → server发送SYN包：

- SYN标志位置为1；

 - 生成序列号（Sequence Number）：序列号不会从0或1开始，而是在建立连接时由计算机生成的随机数作为其初始值，序列号是指发送数据的位置，每发送一次数据，就累加一次该数据字节数的大小。

这个包不携带待发送的业务层数据。
</td><td>

![1423484-20230919222751995-69249474](https://github.com/user-attachments/assets/ab899b76-6d21-4e49-877b-4fd450544b90)
</td></tr></table></html>
### 二
<html><table frame=void style="margin-left: auto; margin-right: auto;"><tr><td>
<!--左侧内容-->

server → client发送ACK + SYN包：

- ACK和SYN标志都置为1；
- server生成自己的序列号；
- 把client生成的序列号+1作为确认应答号，意思是把SYN包当作一个字节接收了；

这个包不携带待发送的业务层数据。
</td><td>

![1423484-20230919222756526-1163296370](https://github.com/user-attachments/assets/b3925253-1dc4-41d2-9e5b-2f35a683e7cb)
</td></tr></table></html>
### 三
<html><table frame=void style="margin-left: auto; margin-right: auto;"><tr><td>
<!--左侧内容-->
client → server发送ACK包，作为服务端SYN包的响应：

- ACK标志位置为1；
- 把收到的服务端序列号+1作为确认应答号；

**这个包可以携带业务数据。**
</td><td>

![1423484-20230919222802441-1256757742](https://github.com/user-attachments/assets/ec100aa1-9799-4493-9bd4-5f92ee2d2951)

</td></tr></table></html>

## 断开

MSL是Maximum Segment Lifetime，即报文的最大生存时间，它表示报文在网络中存在的最长时间。超过此时间，报文将被丢弃。因为TCP协议是基于IP协议的，IP头部有一个TTL字段，它表示数据报可以经过的最大路由数。每经过一个路由器，TTL值就减1。当TTL值为0时，数据报将被丢弃，并且发送ICMP报文通知源主机。

<html><table frame=void style="margin-left: auto; margin-right: auto;"><tr><td>
断开为什么要发4个包？

四次挥手实际上就是把三次握手中的第二次握手的ack+syn拆开了。

当客户端发送fin包时，服务端会回ack表示接收到了这个包，此时客户端就不会在发送数据了，但是不代表服务端没有数据需要发送，所以服务端不会立即送出fin包，而是等自己的包发完了才送出fin包。

还是用视频通话举例：

A：我没什么要说的了？（标识A的话已经说完了，后面没话要说了，但是这时候还不能挂电话，因为B可能还有话说）；

B：好的（此时A确定B听到他说的了）

B：（继续说话）

B：我话说完了

A：好的（此时B确定A听到他说的了）

A&B：挂电话
</td><td>
![1423484-20230920232500420-2126674419](https://github.com/user-attachments/assets/fc05ff01-6a97-41bd-9985-02296dee7ba6)

可以看到客户端在发出ACK以后还等了一个2MSL的时间才最终close，原因是害怕服务端发出的包还在传输中没有到，所以会等一个时间，这个时间是包能在网络上生存的最大时间。为什么是2MSL，是因为还要等ack包也到达。

</td></tr></table></html>
