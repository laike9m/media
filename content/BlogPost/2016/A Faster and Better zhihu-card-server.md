zhihu-card 又有几个新用户了！

[Polo's Blog](http://blog.polossk.com/)

[xlzd杂谈](http://xlzd.me/)

[小狐濡尾](http://metaphor.space/)

[Tian Jun](http://tianjun.ml/)

其实有些并不是新用户，只不过我上次没发现。Redis 里还有几条别的用户记录，但并未找到个人网站。

这篇文章主要讲什么呢？国庆及最近几天大改了后端架构，并且[开源](https://github.com/laike9m/zhihu-card-server)出来，所以就来说一说。这篇文章在我看来价值并不大，因为并没有包含多少探索的过程，不过姑且做个记录。

![](/media/content/BlogPost/images/zhihu-card-server.png)

目前的架构长这个样。

jsDelivr 和 Chrome 的部分都在 [zhihu-card](https://github.com/laike9m/zhihu-card) 项目里，不是本文重点，总之后端会接收前端发的请求，并返回用户数据。

后端主要就三块，一个 Golang 的 HTTP Server，一个 Redis，一个 [Nameko](https://github.com/onefinestay/nameko) 的 local server。Go server 和 Redis 的交互就是很普通的 GET/SET，Redis 在 key 过期的时候会给 Go server 发通知让它去更新数据。不论是获取新用户数据还是更新已存在用户的数据，Go server 都是通过向跑在本地 Nameko server 发送一个请求，由 Nameko server 返回最新的用户数据。

OK，那么 Nameko 是什么？我最早是从大名鼎鼎的 Flask 作者 Armin Ronacher 的[博文](http://lucumr.pocoo.org/2015/4/8/microservices-with-nameko/)知道的，当然他并不是主要贡献者。Nameko 是一个用 Python 写的微服务框架，接口设计非常简洁，基本看一分钟文档就能上手。在做毕设的过程中我一度想尝试，不过后来发现需求不同转而使用了 [huey](https://github.com/coleifer/huey)（顺带一提 huey 也是个好东西，一个超轻量级的 task queue，该有的都有用起来也很稳定）。

所以为什么要用 Nameko？主要是因为我想用 [zhihu-oauth](https://github.com/7sDream/zhihu-oauth)，这样就可以省去因为知乎前端变动带来的代码修改工作。zhihu-oauth 是 Python 写的，那么 Nameko 就非常合适了。本来也不是说一定要用 zhihu-oauth，如果有 Go 的 zhihu oauth 客户端更好，这样连 Nameko 都不需要，但是没有啊。另外我对维护者 7sdream 同学的代码水平和修 bug 速度还是很有信心的，感谢他的出色工作。

在这次大改之前，后端会在请求到来时检查 Redis 中有无对应数据，如果没有，就把知乎的个人主页抓下来，提取数据存入 Redis 并返回给用户。这么做有两个问题，其一是知乎前端一变我就得改代码，其二是速度——每个 hit 到 cache miss 的访问者将不得不额外忍受一个访问知乎个人主页的 RTT。虽然不是大问题，但总觉得不爽。现在和知乎的交互全交给 zhihu-oauth 去做，速度比 GET 网页快很多。另一方面，由于 Redis 在 key 过期时会通知 Go server 并获取最新数据，因此对于已存在的用户，Redis 中总是有数据的，进一步减少了访问耗时。这个结论只在数据很少的时候成立，根据 Redis [文档](http://redis.io/commands/expire)，key expire 是每秒检查 10 次，每次 20 个：

>  Specifically this is what Redis does 10 times per second:
>
> 1. Test 20 random keys from the set of keys with an associated expire.
> 2. Delete all the keys found expired.
> 3. If more than 25% of keys were expired, start again from step 1.

现在的数据量完全可以保证在 1s 内检查完所有 key。

[zhihu-card-server](https://github.com/laike9m/zhihu-card-server) 目前还不完善。上面我说 Nameko 很好，其实这个库用起来特别坑，有些很重要的事文档里是不写的。目前又遇到程序莫名其妙退出的 bug，还没有解决，只好先加上一个 fallback，在无法从 Nameko 获取数据的时候直接访问网页。还有 Go server 的异常处理，logging 这些也还没写……总之慢慢搞吧。

最后推荐一下我正在用的 markdown 编辑器 [typora](https://www.typora.io/)，前段时间正好在想类似功能，发现已经有人做出来了。简单来说，你不再需要左边 markdown 右边渲染了，渲染和文本现在放在一起，就好像在写 Word 一样。