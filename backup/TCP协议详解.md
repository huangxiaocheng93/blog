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

`Gmeek-html<html><table frame=void style="margin-left: 50; margin-right: 50;"><tr><td>
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

可以看到客户端在发出ACK以后还等了一个2MSL的时间才最终close，原因是害怕服务端发出的包还在传输中没有到，所以会等一个时间，这个时间是包能在网络上生存的最大时间。为什么是2MSL，是因为还要等ack包也到达。
</td><td>

![1423484-20230920232500420-2126674419](https://github.com/user-attachments/assets/8b142c7a-36de-431c-92ac-757e59bf5391)

</td></tr></table></html>`

# 数据发送
## MSS和段

在建立完连接后就可以开始发送数据了，那么一个数据包能发送多少数据呢？

我们把最大能发送的数据大小叫做：**最大消息长度**或者**最大段长度**（MSS：Maximum Segment Size），注意这里的长度指的是有效荷载，去掉所有标头的纯数据长度。

最大段长度的取值和MTU有关系，MTU是网络层的指标在IP协议中详细讲解，当IP数据报的长度超过MTU就会被分片，所以最理想的情况是，最大消息长度正好是IP中不会被分片处理的最大数据长度。

例如：

MTU = 1500字节

IP标头 = 20字节

TCP表头 = 20字节

那么MSS = 1500 - 20 - 20 = 1460字节

段长度的确定是在连接建立的三次挥手中协商产生的，以两边更小的值为准。

> [!NOTE]
> 总结来说：TCP以段为单位发送数据，段的长度就是MSS，MSS由通信双方协商产生，以其中较小的值为准，MSS的确定和MTU有关，正好是IP数据报不需要拆分的长度。

## 窗口控制和重发控制
### 数据发送
<html><table frame=void style="margin-left: auto; margin-right: auto;"><tr><td>
TCP以1个段为单位，每发一个段进行一次确认应答的处理。这样的传输方式有一个缺点。那就是，包的往返时间越长通信性能就越低。

为了解决这个问题，TCP引入了窗口的概率，实际上就是并发的发包，窗口控制逻辑就是用来控制并发的逻辑。
</td><td>

![Image00234](https://github.com/user-attachments/assets/02247b32-cbfb-4150-99d5-01e086719403)


</td></tr></table></html>

### 重发超时

重发超时是指在重发数据之前，等待确认应答到来的那个特定时间间隔。如果超过了这个时间仍未收到确认应答，发送端将进行数据重发。

TCP要求不论处在何种网络环境下都要提供高性能通信，并且无论网络拥堵情况发生何种变化，都必须保持这一特性。为此，它在每次发包时都会计算往返时间（Round Trip Time也叫RTT。是指报文段的往返时间。） 及其偏差（RTT时间波动的值、方差。有时也叫抖动。） 。将这个往返时间和偏差相加重发超时的时间，就是比这个总和要稍大一点的值。

### 窗口

<html><table frame=void style="margin-left: auto; margin-right: auto;"><tr><td>
假如窗口大小是4000，那么客户端就会一次性发送4个段给服务端，这就相当于是单个段发送的4倍速了。

但是实际操作中也不需要等待4个段的确认应答都收到了才能发送下一个段。TCP采用了**滑动窗口**的方式来处理这段逻辑。
</td><td>

![Image00235](https://github.com/user-attachments/assets/32d93c2e-1c98-4ec5-8e9d-d6cc40935faa)

</td></tr></table></html>

### 窗口控制

<html><table frame=void style="margin-left: auto; margin-right: auto;"><tr><td>
TCP采用了滑动窗口做窗口控制，

收到确认应答的情况下，将窗口滑动到确认应答中的序列号的位置。这样可以顺序地将多个段同时发送提高通信性能。
</td><td>

![Image00236](https://github.com/user-attachments/assets/ddcd02af-a69b-4d46-bf9b-a26707ad376c)

</td></tr></table></html>

### 重发控制

<html><table frame=void style="margin-left: auto; margin-right: auto;"><tr><td>
正常来说，没有收到确认应答的包都应该重发，但是按照TCP顺序接收的特性，如右图的场景，序列号为1001的确认应答丢失了，但是2001的确认应答收到了，实际上间接的说明了第一个包是收到了的。

所以这里实际上并不需要每一个确认应答都收到。
</td><td>

![Image00237](https://github.com/user-attachments/assets/67ab2e28-36e2-4434-9543-d229039e8e45)

</td></tr></table></html>

### 高速重发机制

<html><table frame=void style="margin-left: auto; margin-right: auto;"><tr><td>
一个反面的例子，如果1000~2001包未能成功发送，那么服务端就会一直响应1001确认应答，客户端就会知道

1000~2001这个包丢了，需要重发。

这个机制也叫做高速重发机制。
</td><td>

![Image00238](https://github.com/user-attachments/assets/68be6fae-8402-441d-8842-558e9ce4e697)

</td></tr></table></html>

## 流量控制

<html><table frame=void style="margin-left: auto; margin-right: auto;"><tr><td>
前面有说到窗口，那么窗口的大小是怎么来的呢？

答案是服务端决定的，TCP表头有一个字段用于传递窗口大小，客户端要根据服务端返回的窗口大小来调整发送数据的评率；

服务端的窗口最大主要和缓存的大小有关系，另外处理效率也会影响到窗口的恢复速度。

通过这种机制，服务端就可以保证自己的缓存不会被打爆；
</td><td>

![Image00239](https://github.com/user-attachments/assets/5101e39f-1dc0-407b-b5a8-e3966caae626)

</td></tr></table></html>