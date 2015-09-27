本文是另两篇文章的预备。想必很多人都知道 iterator 这个概念，也知道 Python 里有一个 `__iter__` 方法。
这里想讲一些关于 iterator 的历史。

[Iterator][1] 这个概念在很多编程语言里都有，Python 引入 iterator 是在 2.2 版本，也就是 2001 年的时候。
Iterator 是用来干嘛的，维基百科已经写得很清楚：

> In computer programming, an iterator is an object that enables a programmer to traverse a container, particularly lists.

那么问题来了，在引入 iterator 之前，Python 是怎么遍历，比如说，list 或者 dict 的呢？

先来看 list。实际上是依赖于 `__getitem__` 方法。估计大部分人对这个方法并不熟悉，其实很简单：

```python
>>> [1,2,3].__getitem__(1)
2
```
给一个 index，返回相应的元素，相当于调用 `[1,2,3][1]`。  
于是，只要定义了这一个方法，我们就可以遍历自定义的类了：

```python
class MyClass:
    def __getitem__(self, index):
        if index > 3:
            raise IndexError("that's enough!")

        return index

for name in MyClass():
    print(name)
```
输出
```shell
0
1
2
3
```
看见 `IndexError`，`for in` 就知道遍历到头了，所以并不会报错。实际上 2.2 之前的 `for in` 都是调用
`__getitem__`。

再看 dict。在 2.2 之前，我们**并不能** `for x in some_dict` 因为 dict 并没有定义 `__getitem__`。
想遍历 dict 的话只能这样：
```python
for key in some_dict.keys():
    print(some_dict[key])
```
显然这样性能不会好，因为要先生成一个包含所有 key 的 list。

Python2.2 引入了 iterator，相关文档参见 [What’s New in Python 2.2 - PEP 234: Iterators][2]。总结起来有以下改进：

1. `for x in C` 会默认调用 `iter(C)`，如果 `C` 没有定义 `__iter__` 方法，会使用 `__getitem__`；
2. dict（以及其它一些内置类型）现在实现了 `__iter__` 方法，所以可以 `for x in some_dict` 了。这样遍历
dict 比原来快很多，因为不需要生成一个 key list；
3. `for in` 默认调用 `iter()` 的好处在于，用户可以对遍历过程应该返回什么，如何返回有更强的控制。另一方面就是拓宽了 `for in` 的使用范围，因为并不是所有我们希望遍历的 object 都拥有有意义的 index。比如之前 `MyClass`
的例子，`index` 就没有含义，所以用 `__getitem__` 实现并不合适。

<br/>
这篇文章主要是讲历史。下一篇文章详细讲 `iterator` 要怎么用，最后一篇对三种实现遍历的方式进行比较。


参考资料：  
https://en.wikipedia.org/wiki/Iterator  
http://effbot.org/zone/python-for-statement.htm  
https://docs.python.org/3/whatsnew/2.2.html#pep-234-iterators  


[1]: https://en.wikipedia.org/wiki/Iterator
[2]: https://docs.python.org/3/whatsnew/2.2.html#pep-234-iterators 
