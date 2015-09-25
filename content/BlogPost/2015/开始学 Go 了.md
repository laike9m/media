虽然早就想开始学，然而一直没有时间。正好之后也不面试了，全力准备 team match，正是开始学 Go 的好时机。

目前在跟着无闻的[《Go编程基础》][1]学。感觉看视频还是最快的。

学了变量的声明&定义的写法之后，我感觉 Go 搞出三种不同的写法完全就是为了迎合不同语言的开发者。

原来是写 C/C++/Java 的，你还是可以像以前那样明确地声明类型 `var a string = "ss"`，也可以先声明再定义。

原来是写 Js 的，那还犹豫啥，`var a = "ss"` 走起。

原来是写 Python/Ruby 等动态语言的，就加个冒号吧：`a := "ss"`

于是大家都觉得，嗯？Go 好像和我之前写的语言挺像的，不错。

不过也许这么设计是有其它原因的吧。

**9.16**

又多学了一些语法，发现 Go 的设计特别有意思，经常让我想笑（并非贬义）。这些设计我把它总结为：**消除最佳实践，消除争辩**。什么意思？在对任何一种编程语言的讨论中，程序员往往会为完成同一件事的程序到底怎么写是最好的而争论不休。最著名且吵得最厉害的例子自然要数“大括号到底放在函数定义的同一行还是下一行”。这种争议也不能说完全没有价值，但是很浪费时间。Go 的设计者说，让我们从最开始设计语言的时候就消除这些争议：

* `a++`/`a--` 只能单独作为一行，再也不怕谭浩强出的那些奇奇怪怪的问题了

* `switch` 语句默认不 fallthrough，防止一些自以为很聪明的人到处宣扬自己多么会利用 fallthrough 特性

* 用首字母大小写决定可见性，避免每个程序员自己发明一套可见性命名规则

* 大括号规定必须放在同一行。看到这个我真的笑了，那些鼓吹新开一行好的程序员要尴尬死了，不过就让他们继续写 C++/Java 吧

以上几个算是为了消除争议而做的设计。还有为了消除最佳实践的设计： 

* `break LABEL`/`continue LABEL` 简直不要太爽  
	看看这个问题 [How to break out of multiple loops in Python?][2]，最高票答案提出了一种最佳实践：把 nested loop 部分单独写成函数，然后把 `return` 当 `break` 用。我觉得这个方法相当丑陋，怎么可能 nested loop 部分的逻辑正好适合抽象成函数？更逗的是，下面有人提到：
> [PEP 3136][3] proposes labeled break/continue.  Guido [rejected it][4] because "code so complicated to require this feature is very rare".  The PEP does mention some workarounds, though (such as the exception technique), while Guido feels refactoring to use return will be simpler in most cases.  

    虽然我支持 Guido 强推 Python3，但不得不承认老爹有时候是比较顽固，因为我实在不觉得这是 “very rare” 的情况。

* 数组作为值而不是引用传递  
	把列表传入函数并对其进行修改是常见需求。 如果是 Java/Python 这种默认传引用的语言，显然既可以在原处修改列表，也可以返回新列表。于是有人提出最佳实践，说应该返回新的列表而不是在原地修改。Go 这种新语言居然不是传引用而是像 C 一样默认传值，我认为是刻意设计的（当然也可能是我想多了），即是说，我们不需要讨论最佳实践，你们老实地返回新列表就行了。

**9.25**

现在又学了 `slice`, `map` 和函数。在理解 `panic` 和 `recover` 机制上花了许多时间，找了几个教程包括官方文档都没讲清楚。最后终于找到一个：[Understanding Defer, Panic and Recover][5]。现在总算是明白了。里面有一个比喻非常好：调用 `panic` 就好像推倒一块多米诺骨牌，函数会逐层停止执行，直到 `main` 函数，然后程序崩溃。而 `recover` 就好像是把其中一块骨牌抽走，把程序崩溃的过程在 `recover` 处停止。所以 `recover` 之后的代码还能够正常执行。

这篇文章里还讲了使用 `panic`, `defer`, `recover` 的一些最佳实践，比如为了让 `main` 函数可以接收到 `panic error`，`defer` 要传 `&err`，然后在 defer 函数里给 `*err` 赋 `recover` 接收到的 `Error`。看来 Go 还是有最佳实践的，只不过需要应用的东西不同了。

[1]: https://github.com/Unknwon/go-fundamental-programming
[2]: http://stackoverflow.com/questions/189645/how-to-break-out-of-multiple-loops-in-python
[3]: http://www.python.org/dev/peps/pep-3136/
[4]: http://mail.python.org/pipermail/python-3000/2007-July/008663.html
[5]: http://www.goinggo.net/2013/06/understanding-defer-panic-and-recover.html