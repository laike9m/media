本文是对 [http://stackoverflow.com/questions/14132789/python-relative-imports-for-the-billionth-time#answer-14132912](http://stackoverflow.com/questions/14132789/python-relative-imports-for-the-billionth-time#answer-14132912) 这个 SO 答案的翻译。本人的翻译一向只追求含义准确而不追求字字对应，有些不好翻的术语或者固定说法就直接保留。

这个答案能解释大多关于 relative import，即相对导入的疑惑，讲解十分详尽清晰，算是 SO 上被低估的一个答案。

问题不翻译了，直接摘录下来：   
The forever-recurring question is this: With Windows 7, 32-bit Python 2.7.3, how do I solve this "Attempted relative import in non-package" message? I built an exact replica of the package on pep-0328:

```
package/

    __init__.py

    subpackage1/

        __init__.py

        moduleX.py

        moduleY.py

    subpackage2/

        __init__.py

        moduleZ.py

    moduleA.py
```

I did make functions named spam and eggs in their appropriate modules. Naturally, it didn't work. The answer is apparently in the 4th URL I listed, but it's all alumni to me. There was this response on one of the URLs I visited:

Relative imports use a module's name attribute to determine that module's position in the package hierarchy. If the module's name does not contain any package information (e.g. it is set to'main') then relative imports are resolved as if the module were a top level module, regardless of where the module is actually located on the file system.

The above response looks promising, but it's all hieroglyphs to me. So my question, how do I make Python not return to me"Attempted relative import in non-package"? has an answer that involves -m, supposedly.

Can somebody please tell me why Python gives that error message, what it Means by non-package!, why and how do you define a 'package', and **the precise answer put in terms easy enough for a kindergartener to understand.** Thanks in advance!

Edit: The imports were done from the console.

---

### [BrenBarn](http://stackoverflow.com/users/1427416/brenbarn) 的精彩答案（这个哥们可以算是 import 专家了，答了好多这方面的题）

简单地说，直接运行 .py 文件和 import 这个文件有很大区别。Python 解释器判断一个 py 文件属于哪个 package 时并不完全由该文件所在的文件夹决定。它还取决于这个文件是如何 load 进来的（直接运行 or import）。

有两种方式加载一个 py 文件：作为 top-level 脚本或者作为 module。前者指的是直接运行脚本，比如 `python myfile.py`。如果执行 `python -m myfile`，或者在其它 py 文件中用 `import` 语句来加载，那么它就会被当作一个 module。有且只能有一个 top-level 脚本，就是最开始执行的那个（比如 `python myfile.py` 中的 `myfile.py`，译者注）。

当一个 py 文件被加载之后，它会被赋予一个名字，保存在 `__name__` 属性中。如果是 top-level 脚本，那么名字就是 `__main__`。如果是作为 module，名字就是把它所在的 packages/subpackages 和文件名用 `.` 连接起来。

例如，`moduleX` 被 import 进来，它的名字就是 `package.subpackage1.moduleX`。如果 import 了 `moduleA`，它的名字是 `package.moduleA`。如果直接运行 `moduleX` 或 `moduleA`，那么名字就都是 `__main__` 了。

另一个令人担忧的问题是，一个 module 的名字取决于它是直接从它所在的文件夹 import 还是通过某个 package import 的。不过只有当你在某个路径中运行 Python 并试图从当前文件夹 import 一个 py 文件时，才需要关注它们的不同。例如，在路径 `pacakge/subpackage1` 中运行 python 解释器，然后脚本中有 `import moduleX` 这个语句，此时 `moduleX` 的名字正是 `moduleX`，而不是 `package.subpackage1.moduleX`。这是因为 Python 解释器在启动时把当前路径（这里答案写的不准确，其实加入的是 top-level 脚本的路径，因为两者在这种状况下相同，所以也并不算错。译者注）加入了它的搜索路径 (sys.path)；如果发现要 import 的 module 就在当前路径，那么 Python 解释器就不知道当前路径是属于哪个 package 的，所以 pacakge 的信息就不会成为 module 的名字的一部分。

一个特例是直接运行 python REPL，这个 REPL 的 session 的名字是 `__main__`。

关于你遇到的错误信息，关键点来了：** 如果一个 module 的名字中没有点（即 package.subpackage1 中的那个点，译者注），那么它就被认为不属于任何一个 package**。文件在磁盘上的位置在哪里都不影响，唯一起决定作用的就是 * module 的名字 *，而这又取决于它是如何被加载的。

先在我们看看你在问题中引用的这段话
> Relative imports use a module's name attribute to determine that module's position in the package hierarchy. If the module's name does not contain any package information (e.g. it is set to'main') then relative imports are resolved as if the module were a top level module, regardless of where the module is actually located on the file system.  

relative import 使用 module 的名字来决定它是否属于一个 package，属于哪个 package。当你使用这种 relative import `from .. import foo`，其中的点的数量代表了 package 结构中的某个层次。例如，如果当前 module 的名字是 `package.subpackage1.moduleX`，那么 `..moduleA` 代表 `package.moduleA`。为了让形如 `from .. import` 的这种导入能够正常工作，module 的名字里的点数量应当至少和 import 语句中一样多。

前面说了，如果 module 的名字是 `__main__`，那么 Python 就不认为它属于某个 package。** 由于名字里不包含点，所以在这个 py 文件中 `from .. import` 语句无法正常工作 **。试图执行这条语句就会报 "relative-import in non-package" 错误。

你犯的错误可能是从命令行运行 `moduleX` 或类似的操作。当你执行这个操作，moduleX 的名字被设置成 `__main__`，所以 relative imports 失败了，因为不包含 package 信息。正如前面说的，如果在同一个路径里 import 一个文件，这时 module 的名字就是文件名，不包含 package 信息，所以相对导入也会失败。

记住，因为 REPL session 的名字总是 `__main__`，所以试图在 REPL 里执行 relative import 是不行的。relative import 应当只在 module 文件中被使用。

（无法相对导入的问题）有两个解决方法。如果你真的想直接运行 `moduleX`，同时又希望它被当作所在 package 的一部分，可以这么做：`python -m package.subpackage.moduleX`。`-m` 参数告诉 Python 解释器，把这个文件当作一个 module 载入，而不是 top-level 脚本。

如果你并不想直接运行 `moduleX`，而是想在另一个文件比如 `myfile.py` 中使用 `moduleX` 中定义的函数，那么解决方法是把 `myfile.py` 文件挪到另一个地方，只要不在 `moduleX` 所属的 package 的文件夹里就行。然后在 `myfile.py` 中执行 `from package.moduleA import spam`，就能正常工作了。

注意，不论哪种解决方案，都需要 package 的路径（上文中的 `package`）在 python 的搜索路径也就是 `sys.path` 里。如果不在，那么就无法使用这个 package 中的任何文件。

（更严谨的说明：从 Python2.6 开始，在做 package-resolution 时，module 的 “名字” 并不完全等于 `__name__` 属性，还和 `__package__` 属性有关。这也是为什么上文中我一直尽量避免用 `__name__` 来指代 module 的名字。从 python2.6 开始，一个 module 的 “名字” 实际上是 `__package__ + '.' + __name__`, 或者直接就是 `__name__`，如果 `__package__` 是 `None` 的话）