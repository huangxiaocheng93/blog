# ARP协议是什么

在网络层，唯一地址是IP地址，所以在网络层，只能知道下一个节点的IP地址。但是只知道IP地址还不够，因为我们如果要调用数据链路层发送数据，必须要知道下一个节点的MAC地址才行，ARP协议就是来解决这个问题的。

ARP全称是Address Resolution Protocol，是网络层协议，是IP协议的辅助协议，它的职责是通过IP地址找到节点的MAC地址。ARP虽说是网络层协议，但它只能工作在同一个数据链路内，个人感觉它更像是网络层和数据链路层中间的胶水层。

> [!IMPORTANT]
> **IP数据报所在的当前节点和下一个节点一定是在同一个数据链路内的**，这个点不知道好不好理解。因为说有的数据最终都是要通过数据链路层发送出去的，而数据链路层只能传输数据只能是在同一个数据链路内，所以说当前节点和下一个节点一定是在同一个数据链路内的。路由器是可以连接多个数据链路的，所以IP数据报就是从一个数据链路跳到另一个数据链路，最终找到目标主机。

# ARP是怎么工作的

ARP协议就是来解决只知道IP地址，不知道MAC地址的问题的。ARP协议使用数据链路层的广播功能，在数据链路内广播ARP请求，ARP协议会携带下一个节点的IP地址，当对应地址的节点接收到请求后，会响应这个请求，在响应中填上自己的MAC地址，这样发送端收到响应后就知道下一个节点的MAC地址啦。

ARP协议定义了两种类型的包，ARP请求包和ARP响应包，包格式如下：

![image](https://github.com/user-attachments/assets/bcacaedd-b077-451f-9130-62f8bba32775)

目的MAC地址在请求包中是不存在的，因为不知道，所以用0填充。

还是借用一张《图解TCP/IP》的图：

![Image00187](https://github.com/user-attachments/assets/ee8a4fa9-10f2-4a81-88e0-153541c5f412)

**不知道目的MAC地址，ARP协议是怎么把包送到目标主机的呢？**

是用了数据链路层的广播功能，数据链路层的所有主机都会收到这条消息，IP地址匹配的主机就会发送一个ARP响应包，响应包中就会携带自己的MAC地址。

## ARP表

如果我们向一个节点发送过一次包，那么就极有可能会继续发送多次，如果每次发送都需要发ARP包询问MAC地址的话，那就有点太费时了，所以所有网络节点都会维护一个IP地址和MAC地址的映射表缓存，叫**ARP表**。下一次发送的时候就可以直接查表找到目标节点的MAC地址。

> [!NOTE]
> 但是IP地址时有可能变化的，所以ARP表还会有维护的逻辑。

