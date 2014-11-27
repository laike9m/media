作为技术验证，最近实现了一下NAT穿透，并在此基础上完成了P2P聊天的客户端（虽然只能在命令行中打字）。理论上能不论电脑处于何种类型NAT设备后，均可以实现P2P聊天。代码和使用方法参见  
[https://github.com/laike9m/PyPunchP2P][1]  
这篇文章主要（简单）介绍一下必要的背景知识和原理，github上已有的内容就不再说明。

什么是NAT穿透？
--
[穿越防火墙技术][2]

什么是STUN, TURN？
---
[WebRTC and the Ocean of Acronyms][3]

如何实现NAT穿透？
---
[Peer-to-Peer Communication Across Network Address Translators][4]

喂这就算介绍完了吗(╯‵□′)╯︵┻━┻  
咳咳，总之原理部分就这样吧

<br />
PyPunchP2P工作流程
--
### PART ONE: 连接

假定你已经运行了`server.py`，并让其监听`1234`这个端口。客户端A首先会通过从[pystun][5]里面弄出来的那部分代码检测自己的NAT类型  
```python
nat_type, _, _ = self.get_nat_type()
```  
然后通知服务器端，发起连接请求，同时告知服务器自己的NAT类型。`client.py`的第三个参数是**pool**值，这个值是用来匹配客户端用的。如果说两个发起连接的客户端有一样的pool值，那么就认为它们是希望通信的客户端。指定的pool值也会发送给服务器。  
```
self.request_for_connection(nat_type_id=NATTYPE.index(nat_type))
```  
其中  
```
NATTYPE = (FullCone, RestrictNAT, RestrictPortNAT, SymmetricNAT)
```  
如果一切顺利，服务器接到了这个请求，那么它会保存客户端A的信息（addr, pool, nat_type），同时继续等待另一个客户端发起请求。  
好，现在客户端B也发了个请求过来，并且pool值和之前相同。服务器意识到A和B希望和对方通信，于是分别把A和B的信息发给对方。显然，这就是**STUN server**的本职工作。
```python  
a, b = poolqueue[pool].addr, addr  
nat_type_id_a, nat_type_id_b = poolqueue[pool].nat_type_id, nat_type_id  
sockfd.sendto(addr2bytes(a, nat_type_id_a), b)  
sockfd.sendto(addr2bytes(b, nat_type_id_b), a)
```    
至此第一部分的工作就完成了，clientA和clientB已经连接起来了，祈祷到这里一切顺利吧。

### PART TWO: 通信
注意到我们之前并没有利用NAT类型信息，下面就需要了。让我们分情况看看：  
#### 至少有一方是symmetric NAT  
这是最优先考虑的情况，因为symmetric NAT是最让人头大的情况。这种情况下只能通过服务器来转发消息。于是我们的服务器华丽变身为**TURN server**。当然，服务器不可能什么包都转发，所以这种通信方式下双方的消息带有一个`msg`的前缀，目的就是标识出这是希望服务器转发的消息而不是PART ONE中发起连接的那种消息。我们的服务器是不可能使用多个端口的，因为如果端口和之前建立连接时不同，那么服务器转发的消息就会直接被symmetric NAT丢弃了。既然和之前使用的是一个socket，那么标识显然是必要的。   
还有个问题是转发给谁。这一点无须担心，在建立连接时服务器已经把两个client配对了，如果是从一边来的消息，它会自动转发给另一边。
#### 不存在symmetric NAT，至少有一方是restrict NAT  
这里所指的restrict NAT包含了 RestrictNAT 和 RestrictPortNAT 两种情况。这时，是restrict NAT的那一方需要做一件事，那就是**持续发包！**不妨称这种包为**punching包**，设定为0.5s一次。另一方，不管是不是restrict NAT，接到punching包之后都会自动给出回复。原理上不难理解，因为受限的一方只有持续发包，才能让NAT设备知道对方是“已知”的，而一旦接收到回复，持续发包停止，可以开始聊天。
#### 双方都是Full Cone  
这种情况简直是天堂，直接向对方发送就行了，so easy.  
实际上，大部分情况都是这种。看来生活还是有希望的╮(╯▽╰)╭

大概就是这样了。~~再次声明，代码并未在真实情况下测试过，所以未必一定能正常工作。可以保证的是原理正确，以及在模拟状况下测试正常。~~目前已经测试过了，各种状况下都能正常工作，除非路由器或者防火墙被设定为阻挡来自某些IP的UDP报文，那确实无能为力了。另外，我不知道ICE是具体是怎么工作的，到处都说是对STUN+TURN的封装，难不成就和这个差不多？

**Update**  

有同学看完文章之后发邮件问我，正好这里也可以补充说一下：
> 你好！
> 看到你的blog，想问几个关于NAT 穿透的问题。
> 我们现在基于局域网+websocket实现了一个聊天软件，想知道如果走互联网的话，需要穿透NAT，是不是只能通过nat穿透的这个socket通信了？还是说如果nat穿透后，两个client就可以随意通信了？

我的回答：  
现在你想把局域网 websocket 聊天扩展到任意网络，我个人认为这个是不太现实的。websocket 底层是用 TCP 实现的，设计的目的并不是为了让客户端(比如浏览器)之间可以相互通信，而是客户端和服务器之间的通信。更广泛地说，要实现互联网上任意两台电脑之间的  TCP 连接，靠谱的做法只能是 UPNP，也就是各种 BT 软件的做法。虽然我的 blog 引用的那篇论文讲了 TCP 穿透，但是太复杂了。我说的 NAT 穿透其实都是针对 UDP 的。  
不论是用 TCP 还是 UDP 作穿透，之后必须继续沿用那个 socket，这一点是毫无疑问的。因为穿透的第一步是获知对方公网 ip:port，而每新开一个本地 socket，它对应的公网 port 一定会变化，所以如果你新开一个 socket 的话即使原来穿透成功了也没法通信，因为公网 port 变了。

[1]:https://github.com/laike9m/PyPunchP2P
[2]:http://www.cs.nccu.edu.tw/~lien/Writing/NGN/firewall.htm
[3]:https://hacks.mozilla.org/2013/07/webrtc-and-the-ocean-of-acronyms/
[4]:http://www.bford.info/pub/net/p2pnat/index.html
[5]:https://pypi.python.org/pypi/pystun







