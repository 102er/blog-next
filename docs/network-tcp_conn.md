## 简介

TCP 是一种面向连接的、可靠的、基于字节流的传输层通信协议，位于网络分层模型的传输层：

![osi](https://102er.github.io/images/osi-model.png)

- 面向连接：两个端必须建立tcp连接，才能通讯交换数据。
- 基于字节流：tcp连接双方的数据交换是以字节构成的有序但无结构的字节流。
- 可靠性：通过连接管理，流量控制，拥塞控制，超时重传机制，序号和确认序号等机制保证传输可靠。

<!-- more -->

## TCP报文段

![osi](https://102er.github.io/images/tcp-bw.png)

- 端口号：每个tcp报文包含源和目的端口号，2字节的端口号，端口号+ip可以组成一个socket
- 序号（seq）：数据序号，表示这个数据流在整个数据流中的序号，接收端可以根据序号组装数据
- 确认序号（ack）：确认序号，接收方成功接收数据，会回复发送端，并把接收的序号+1，告诉发送端自己接收了哪个序号的数据，下次数据要从ack序号开始发
- 首部长度：记录tcp头的长度，tcp
- 保留位：暂时没用
- 标志位：标记请求的目的，状态等
  - URG：值为 1 时，紧急指针生效，表示本次报文需要尽快传输，不要按照原本的队列次序传输
  - ACK：值为 1 时，确认序号生效，表示数据已经被接收
  - PSH：接收方应尽快将这个报文段交给应用层
  - RST：发送端遇到问题，想要释放当前连接，重建传输连接。
  - SYN：同步序号，用于发起一个连接
  - FIN：发送端要求关闭连接
- 窗口：TCP的流量控制由连接的每一端通过声明的窗口大小来提供。窗口大小为字节数，起始于确认序号字段指明的值，这个值是接收端正期望接收的字节。
- 检验和 (Checksum)：强制性必须携带的字段。检验和覆盖了整个 TCP 报文段，包括 TCP 首部和 TCP 数据，发送端根据特定算法对整个报文段计算出一个检验和，接收端会进行计算并验证。
- 紧急指针 (Urgent Pointer)：当 URG 控制位值为 1 时，此字段生效，紧急指针是一个正的偏移量，和序号字段中的值相加表示紧急数据最后一个字节的序号。 TCP 的紧急方式是发送端向另一端发送紧急数据的一种方式。
- 选项 (Options)：这一部分是可选字段，也就是非必须字段，最常见的可选字段是“最长报文大小 (MSS，Maximum Segment Size)”。
- 有效数据部分 (Data)：这部分也不是必须的，比如在建立和关闭 TCP 连接的阶段，双方交换的报文段就只包含 TCP 首部。

## 三次握手

![osi](https://102er.github.io/images/tcp-3ws.png)

1. 第一次握手：客户端向服务端发送连接请求报文。此时标志位：SYN=1，同时会初始化一个序列号x 填充到 seq序号位。发送完，客户端进入syn-send状态，等待服务端的确认。<u>TCP规定SYN=1的报文不能携带数据，但需要消耗一个序号。</u>
2. 第二次握手：服务端接收报文之后，如果同意连接，会发出确认报文，包含标志位：SYN=1,ACK=1，确认序号ack=x+1(<u>x是客户端发送的seq序号值，序号+1代表服务端期望下次接收到客户的数据序号为x+1</u>)，序号seq=y。发送完报文，服务端进入syn-received状态。<u>TCP规定SYN=1的报文不能携带数据，但需要消耗一个序号。</u>
3. 第三次握手：客户端接收到确认报文，还需要向服务端给出确认报文，包含标志位：ACK=1，确认序号ack=y+1，序号seq=x+1，此时，tcp连接建立，客户端进入established。<u>TCP规定，ACK报文段可以携带数据，但如果不携带数据，则不消化序号。</u>当服务端接收到客户端的确认报文后也进入established。此后，双方就可以开始通信了。

### 为什么三次握手？

- 防止已过期的连接请求报文突然又传到服务器，浪费服务器资源
  - 第三次握手可以对失效请求报文，进行确认，当他接收了失效请求报文会回复，如果客户端是关闭状态的，那没办法进行确认请求，所以服务端收不到客户端确认报文，会判断客户端并没有提交请求连接。（**失效请求**：客户端发送了第一次握手，但是网络因素滞留。客户端迟迟没有接收到服务端的确认报文，会再次发送握手请求。那么此时，第一次发送的握手请求就是失效的。）
- 三次握手才能确认让双方确认彼此的发送和接收能力
  - 第一次握手，服务端可以确认自己的<u>接收</u>能力和客户端的<u>发送</u>能力 
  - 第二次握手，客户端可以确认自己的<u>收发</u>能力和服务端的<u>收发</u>能力 
  - 第三次握手，前两次握手，服务端并不能知道自己的<u>发送</u>能力和客户端的<u>接收</u>能力是否正常。第三次握手，服务端收到了客户端第二次握手的回应，从服务端角度可以确认自己第二次握手发送的包发送出去且客户端接收了，所以确认了自己的<u>收发</u>能力和客户端的<u>收发</u>能力。
- 告知对方自己的初始序号值，并确认收到对方的初始序号值
  - 三次握手，才能保证服务端发送的seq初始序号得到确认。

### SYN FLOOD攻击

伪造大量的源ip地址，分别向服务端发送大量的syn包，服务端返回的SYN/ACK包，因为源地址是伪造的，所以不会有应答，服务端没有收到应答，会重试并且等待一个syn time，如果超时则丢弃这个连接。这种半开连接会消耗服务端的资源，导致服务端无法正常服务。

## **四次挥手**

![osi](https://102er.github.io/images/tcp-4hs.png)

1. 客户端发送连接释放报文段并且停止发送数据，此时标志位：FIN 标志位1，序号字段 seq = x (等于之前发送的所有数据的最后一个字节的序号加一)，然后客户端会进入 FIN-WAIT-1 状态，等待来自服务器的确认报文。<u>TCP规定，FIN报文段即使不携带数据，也要消耗一个序号。</u>
2. 服务器收到 FIN 报文后，发回确认报文，此时标志位：ACK = 1， 确认序号段 ack = x + 1，并带上自己的序号 seq = y，此时，服务器就进入 CLOSE-WAIT 状态。服务器还会通知上层的应用程序对方已经释放连接，此时 TCP 处于半关闭状态，即使客户端没有数据要发送，但是服务器还可以发送数据，客户端也还能够接收。
3. 客户端收到服务器的 ACK 报文段后随即进入 FIN-WAIT-2 状态，此时还能收到来自服务器的数据，直到收到 FIN 报文段。
4. 服务器发送完所有数据后，就向客户端发送连接释放报文，此时标志位：ACK=1,FIN=1,序号为seq=z，确认序号ack=x+1，随后服务器进入 LAST-ACK 状态，等待来自客户端的确认报文段。
5. 客户端收到来自服务器的 FIN 报文段后，向服务器发送 ACK 报文，随后进入 TIME-WAIT 状态，等待 2MSL(2 * Maximum Segment Lifetime，两倍的报文段最大存活时间) ，这是任何报文段在被丢弃前能在网络中存在的最长时间，常用值有30秒、1分钟和2分钟。如无特殊情况，客户端会进入 CLOSED 状态。
6. 服务器在接收到客户端的 ACK 报文后会随即进入 CLOSED 状态，由于没有等待时间，一般而言，服务器比客户端更早进入 CLOSED 状态。

### **为什么客户端最后还要等待2MSL？**

MSL（Maximum Segment Lifetime），TCP允许不同的实现可以设置不同的MSL值。去向ACK消息最大存活时间（MSL) + 来向FIN消息的最大存活时间(MSL)。这恰恰 就是**2MSL( Maximum Segment Life)。

1. 保证客户端发送的最后一个ACK报文能够到达服务器，因为这个ACK报文可能丢失，站在服务器的角度看来，我已经发送了FIN+ACK报文请求断开了，客户端还没有给我回应，应该是我发送的请求断开报文它没有收到，于是服务器又会重新发送一次，而客户端就能在这个2MSL时间段内收到这个重传的报文，接着给出回应报文，并且会重启2MSL计时器。
2. 等待2MSL时间，客户端就可以放心地释放TCP占用的资源、端口号。如果不等，释放的端口可能会重连刚断开的服务器端口，这样依然存活在网络里的老的TCP报文可能与新TCP连接报文冲突，造成数据冲突，为避免此种情况，需要耐心等待网络老的TCP连接的活跃报文全部死翘翘，2MSL时间可以满足这个需求（尽管非常保守）

### **如果已经建立了连接，但是客户端突然出现故障了怎么办？**

TCP还设有一个**保活计时器**，显然，客户端如果出现故障，服务器不能一直等下去，白白浪费资源。服务器每收到一次客户端的请求后都会重新复位这个计时器，时间通常是设置为2小时，若两小时还没有收到客户端的任何数据，服务器就会发送一个探测报文段，以后每隔75秒发送一次。若一连发送10个探测报文仍然没反应，服务器就认为客户端出了故障，接着就关闭连接。

### **为什么建立连接是三次握手，关闭连接确是四次挥手呢？**

建立连接的时候， 服务器在LISTEN状态下，收到建立连接请求的SYN报文后，把ACK和SYN放在一个报文里发送给客户端。 关闭连接时，服务器收到对方的FIN报文时，仅仅表示对方不再发送数据了但是还能接收数据，而自己也未必已经将全部数据都发送给对方了，所以己方可以立即关闭，也可以发送一些数据给对方后，再发送FIN报文给对方来表示同意现在关闭连接，因此，己方ACK和FIN一般都会分开发送，从而导致多了一次。
