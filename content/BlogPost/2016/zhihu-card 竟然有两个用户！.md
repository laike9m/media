今天田俊给 zhihu-card 提了个 [issue](https://github.com/laike9m/zhihu-card/issues/1)，于是在刷 Redis 的时候顺便看了下之前的数据，发现居然有五条！

```bash
127.0.0.1:6379> KEYS *
1) "6487fb46d752cc83f9a5eebf2134ed1c"
2) "1f644a1b7da169d2b56e1a4c6da61fea"
3) "5a6237cebe73cf47a10f1ba0c604cb8c"
4) "ac5860a0000ac307eda1c957a4a23643"
5) "9dc46aa6570ad50f998d93e5e63049de"
```

不过其实有三条分别是我、程浩和田俊，算不上真正的用户。那么其它两条呢？

一位是：https://www.zhihu.com/people/zhihudianxiaoer。我点进它的主页，果然有个人网站，网站上的确在使用 zhihu-card。 

![](/media/content/BlogPost/images/店小二.png)

然而尺寸似乎大了点，会不会是用户不知道可以调宽度呢？于是我就试着在网页里改了一下：

![](/media/content/BlogPost/images/zhihu-card-bug.png)

这下尴尬了，发现是因为代码有 bug 所以用户才不调整的……明天修一下好了。

另一位是 https://www.zhihu.com/people/xlzd，不过没看到他的网站。

