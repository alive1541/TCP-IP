# TCP-IP
图解TCP／IP连接过程
## 前言
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;我们知道Ip层包裹着tcp报文段把它从源Ip运送到目的Ip，如果过程中出现差错（16位的Ip检验和错误），Ip协议会直接丢弃该数据报并且不生成差错报文。这种情况tcp会发现数据丢失并进行重传。
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;这篇文章想探讨一下TCP协议是通过什么方式做到这些的，曾经做过设计师的我忍不住抄起老本行画两张图。
## IP、UDP、TCP差异
![首部.jpg](http://upload-images.jianshu.io/upload_images/10099568-8c37c31b5b9efcf3.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;IP是网路层协议，TCP和UDP都是运输层协议，他们都是基于IP协议传递数据。接下来我们通过几个重要首部分析一下它们的异同。
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;三个协议首部中都有16位检验和，但是，IP协议首部中的检验和只覆盖IP首部，而TCP和UDP首部中的检验和是包含所有数据的，也就是说IP协议不关心数据是否正确，而TCP和UDP关心。
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;IP首部中最重要的是32位源IP和目的IP地址，其它首部都是配合它顺利的把数据从源IP传到目的IP，TCP和UDP负责标记对应端口，那么TCP和UDP的区别有是什么呢？
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;我们先看UDP，它只有4个首部，前两个确定端口，后两个保证数据的准确。IP协议把数据传递到目的IP地址的机器上，UDP通过首部中的长度和检验和校验数据，如果校验没有通过，那么UDP协议会直接丢弃数据，而不会向目的IP和端口发送消息，所以说UDP协议是面向数据报的。
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;UDP的缺点就是：我们不能保证对方一定收到了我发送的数据。但是从首部中很容易能看出它的优点:就是轻量、速度快。
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;在看TCP，它除了关心数据之外明显还关心更多的东西，其中最重要的就是关心连接的可靠性。下面我们将主要通过TCP连接和断开过程看看这些首部都是什么作用。
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;TCP首部中有6个标志比特位，它们默认都是0，需要时设置为1，通过它们完成询问和应答。它们分别表示的含义如下：
> URG： 紧急指针
> ACK： 确认序号有效
> PSH： 尽可能快地将数据送往接收进程
> RST： 重建连接
> SYN： 同步序号用来发起一个连接
> FIN： 发端完成发送任务

## 连接过程和状态
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;我画了一张图来描述TCP连接的过程和状态变化：
![连接过程.jpg](http://upload-images.jianshu.io/upload_images/10099568-9659ea2517b445b3.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
w=1242&h=1000&f=jpeg&s=166108)
#### 一般情况(图中紫色和绿色部分)
##### 三次握手
###### 第一条紫色箭头（从上到下）：
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;服务器端默认处于<code> LISTEN </code>状态监听着某个端口。当客户端想要连接这个端口的时候，客户端首先会随机生成一个初始序号(图中的j),首部中的32位序号(以下简称序号)就变成j+1,同时首部中的SYN由0置为1。客户端发送完带有这些首部的TCP段后状态后<code> SYN_SENT </code>。
###### 第一条绿色箭头：
&nbsp;&nbsp;&nbsp;&nbsp;当服务器收到客户端第一个带有SYN的TCP段的时候，客户端由<code> LISTEN </code>状态变为<code> SYN_RCVD </code>状态，并发送一段数据，其中首部ACK置为1，32位确认号(以下简称确认号)为客户端发过来的序号(j)+1，含义就是客户端的数据已经收到。同时SYN也会置为1，序号为k。含义是我也要与你建立连接。
###### 第二条紫色箭头：
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;当客户端收到服务器的ACK后，状态变更为<code> ESTABLISLISHED </code>，同时发送数据，首部ACK为k+1，含义是服务器的数据已经收到。这时客户端已经建立连接，而服务器端是否能建立连接取决于它能否收到当前客户端发送的这段带有ACK的数据。如果服务器收到的话，它的状态也会变成<code> ESTABLISLISHED </code>。
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;到此，连接建立，可以继续交换数据。
##### 四次挥手
###### 第三条紫色箭头：
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;在不考虑断电等特殊情况下，主动关闭的一方(图中为客户端但不总是)发送FIN+序号m，状态变更为<code> FIN_WAIT_1 </code>。
###### 第二条绿色箭头：
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;被动关闭的一方(图中为服务器但不总是)收到后状态变更为<code> CLOSE_WAIT </code>。同时里立即发送确认号m+1,头部ACK置为1，这时服务端状态变更为<code> LAST_ACK </code>，很容易理解，这是最后一次确认，客户端收到ACK后，状态变更为<code> FIN_WAIT_2 </code>。
###### 第三条绿色箭头：
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;服务器<code> LAST_ACK </code>后还可能继续做其它的操作，做完之后，服务器也会发送FIN+序号n。
###### 第四条紫色箭头：
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;客户端收到FIN后，状态变更为<code> TIME_WAIT </code>，同时发送确认信息ACK n+1，服务器收到ACK后，状态变更为<code> CLOSE </code>。
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;到此，连接断开。
#### 为什么挥手是4次而不是3次?
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;挥手的时候有个问题，为什么ACK和FIN不能一起发送呢？这是由TCP的半关闭(half-close)造成的。TCP连接是全双工(即数据在两个方向上能同时传递)，因此每个方向必须单独地进行关闭。
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;收到一个FIN只意味着对方不会再向我发送数据了，但是TCP仍然支持我继续向对方发送数据（当然，在实际的应用中，直有很少的程序这样做），如果我们用Wireshark等工具抓包时也经常会看到挥手只有3次的情况。
#### TIME_WAIT旁边的2MSL是什么?
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; <code> TIME_WAIT </code>状态也称为2倍MSL等待状态。MSL是TCP的最大报文生存时间。当一个TCP主动关闭并发送最后一个ACK(图中右下角最后一条紫色线)后，必须在<code> TIME_WAIT </code>状态等待两倍的MSL时间，防止对方没有收到这个ACK然后重发FIN。所以一去一回最多就是两倍时间。
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;操作系统默认240s(也就是2倍的两分钟)后会关闭处于<code> TIME_WAIT </code>的连接，在这之前端口一直会被占用。
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; 在Linux服务器上可以通过变更/etc/sysctl.conf文件去修改该缺省值（秒）：net.ipv4.tcp_fin_timeout=30。但这通常也会出现一些问题。
#### 同时打开(图中紫色和橙色部分)
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;两个应用程序彼此同时打开的可能性很小。这时双方都是客户端也都是服务器。具体流程和一般情况一样，只不过握手的时候是4次而不是3次。TCP协议认为这种情况是建立一条连接而不是两条。
#### 同时关闭(图中紫色和橙色部分)
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;为了方便我把同时打开和同时关闭画在了一起，其实同时打开不一定同时关闭。不同时打开也有可能同时关闭。
## 总结
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;对于前端工程师来讲，了解TCP协议能够帮助我们更好的理解并使用HTTP协议。也能够帮助我们对网络进行更加细致的优化。我的理解也很有限，希望能够和大家共同讨论。

## 参考资料
- 《Tcp/ip详解-卷一》
- 《图解Tcp/ip》
