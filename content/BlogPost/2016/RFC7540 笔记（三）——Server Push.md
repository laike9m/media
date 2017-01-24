接着上回继续讲 stream 状态机。

## 5. Streams and Multiplexing

### 5.1 Stream States

上次讲了我们所熟知的 request-response 工作流在 HTTP/2 中的样子。这里再重复一次：**在 HTTP/2 中一个 request-response 就对应一个 stream，换句话说一个 stream 的任务就是完成一组 request-response 的传输，然后就没用了**。

这次讲 server push。为了完整，我把不属于本章的一些内容（6.6, 8.2）也挪到这里。首先用一个例子介绍一下 server push。比如你在 HTTP/1.1 下访问 example.com，服务器会首先把 html 返回给你，然后浏览器看到 html 的内容，再发请求去拿 html 包含的资源比如图片、css 等。Server push 做的事是，服务器在返回 html 之后，因为知道 html 内容，所以可以直接把里面包含的资源发给客户端，这样客户端就不需要再发请求了。Server push 的好处是显而易见的：1. 减少了请求；2. 因为服务器不再需要等待后续请求到来而是主动 push，减少了一个 RTT 的等待时间。

![](/media/content/BlogPost/images/server-push-flow.png)

上面淡紫色线表示的就是一个 server push 的流程。实际上，server push 依然符合之前对 request-response 的描述，即一个 request-response 对应一个 stream。下面会看到为什么。

<s>先从熟悉的部分开始。上篇文章中详细讲了 stream 的关闭过程，对于 server push stream，这个过程是一样的。reserved 状态和 open 状态差不多，DATA frame 是在 reserved 状态下传输的。在 reserved 状态下，要么发送 RST\_STREAM 直接 close stream，要么 client 和 server 各自发送 END\_STREAM 先 half-closed 再 closed。这是我们已经了解的过程，如果不了解可以去阅读前一篇文章。</s>

所以现在问题有两个：

1. 既然功能差不多，为什么要弄一个 reserved，而不使用 open？reserved 到底是什么意思？
2. 从 idle 到 reserved 的过程是怎样的？

现在没法回答这两个问题，因为要理解 server push，必须得先跳出 server push stream，去看前一个 stream。

### 前一个 stream

前一个 stream 是这样的 stream，**该 stream 的 response reference 了要 push 的资源。**什么意思？下面的继续使用访问 example.com 的那个例子来解释。

example.com 的服务器之所以可以提前 push 一些资源，是因为知道将要返回给客户端的 html 中 reference 了这些资源。在此场景下，“前一个 stream”，指的就是这个 html 作为 response 的 stream，也就是你访问 example.com 时发出的第一个 request 和 response 对应的 stream。

我们已经知道，server push 的好处是减少了 client 的请求。但是，client 怎么知道什么时候该发请求去拿 css，什么时候不该发呢？显然只能由 server 告知。问题是告知的时机。等 client 收到 html 之后再告知来得及吗？来不及，因为可能在 client 接收到 html 的一瞬间看到有 css 马上就发出请求，之后再告知也没意义了。所以，**server 必须在 html 到达 client 之前告知将要 push 的资源**。这样便引出了 [8.2.1 节](http://httpwg.org/specs/rfc7540.html#PushRequests)中的一个原则，我直接引用如下：

> The server SHOULD send **PUSH\_PROMISE**([Section 6.6](http://httpwg.org/specs/rfc7540.html#PUSH_PROMISE)) frames prior to sending any frames that reference the promised responses. This avoids a race where clients issue requests prior to receiving any **PUSH\_PROMISE** frames.

至此终于可以正式引入 PUSH\_PROMISE 这个概念了。PUSH\_PROMISE 正是之前所说的“server 对 client 不要为某个资源发请求的告知”。所谓“promised responses”指的则是 server 打算 push 的那些资源。所以这个原则其实就是我们之前说的“server 必须在 html 到达 client 之前告知将要 push 的资源”，只不过把 html generalize 成了“frames that reference the promised responses”。

### PUSH\_PROMISE

OK，现在来讲 PUSH\_PROMISE，以下简称 PP。PP 存在的意义上面已经说过了，下面是另外几件你需要知道的事：

1. **PP 必须在前一个 stream 上发送**

   即是说，PP frame 的 stream ID 得是前一个 stream 的 stream ID。

2. **PP 实质上是一个 request**

   前面说到在 server push 中，一个 request-response 仍然对应一个 stream。然而 client 并不发送 request，难道不是应该只有 response 么？原因在于，PP 充当了 request 的角色。没错，这是一个由 server 发出的 request。

3. PP 的结构

   ```tex
   +---------------+
   |Pad Length? (8)|
   +-+-------------+-----------------------------------------------+
   |R|                  Promised Stream ID (31)                    |
   +-+-----------------------------+-------------------------------+
   |                   Header Block Fragment (*)                 ...
   +---------------------------------------------------------------+
   |                           Padding (*)                       ...
   +---------------------------------------------------------------+
   ```

   PP 的 payload 结构如图所示。Header Block Fragment 和 [6.2 HEADERS](https://http2.github.io/http2-spec/#HEADERS) 的 Header Block Fragment 是一个东西，也就是 HTTP/2 的 header fields。上篇文章说过，HTTP/2 中的 HEADERS 可以视作 HTTP/1.1 中的 request。所以现在就很清楚了，PUSH\_PROMISE 中携带了 request 的所有信息，故它实质上就是一个 server 发出的 request。

   注意，图中只画了 payload，并未包含 stream ID，不要和 Promised Stream ID 弄混。那么 Promised Stream ID 是什么？它是由 server 选的一个偶数，代表一个还没有被使用过的 stream。之所以必须是没使用过的 stream，因为这个 ID 代表的 stream 正是接下来 push 要用的 stream，即传输资源 DATA 的 stream，也即 server push 的 request-response 对应的 stream。

### Reserved State

现在来回答一开始提出的两个问题

1. 既然功能差不多，为什么要弄一个 reserved，而不使用 open？reserved 到底是什么意思？

   PP 中的 Promised Stream ID 代表接下来发送 response 要用的 stream。不同于 open，之前并没有任何数据在这个 stream 上被发送，因为 PP 是在前一个 stream 上被发送的。这就好比，我打电话给餐馆订桌，虽然我现在不在餐馆，但是保证下午六点前会过去，这就是一个 promise。PUSH\_PROMISE 也是一样的，虽然在 Promised Stream ID 代表的 stream 上还没有发送过任何数据，但是我 promise 马上会有一个 push response 在上面发送。

   Promise 的含义清楚了，但是 reserved 还没解释。继续用餐馆举例子。餐馆接到电话后，会给我留一个桌，于是这个桌就被预定了，处于 reserved 状态。Stream 也是同样的道理，reserved 状态表示这个 stream 被预定了，其它 request-response 不能用。

   这个万能的例子还可以展示另一种情况：client 拒绝 push。client 在接到 PP 之后，如果发送 RST\_STREAM，代表它并不想接收 push。server 在接收到 RST\_STREAM 时有可能已经发送了一些 DATA，若没发完，也就没必要接着发了。这就好比餐馆答应帮我留桌，但是几分钟后打电话过来说刚才搞错了，其实并没有空桌了。于是我只能取消在这个餐馆吃饭的计划。如果我已经在路上，则有必要更改目的地。

   提醒一下，上文所举的例子只是为了给读者一个概念，请勿强行对应。

2. 从 idle 到 reserved 的过程是怎样的？

   已经解释过了，server 发送 PUSH\_PROMISE 把一个 stream 由 idle 变为 reserved。再次强调，server 在选择 Promised Stream ID 的时候必须选那些还没被使用过，即处于 idle 状态的 stream 才行。

那么 server push 主要的内容就讲完了。虽然是一篇 2000 字的文章，但我并没有自信让读者看完就彻底理解 server push。想真正理解，还是要读 RFC，并且多想。