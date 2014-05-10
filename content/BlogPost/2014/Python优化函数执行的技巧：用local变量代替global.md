这个技巧太简单了，但是直到阅读这篇文章的时候才意识到可以这么做  

[http://effbot.org/zone/default-values.htm][1]

里面有一段话是这么说的

>and, for highly optimized code, local rebinding of global names:   
>import math  

>def this_one_must_be_fast(x, sin=math.sin, cos=math.cos):  
>    ...  

意思是说，如果需要严格地优化（运行时间），可以把gloabl的函数/变量传入某个函数，这样对于这个函数来讲，它们就变成本地的了。Python在搜索变量时按照**Local->Enclosing->Global->Builtin**的路线进行，因为local最先搜索的，所以用局部变量显然是最快的。如果要弄清这个问题，可以搜索"LEGB"，在这里只要知道使用局部变量最快就可以了。

测试了一下，还是差别不小的
```python
import timeit

setup="""
import math

def calc(a, b):
	math.sin(a) + math.cos(b)

def calc2(a, b, sin=math.sin, cos=math.cos):
	sin(a) + sin(b)
"""

def test():
	print(timeit.timeit('calc(1, 2)', setup))
	print(timeit.timeit('calc2(1, 2)', setup))

if __name__ == '__main__':
	test()
```

`calc`耗时0.793981s，`calc2`耗时0.522706s。这里默认测试10000次，可以想象，如果代码中的变量更多，计算更复杂，那么时间的差别会更大。

[1]:http://effbot.org/zone/default-values.htm