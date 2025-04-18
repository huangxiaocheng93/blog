# 网络是什么？

网络应该是现代社会离不开的东西了，每个人都依赖网络做很多事情，例如打电话、发微信、购物等等。网络就是通过某种传输介质加上特定的协议，将所有加入网络的设备连接起来，让设备之间可以直接通信的一种技术。网络的出现本质上是为了解决多设备之间数据传输的问题，当网络联通以后，我们就可以轻松的访问到千里之外的数据。

# 网络的形态

![image](https://github.com/user-attachments/assets/f0db2e94-0798-426f-b731-7cef3cf69947)

如上图网络是按照一种层级结构来组织的，大概由如下几种范围：
- **局域网**（Local Area Network，LAN）：局域网是范围最小的网络，例如家庭网络，公司内网络或者学校内网络。特点是局域网内的主机可以通过内网地址互相通信，但是无法与局域网外的主机互相通信；
- **广域网**（Wide Area Network，WAN）是把多个局域网或者广域网连起来的网络，广域网实现了更大范围内的主机之间的通信。如图所示的局域网之外的无论是城域网还是骨干网都可以认为是广域网；

当广域网连接的范围越来越大，越来越多的局域网和广域网加入，甚至和其他国家的网络连接，形成国家之间的广域网，就逐渐形成了现代**互联网**的形态。互联网也可以认为是范围最大的广域网。

> [!TIP]
> 城域网和骨干网更多的是从网络服务商（ISP）的角度来看的，并不属于网络的基础知识。例如把整个广州市连接起来的网络可以认为是城域网，城域网要进一步与其他城域网互通或者和国外的网络互通，就要再往上接入**骨干网**。城域网并不是一个地域概念，不能认为广州市就是一个城域网，佛山市就是另一个城域网，城域网的“城”字我觉得可以认为是一个形容词，描述的是这个这个级别的广域网连接的一个大概范围。

# 传输介质

> [!TIP]
> 没有特指某种设备的情况下，后续网络设备统一称为主机。

网络的本质就是把数据从一台主机传输到另一台主机，那我们就需要一些介质来作为数据的载体。网络中常用的载体有下面这些：

## 同轴线
<html><table frame=void style="margin-left: auto; margin-right: auto;"><tr><td>
同轴线是常用的传输介质之一，同轴线只有中间一根导线用来传递信号，所以同轴线只能工作在半双工模式下，同轴线的优点的线缆够粗，所以电阻就小，信号传递损耗就小，同时还有一层屏蔽层，信号受到的干扰也会比较小。
</td><td>

![image](https://github.com/user-attachments/assets/3d1f5d7d-827a-4ffb-8df9-09a4894a51a2)

</td></tr></table></html>

## 双绞线
<html><table frame=void style="margin-left: auto; margin-right: auto;"><tr><td>
双绞线是常用的传输介质之一，如今我们常用的网线都是双绞线。双绞线有4对共8根线，它们两两一股“绞”在一起。双绞线的优势在于线多，能传输的信号就多，所以双绞线可以工作在全双工模式下。但同时双绞线细，所以传输距离有限制，虽然高规格的线外层也有屏蔽层，但是内部的8根线也会互相干扰。
</td><td>

![image](https://github.com/user-attachments/assets/76f05251-e3a4-42a1-82d5-04f5a901ddb2)

</td></tr></table></html>

## 光纤
<html><table frame=void style="margin-left: auto; margin-right: auto;"><tr><td>
光纤使用光作为传播媒介，光纤的优势很多，例如：
- 光的传播速度快；
- 不带电，所以不受电磁干扰，功耗也小，安全性也更高；
- 传输损耗小，可以传播几十上百公里的距离；
当然光纤也不全都是优点，缺点也有，例如：
- 造价更高，包括线材和设备；
- 更不易维护，弯折时易断，普通金属线就没有这个问题；
</td><td>

![image](https://github.com/user-attachments/assets/3a0055f2-5d1b-48b4-8e1c-3173d16394f4)

</td></tr></table></html>

## 电磁波
无形的电磁波对比有型的线材来说，最大的优势就是不用接线，这也使得有线电话拜托了线的束缚，变成了我们现在用的手机。缺点也是没有线，在空气中传播，会受非常多因素的影响，例如建筑物的墙壁会削弱信号，电梯的金属轿厢会屏蔽型号，甚至天气不好也会影响信号传播。

> [!TIP]
> 通过双绞线，光纤等介质连接的网络可以称之为有线网络，通过电磁波作为介质的网络可以称为无线网络。

# 协议

前面说过网络就是一台主机通过某种介质加某种协议把数据传送给另一台主机，前面了解了介质，现在看看协议。

以有线网络为例，常用的传输介质有双绞线（我们常用的网线），光纤等。很显然，双绞线只能传递电信号，光纤只能传递光信号，如果我们要通过这种介质来传递数据，应该会是如下的步骤：

![image](https://github.com/user-attachments/assets/ab26e392-698c-4e6d-8abf-38d3c13b0dee)

这中间，数据转换所遵循的规则就是**协议**。所以发送端和接收端必须使用同一套转换规则才有可能正常的通信。

举个比较通俗的例子来理解协议，人类所使用的语言就是一种协议，交流的双方（两端主机）如果都会说中文（中文协议）或者英文（英文协议），那么就可以沟通。如果说双方会的语言不一样，一方说英语，一方说中文（协议不通），那么显然是无法沟通的。

> [!TIP]
> 所以这里就能看出协议标准化的重要性了。

# OSI七层网络模型

随着网络的发展，网络中的设备也是越来越多，也越来越复杂，协议也越来越多，所以ISO组织为网络做了分层，也就是**OSI七层网络模型**。分层的目的是为了解耦和职责分离，这和代码编码种的分层道理是一样的。

上下层之间交互使用**接口**，这样就可以使得各层直接职责分离，不需要关注其他层的具体实现；同一层之间的交互使用的就是**协议**，OSI七层网络模型（图片来自于《图解TCP/IP》）：

![Image00022](https://github.com/user-attachments/assets/3a038b99-c758-472b-bcb0-a7b6d4bb7568)

那么数据在这七层种是如何流转的呢？（图片来自于《图解TCP/IP》）：

![Image00023](https://github.com/user-attachments/assets/d2caede2-4235-45da-88fe-0bf60c504ed8)

可以看出，数据在发送端从应用层开始，一层层往下传递，每一层都按照自己的协议处理数据，最终到达接收端，接收端的处理顺序刚好反过来，从物理层开始，一层一层用对应的协议解析，最终就到应用层就解析出实际的数据了。

# TCP/IP的网络模型

在其他的领域，一般都是先有标准，再有实现，例如先有Java虚拟机规范，然后各个虚拟机厂商根据规范来开发各自的虚拟机。网络层的规范就是OSI七层模型。

但是TCP/IP协议簇是先有了协议，才总结出了TCP/IP的网络模型。为什么TCP/IP协议簇没有按照OSI模型设计呢？我查完资料总结出来的原因大概有如下几个：

1. TCP/IP协议出现的时间要早于OSI参考模型的制定；
2. OSI模型基于理论，分层多，会更复杂和繁琐，TCP/IP基于实践，分层更少，更加简单和实用；
3. OSI参考模型制定出来的时候TCP/IP已经事实占领了市场，所以已经是TCP/IP说了算了。

> [!TIP]
> 但是话说回来，TCP/IP模型和OSI参考模型实际上相似度很高，TCP/IP模型可以看做是OSI参考模型的简化版本。


下面是TCP/IP模型和OSI模型的对比：

![Image00072](https://github.com/user-attachments/assets/aaa91c4d-68f7-40b6-8778-1be5eb49ae02)

可以看出来传输层及以下，OSI模型和TCP/IP模型是对齐的，实际功能也基本是对齐的，TCP/IP模型合并了OSI模型的应用层、表示层和会话层，统一成了应用层。

> [!TIP]
> 在我个人看来，TCP/IP模型会更加合理，OSI模型中的应用层、表示层和会话层这三层也是我个人认为最难理解，边界最模糊的部分。

# 总结

现在我们了解到了，网络是什么，网络的形态，根据网络的范围对网络的划分，以及网络传输数据使用的介质。
什么是网络协议，ISO组织定义的OSI七层模型以及TCP/IP的五层模型。
