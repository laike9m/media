Stream 是 RFC7540 的重点，理解了 stream 的基本概念、flow-control、priority 再加上 server push，也就理解了 HTTP/2。

## 5. Streams and Multiplexing

stream 可以由 client 或 server 的任意一方创建。frame 在 stream 中的流动是双向的，且接收方会按照接收到 frame 的顺序处理它们。

### 5.1 Stream States

这个状态机示意图很重要，表示 stream 的状态变化。

```bash
                                +--------+
                        send PP |        | recv PP
                       ,--------|  idle  |--------.
                      /         |        |         \
                     v          +--------+          v
              +----------+          |           +----------+
              |          |          | send H /  |          |
       ,------| reserved |          | recv H    | reserved |------.
       |      | (local)  |          |           | (remote) |      |
       |      +----------+          v           +----------+      |
       |          |             +--------+             |          |
       |          |     recv ES |        | send ES     |          |
       |   send H |     ,-------|  open  |-------.     | recv H   |
       |          |    /        |        |        \    |          |
       |          v   v         +--------+         v   v          |
       |      +----------+          |           +----------+      |
       |      |   half   |          |           |   half   |      |
       |      |  closed  |          | send R /  |  closed  |      |
       |      | (remote) |          | recv R    | (local)  |      |
       |      +----------+          |           +----------+      |
       |           |                |                 |           |
       |           | send ES /      |       recv ES / |           |
       |           | send R /       v        send R / |           |
       |           | recv R     +--------+   recv R   |           |
       | send R /  `----------->|        |<-----------'  send R / |
       | recv R                 | closed |               recv R   |
       `----------------------->|        |<----------------------'
                                +--------+

          send:   endpoint sends this frame
          recv:   endpoint receives this frame

          H:  HEADERS frame (with implied CONTINUATIONs)
          PP: PUSH_PROMISE frame (with implied CONTINUATIONs)
          ES: END_STREAM flag
          R:  RST_STREAM frame
```

图比较难看懂，主要在于它包含了两个不同的过程。因此我把它们分开表示（server push 过程下次讲）：

![](/media/content/BlogPost/images/normal-flow.png)

绿线所覆盖的流程代表我们所熟知的 request-response 工作流。**在 HTTP/2 中一个 request-response 就对应一个 stream，换句话说一个 stream 的任务就是完成一组 request-response 的传输，然后就没用了。**先不考虑 half-closed，于是流程可以简化为 `idle ——> open ——> closed` ，下面先解释这个流程。

#### idle

idle 很好理解，就是 stream 的初始状态。你也可以认为凡是还没被使用过的 stream 都是 idle 状态。

#### open

stream 发出或接收到 HEADERS（HEADERS + CONTINUATION 就相当于 HTTP/1.1 中的 request）之后进入 open 状态，这时 server 开始发送 DATA 也就是 response（没有标在图上）。有人可能会不理解为什么图中要把 send HEADER 和 receive HEADER 分开写，这是因为 stream 的状态实际上是在 client 和 server 端同时被维护的，即保存了两份。它们表示同一个 stream 的状态，理论上应该实时同步，但实际上两边维护的 stream state 变化过程是不同的，且并非每时每刻都相同。说得更明白点，`idle ——> open` 的过程是：**client 发出 HEADER + CONTINUATION 后把 client 端维护的 stream state 变为 open，server 接收到 HEADER + CONTINUATION 后把 server 端维护的 stream state 变为 open**。明白了 stream state 有两份这点，才能理解之后的内容。

#### closed

DATA（response） 发完之后就要关闭 stream，最终 stream 进入 closed 状态，永远不再被使用。

可以看到，`idle ——> open ——> closed` 的流程其实是我们已经熟悉的 request-response 工作流。不过 stream 并不是说关就关，下面解释 stream 由 `open ——> closed` 的过程。

#### Stream 是怎么关闭的？

第一种，通过 RST\_STREAM。发送、接收到 RST\_STREAM，那么这个 stream 直接进入 closed 状态。

第二种，通过 END\_STREAM。Reset 快啊，但一般只有 stream 或某个 endpoint 出了些状况的时候才会使用，属于非常规手段。正常情况下 stream 是通过 END\_STREAM 结束的。某个 endpoint 发送 END\_STREAM 的意思是说，对一个 request-response 而言，我这边该发的数据都发完了，所以我觉得可以 close 这个 stream 了。一边说了不算，如果 peer 还没发完，当然是不能 close 的。如果接到 peer 发的 END\_STREAM，那么说明 peer 也同意 close stream，于是 stream 变为 closed 状态。这是 `open ——> half closed ——> closed` 的流程。

所以你们现在知道，**`half-closed` 表示一边想 close，但另一边还没有同意的状态**。只不过呢，图里把这个流程分成了两条路，描述某个 endpoint 存储的 stream 状态变化（记得之前说到 stream 状态会存两份么）。

第一条路 `open — send ES —> half closed(local) — recv ES —> closed` ，endpoint 先发送 END\_STREAM，再接收到 END\_STREAM，half closed(local) 代表该 endpoint 想 close；

第二条路 `open — send ES —> half closed(remote) — recv ES —> closed` ，endpoint 先接收到 END\_STREAM，再发送 END\_STREAM，half closed(remote) 代表 peer 想 close。

local/remote 的含义需要自行体会，希望现在有把它讲清楚。另外需要说明的是，RST\_STREAM 是一种特殊的 frame 类型，而 END\_STREAM 则是 HEADERS 或者 DATA 带的 flag。前面说的发送 END\_STREAM，指的是发送 set 了 END\_STREAM flag 的 HEADERS 或 DATA。

本来想把两个 workflow 一起讲的，不过篇幅所限，server push 的内容就放到下次好了。