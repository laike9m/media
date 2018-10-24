这两年，我的博客一直放在 Webfaction 上面。在[《用Django搭建个人博客》][1]这篇文章里也提过，当时我说：

> 使用VPS的话，可选择的范围很大。但是第一次建网站，没什么经验，怕配置不好。更重要的是用VPS不够短平快，而我希望博客能尽快投入使用。

现在看来这个决定依然是正确的。两年中，网站一直很稳定。上周他们还帮我把 HTTPS 给装好了。

即使这样，一年之前我就有换到 VPS 的想法。主要原因有两个：

1. 从零开始打理 VPS 对我已经不再是一个问题了
2. Webfaction 实在有点贵

Webfaction 按年买价格最低，但也要 102 刀一年，平均下来就是 8.5 刀一个月。硬件是 512M 内存，600G 流量，100G 存储。按硬件来说是不需要这么多钱的，不过 Webfaction 毕竟是 SaaS，它的收费包含了各种附加服务，比如自动创建 Web App，工作人员解答问题，安装 HTTPS(证书还是得自己买) 等。换到 VPS 可以省下这些费用。

去年 GitHub 搞活动，推出 [Student Developer Pack][2]，只要你是学生，就可以去申请。给的东西除了 GitHub 2 年私有项目使用权，还有一大堆别的，其中最值的，就是 DigitalOcean 的 100 刀代金券。于是 DigitalOcean 就成了首选。

不过因为当时已经又买了一年的 Webfaction，暂时也就没有换。直到最近，才又开始想这件事。

说起来很好笑，起因竟是我没法登陆 Webfaction 的服务器了。我这一年一直用 `SSH -D` 来翻墙，并没有出什么事，然而最近被 GFW 盯上，目测是把对应 IP 的 22 端口给封了。网上说换个端口就 OK 了，我也以为可以这么做，然后发现，靠，Webfaction 没法换端口。Webfaction 的一台机器是 N 多用户共用的，在 `/home` 下面能看到几百个用户的家目录，所以你不能随便动除了自己家目录之外的东西，更不要提换 SSH 端口这种事了。虽然开着 VPN 还是可以登陆的，但是毕竟不方便，而且我也逐渐意识到和别人共用主机，很多操作都做不了。这种情况下，迁移的事又提上日程。

今天买了主机，配置：512MB Ram，20GB SSD Disk，San Francisco 机房，Ubuntu 14.04 x64 系统。选机房还是比较纠结的，看了知乎上的这个[问题][3]，决定在 San Francisco 和 New York 之间选。用 DO 自带的[测速页面][4]测了一下，San Francisco 1 和 New York 2 是最好的。又因为 New York 2 不支持 IPv6，所以最终确定用 San Francisco 1 机房。
 
之后就是逐步迁移网站，等有时间再弄。

最后放个 DigitalOcean 的[推广链接][5]，说是用这个链接买主机，买的人有 10 刀优惠。如果花了 25 刀以上，我也能拿到 25 刀。


[1]: https://laike9m.com/blog/yong-djangoda-jian-ge-ren-bo-ke,22/
[2]: https://education.github.com/pack/offers
[3]: http://www.zhihu.com/question/25529727
[4]: http://speedtest-sgp1.digitalocean.com/
[5]: https://www.digitalocean.com/?refcode=a582fa751343
