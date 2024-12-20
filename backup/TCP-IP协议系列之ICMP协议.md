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

但是我们上面有说，ICMP时使用IP协议发送的，IP协议没有实现可靠传输，ICMP协议自身也没有实现可靠传输，所以ICMP消息也是不可靠的。那么基于此，上层协议肯定不会依赖ICMP消息来实现功能，而是作为锦上添花的设计。

> [!NOTE]
> 所以我个人认为，ICMP消息在数据传输流程里面扮演的角色并没有那么重要。

## ping

ping命令应该大部分人都知道，我们通常使用它检查到目标地址的网络通不通，ping就是使用ICMP协议实现的：

![image](https://github.com/user-attachments/assets/acf32bf3-b944-49c5-b10b-0adc18a1f50f)

具体说来，ping使用的是0（回送应答）和8（回送请求）两种消息类型。

![image](https://github.com/user-attachments/assets/a86f0bf3-0944-4cce-9ab6-778329964164)


这两种消息类型的报文格式如下：

![image](https://github.com/user-attachments/assets/a6bc22ab-3f3f-4226-a95f-5dab4d1c3436)

当发送端主机发起一个ping请求时，发送端会发送一个ICMP type 8的消息给接收端主机，
此时，发送端主机会填充一个请求唯一标识，例如进程的PID，用来识别发送这个请求的应用程序；然后填充识别每一次请求的唯一ID，这个序号是一个自增值，用来识别每一次请求。

当接收端主机收到这个ICMP消息后，会回一个type 0的ICMP消息，携带相同的标识符和唯一ID，如果发送端收到了这个回送应答，那就说明网络是连通的。

ping除了能检查网络通不通之外，还可以检查网络质量。

前面说到了ICMP不保证可靠传输，在这里就派上了用场。假如网络是连通的，我们发出去100个回送请求，就应该收到100个回送应答。如果收到的回送请求没有100个，那说明中间发生了丢包，如果丢包很多，说明网络质量不好。另外通过回送请求和回送应答中间的时间差，也能看出来网络延迟的水平。

## traceroute

相对ping，traceroute的实现就更加巧妙了。traceroute的作用时列举出IP数据报从发送端到达接收端途经的每一个网络节点。

traceroute利用的是IP数据报超时以后会回送一个ICMP消息的特性。当IP数据报在网络种转发是，TTL会相应递减，直到变成0，就会被节点丢弃，节点丢弃这个数据报时会给发送端主机回送一个ICMP消息，类型为0，代表网络不可达。
所以traceroute会发多个IP数据报出去，TTL依次+1，接收中间的路由器回送回来的ICMP消息，直到IP数据报送达到目标主机为止，这样就知道了IP数据报中间会经过多少个路由器了，如下图：

![image](https://github.com/user-attachments/assets/27732378-94b3-497e-a1b9-5ad16969e975)

> [!TIP]
> MAC和Linux上面是traceroute，Windows上是tracert，原理都是差不多的，都是接收ICMP超时消息。但是traceroute是使用UDP协议发包，tracert是直接使用ICMP。

> [!TIP]
> 还有一个问题，IP数据报如果正常达到目标主机，是不会触发一个ICMP消息的，那么要怎么知道数据报到达目标主机了呢？traceroute的思路是还是要触发一个ICMP消息，因为它是使用UDP发包，所以会指定一个没人使用的端口号，那么数据到达以后没有应用程序接收，会触发类型为3的ICMP消息，这个时候发送端就知道已经到达目标主机了。
 