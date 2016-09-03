原本是知乎上的一个答案，现在挪到博客里来。

## 1. Type Hints - Guido van Rossum
视频：https://www.youtube.com/watch?v=2wDvzy6Hgxg  
主要就是讲 PEP 484 - Type Hints，通过 typing 这个模块，从 3.5 版本开始，Python 也可以做静态检查了！（更新：实际上 3.5 里还不能做静态检查，只是加入了类型标记，参见[What's New In Python 3.5](https://docs.python.org/3.6/whatsnew/3.5.html)）

## 2. Raymond Hettinger - Beyond PEP 8 -- Best practices for beautiful intelligible code  
视频：https://www.youtube.com/watch?v=wf-BqAjZb8M  
强烈推荐！Raymond Hettinger 的演讲适合所有层次的程序员看。这个演讲说的是，我们都知道用 PEP8 来规范 Python 代码，但是这样是否就够了呢？我们可能忽视了一件更重要的事：Pythonic！这个演讲举了几个例子，怎么 make code more pythonic，比如使用 context manager，使用 `@property`，使用 Adapter 包装已有代码让 `import` 更简洁。不过其中最神的我觉得还是通过定义 `__len__` 和 `__getitem__` 来构造一个 sequence，然后这个 sequence 自动就是 iterable 的，之前我一直以为必须通过定义 `__iter__` 和 `__next__` 方法构造 iterator 才能达到让类支持 for in 的效果。  
这个演讲没有 slide，不过代码不多。

## 3. David Beazley - Modules and Packages: Live and Let Die!
视频：https://www.youtube.com/watch?v=0oTh1CXRaQ0  
Slide: http://www.dabeaz.com/modulepackage/ModulePackage.pdf  
代码：https://github.com/dabeaz/modulepackage  
David Beazley 以前的视频我也看过几个，他的演讲最大特点就是 mind-blowing，会在短时间内让你意识到你根本就不会 Python！ 如果有人能在不查资料的情况下理解他的演讲的 50% 内容，那这个人绝对是 Python 高手。这次的视频也是一如既往得烧脑，讲的是和 import 相关的各种知识，因为之前写 [ezcf](https://github.com/laike9m/ezcf) 的缘故这次终于能看懂稍微多一点。。。实在不想介绍了，各位可以自己去看。    
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

## 5. Raymond Hettinger - Super considered super!
视频：https://www.youtube.com/watch?v=EiOglTERPEo  
Reddit 讨论：http://dwz.cn/1ns168  
Raymond Hettinger 2 Hit！然而这不是最主要的，最重要的是 Raymond 去讲 `super` 了。没错，在写出著名的[《Python’s super() considered super!》][super]**四年**之后，Raymond 终于在 PyCon 上讲了这个主题。这个迟到了四年的演讲的内容除了文章里的那些，就是在逐条批驳 [《Python's Super Considered Harmful》][harmful] 里的观点，不过我之前并没有读过 Harmful 一文，等读了再来补充。

[super]: https://rhettinger.wordpress.com/2011/05/26/super-considered-super/
[harmful]: https://fuhm.net/super-harmful/

### 6. Ned Batchelder - Facts and Myths about Python names and values
视频：https://www.youtube.com/watch?v=_AEJHKGk9ns  
Slide：http://nedbatchelder.com/text/names1.html  
比较初学者向的 talk，解释了 Python 的 name 到底是什么。比较经典的几句话是：  
**Assignment never copies data**  
**Mutable and immutable are assigned the same**  
**Function arguments are assignments**  
前两句话的意思是，Python 的 assignment 做的事仅仅是把左边的 name refer 到右边的值，而右边给的值是什么，其实和这个 assignment 操作是没有关系的。  
还有一个最佳实践，就是最好不要在函数中原地 modify 作为参数的 list，最好返回一个 new list 并在外面接收。  
一个测试对 Python 了解程度的问题：`a = []`，`a += [1]` 是否等价于 `a = a + [1]`。答案是**否**。

### 7. Brett Slatkin - How to Be More Effective with Functions
视频：https://www.youtube.com/watch?v=WjJUPxKB164   
Slide：http://www.onebigfluke.com/2015/04/how-to-be-more-effective-with-functions.html  
虽然题目是讲 Function，不过核心内容却是在说 iterator 和 generator，以及卖书。。。 
Brett 提倡尽可能不 return list，而是 return generator。说到 `iter(iter(some_list))` 其实和 `iter(some_list)` 返回一样的结果，也就是同一个 iterator。  
然后就是讲了一个使用 iterator 时经常会碰到的问题：一个 iterator 被 exhaust 之后，再遍历就没东西了。他提出的解决方案是：  
1. 如果要多次遍历一个 iterator，假定叫 `it`，使用 `if iter(it) is iter(it): raise xxx` 来防止问题发生；  
2. 当然光有1还不行，他提出使用“generator container”，说白了就是自定义 `__iter__`，比如

```python
class LoadCities(object):

    def __init__(self, path):
        self.path = path

    def __iter__(self):
        with open(self.path) as handle:
            for line in handle:
                city, count = line.split('\t')
                yield city, int(count)
```

然后用 `for x in LoadCities('pop.tsv')` 来遍历。因为每次是返回一个新的 iterator，所以不会出现之前的问题。不过其实吧，用 [`itertools.tee`](https://docs.python.org/3/library/itertools.html#itertools.tee) 可能更简单点。
