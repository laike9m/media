原本是知乎上的一个答案，现在挪到博客里来。

## 1. Type Hints - Guido van Rossum
视频：https://www.youtube.com/watch?v=2wDvzy6Hgxg  
主要就是讲 PEP 484 - Type Hints，通过 typing 这个模块，从 3.5 版本开始，Python 也可以做静态检查了！

## 2. Raymond Hettinger - Beyond PEP 8 -- Best practices for beautiful intelligible code  
视频：https://www.youtube.com/watch?v=wf-BqAjZb8M  
强烈推荐！Raymond Hettinger 的演讲适合所有层次的程序员看。这个演讲说的是，我们都知道 PEP8 来规划 Python 代码，但是这样是否就够了呢？我们可能忽视了一件更重要的事：Pythonic！这个演讲举了几个例子，怎么 make code more pythonic，比如使用 context manager，使用 `@property`，使用 Adapter 包装已有代码让 `import` 更简洁。不过其中最神的我觉得还是通过定义 `__len__` 和 `__getitem__` 来构造一个 sequence，然后这个 sequence 自动就是 iterable 的，之前我一直以为必须通过定义 `__iter__` 和 `__next__` 方法构造 iterator 才能达到让类支持 for in 的效果。  
这个演讲没有 slide，不过代码不多。

## 3. David Beazley - Modules and Packages: Live and Let Die!
视频：https://www.youtube.com/watch?v=0oTh1CXRaQ0  
Slide: http://www.dabeaz.com/modulepackage/ModulePackage.pdf  
代码：dabeaz/modulepackage · GitHub  
David Beazley 以前的视频我也看过几个，他的演讲最大特点就是 mind-blowing，会在短时间内让你意识到你根本就不会 Python！ 如果有人能在不查资料的情况下理解他的演讲的50%内容，那这个人绝对是 Python 高手。这次的视频也是一如既往的烧脑，讲的是和 import 相关的各种知识，因为之前写 ezcf 的缘故这次终于能看懂稍微多一点。。。实在不想介绍了，各位可以自己去看。    
之前有人问 Python 里有哪些黑魔法。那个问题我没有答，我只想说 David Beazley 的每一个演讲都充斥着黑魔法。。。

## 4. Dan Crosta - Good Test, Bad Test
视频：https://www.youtube.com/watch?v=RfR_QRoNZxo  
Slide：https://speakerdeck.com/dcrosta/good-test-bad-test  
文章：http://late.am/post/2015/04/20/good-test-bad-test.html  
关于测试质量的演讲，重点讲述了三个测试的误区：  
(1) 过分追求 100% 覆盖率。他认为有些代码反而应该尽量避免去测试，比如依赖于网络上某个服务的代码。  
(2) 使用过多 assert。他的建议是一个 test function 里面最好只有一个 assert。  
(3) Mock 一定会让测试更好。在测试中用到了数据库，与其直接 mock 一个返回值，不如用简单的代码模拟数据库的工作流程。  
第二条我觉得像只用一个 assert 在很多情况下并不方便，但是把 dict 的 assert 拆分成多个键值 assert 还是可以借鉴的。关于数据库的 mock，其实有很多工具可以使用，比如见过一个专门用来为单元测试生成假数据的第三方库，还有 Django 的测试自带测试数据库。
