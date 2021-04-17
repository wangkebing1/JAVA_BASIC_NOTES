## 三次握手

TCP的三次握手是为了在两台计算机之间建立一条全双工的通道的连接，会占用两个计算机之间的通信线路。直到背一方或者双方关闭为止。TCP报头里面的拓展字段有 6 bit 的FLAG位。其中跟握手相关的有`SYN`(Synchronized Sequence Number)用作建立连接的同步信号,`ACK`(Ackonwledgement),用来对收到的数据进行确认`FIN`表示后面没有数据，表示结束。

最开始的的时候客户端和服务器都是`LISTENING`。

- **第一次握手**建立连接的时候，客户端发动一个数据包将SYN位置为1，表示希望建立连接，并且初始化一个序列号`seq`发送给server，此时客户端的状态进入`SYN_WAIT`
- **第二次握手**就是Server收到客户端的请求,通过STN=1知道这是要建立连接，于是发送一个响应包并将`SYN`位`ACK`位置为1，并且自己`ack（确认序列号）`必须是客户端发过来的`seq＋1`，表示收到了客户端的SYN，此时Server进入`SYN_REVD`状态。
- **第三次握手**，客户端收到服务器的响应包，检查ACK确认序列号是否是发送的`seq+1`，需要对响应包进行确认，确认包将ACK位置为1，确认序列号等于对方发来的序列号`seq+1`，自己的序列号`seq+1`，发送给服务器。此时进入`ESTABLISHED`状态。服务器此时检查收到的ACK和确认序列号,也进入`ESTABLISHED`状态，完成三次握手，连接建立。

```json
Client(send_wait)>>>>>>>>>>[SYN=1,Seq=x]>>>>>>>>>>>>>>>>>>>>Server

Client<<<<<<<<<<<<<<<<<[SYN=1,ACK=1,Seq=y,ack=x+1]<<<<<<<<<<Server(syn_revd)

Client(established)>>>>>[ACK=1,Seq=x+1,ack=y+1]>>>>>>>>>>>>>>Server
```

> 总结：syn是请求同步位，ack确认同步位，seq自己的序列号，，ackNum 确认对方发来的seq。

**可以两次握手吗？**

不可以。

三次连接第一是防止因为出现网络超时出现脏连接。

TTL网络报文的生存时间超过TCP请求的超时时间，假设两次握手可以建立建立。假设客户端发送一个超时的连接暂时没有到达服务器，此时在发送一个正常的连接和服务器通信后断开。断开连接后假设超时的连接到了服务器，此时服务器以为是新来建立连接的，确认建立连接。然后此时客户端不是SYN_SEND的状态，不会理会服务器的确认数据，这导致知识服务器单方面的建立了连接，浪费了资源。如果是三次握手的话，服务器会因为长时间没有收到客户端的这一次回复而导致连接建立失败。

其次就是在第一次握手服务器仅仅是知道了客户端可以发送数据的能力，如果自己发送出去的信息没有得到回复，服务器此时无法证明自己的发送能力和客户端的接受能力是正常的。还有就是双方的初始序列号也没有得到交换。

**可以四次握手吗？**

三次都可以，四次当然可以了。就是降低了传输的效率。本来现在TCP被人诟病的就是性能不好。

四次握手意思就是第二次握手的时候服务器只发送ACK和ACKNum,再发送一次自己的SYN和自己初始化SEQ序列号，显然这是可以合并为一次的。

**如果已经建立了连接，客户端故障了怎么办？**

服务端每次收到客户端的请求都会复位一个计时器，时间通常是2小时，若两个小时还没有收到客户端的消息，就会发送一个探测报文段，每隔75秒发一次，若一连10个报文段都没有响应，就认定客户端出现了故障，连接关闭。

现在也有将这种机制放在了应用层，定期发送心跳包来检测，一旦不对就回收连接。

**初始序列号是什么**

双方各自随机选择一个32位的数字作为发送数据的初始序列号，以后每次发动数据都是以这个数字为起点对发送的数据进行编号，以便于对方可以知道确认什么样的数据是合法的。

## 四次挥手

四次回收用来断开双方的连接.

- **第一次挥手**：Client将FIN置为1，发送一个Seq给Server，进入FIN_WAIT_1状态。
- **第二次挥手**：Server收到FIN后，发送一个ACK，ackNum = Seq + 1,进入CLOSE_WAIT状态。此时客户端没有要发送的数据了，但是可以接受服务器的消息.此时客户端处于FIN_WAIT_2。
- **第三次挥手**：此时服务器需要处理完数据，再发送一个FIN包，ACK=1,Ack = ack = seq+1.进入last_ack状态。
- **第四次挥手**：客户端收到服务的FIN后，进入TIME_WAIT状态，接着将ACK置为1，发送一个确认包给服务器，服务器收到后，进入closed状态，不再发送数据。客户端等待2*MSL（报文段最长寿命）时间，也进入closed状态。

```json
C(fin_wait_1)>>>>>>>>>>[FIN=1,Seq=x]>>>>>>>>S

C<<<<<<<[ACK=1,ack=x+1,seq=y1]<<<<<<<<<<<<<<<<S(close_wait)

c<<<<<<<[FIN=1,ACK=1,ack=x+1,seq=y2]<<<<<<<<<<<S(close)

c(close)>>>>>>>[ACK=1,Seq=x+1,ack=y2+1]>>>>>>>>>>>>>>>>S
```

**为什么不能把服务器的第二次挥手和第三次挥手的ACK和FIN合并起来，变为三次挥手?**

因为服务器收到客户端断开连接的请求，可能还有数据没有处理完毕，此时先确认，表示接收到了断开连接的请求，等处理完了之后再发送一个FIN包。

**TIME_WAIT的意义是什么？**

为了保证被动关闭放能够顺利进入closed状态。假设客户端的最后一次ACK没有成功的发送给了服务器，此时服务器会以为是客户端没有收到自己之前的消息，会再发送一次FIN_ACK,客户端收到第二次的三次挥手，会继续确认一次。并重新计时。

如果没有这个等待时间，服务器没有收到这个消息，四次挥手的ACK消息如果丢了，这回导致对方一直无法正常关闭。

## TCP如何实现流量控制？

在TCP的首部有一个窗口字段，占两个字节。窗口字段用来控制对方发送的数据量，单位是字节。TCP连接的一端根据设置的缓存空间大小确定自己的接受窗口大小，然后通知对方以确定对方发送窗口的上限。

发送窗口的上限不仅仅受接收方窗口大小限制，还有拥塞窗口。拥塞窗口就是网络传输的能力，取二者之间的最小值。

**什么是零窗口，接收窗口0会怎么样**

如果接收方没有能力接收数据，就会将接收窗口设置为0，这时发送方必须暂停发送数据，但是会启动一个持续计时器(persistence timer)，到期后发送一个大小为1字节的探测数据包，以查看接收窗口状态。如果接收方能够接收数据，就会在返回的报文中更新接收窗口大小，恢复数据传送。

****

## TCP的拥塞控制如何实现的？

![image-20210417222415034](ComputerNetWork.assets/image-20210417222415034.png)

拥塞控制主要由四个算法组成。

**慢启动，拥塞避免，快重传，快恢复**

- 慢启动：开始发送数据的时候，先把拥塞窗口cwnd设置为1，每次收到新的确认报文，窗口值翻倍。说是慢启动，其实cwnd的值增长的特别快，发送方的速度增快过快，从而发生网络拥塞的可能性更高，设置一个慢开始门限ssthresh,如果出现了超时，ssthresh = cwnd/2.然后重新开始慢开始。

- 拥塞避免：如果cwnd>=ssthresh，每次窗口值只加1.
- 快重传：快重传算法规定，发送方只要一连收到三个重复确认就应当立即重传对方尚未收到的报文段，而不必继续等待设置的重传计时器时间到期。
- 快恢复：当发送方连续收到三个重复确认时，就把慢开始门限减半，然后执行拥塞避免算法。不执行慢开始算法的原因：因为如果网络出现拥塞的话就不会收到好几个重复的确认，所以发送方认为现在网络可能没有出现拥塞。

## TCP与UDP的区别

1.TCP是面向连接的，UDP是连接的。TCP是可靠的，UDP是不可靠的。TCP只支持点对点通信，UDP支持一对一 ，一对多，多对多。TCP是面向字节流的，UDP是面向报文的。

TCP有一系列的措施来保证连接的可靠性。比如数据包校验，丢弃重复的数据，应答机制，超时重发，流量控制（滑动窗口）。

**无连接和有连接的区别**

无连接属于数据报服务，每个数据包含有目标地址，网络尽最大的努力交付数据，不保证不丢失，不保证先后顺序，不保证在时限内交付。

## HTTP和HTTPS的区别

首先端口号不同。HTTP是80端口，HTTP是443端口。

HTTP是明文传输，HTTPS运行于SSL层之上，添加了加密和认证机制，更加安全。加密解密带来了CPU的开销。HTTPS需要证书。