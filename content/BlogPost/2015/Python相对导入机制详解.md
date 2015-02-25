本文是对[http://stackoverflow.com/questions/14132789/python-relative-imports-for-the-billionth-time#answer-14132912](http://stackoverflow.com/questions/14132789/python-relative-imports-for-the-billionth-time#answer-14132912)这个SO答案的翻译。本人的翻译一向只追求含义准确而不追求字字对应，有些不好翻的术语或者固定说法就直接保留。

这个答案能解释大多关于relative import，即相对导入的疑惑，讲解十分详尽清晰，算是SO上被低估的一个答案。

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

Relative imports use a module's name attribute to determine that module's position in the package hierarchy. If the module's name does not contain any package information (e.g. it is set to 'main') then relative imports are resolved as if the module were a top level module, regardless of where the module is actually located on the file system.

The above response looks promising, but it's all hieroglyphs to me. So my question, how do I make Python not return to me "Attempted relative import in non-package"? has an answer that involves -m, supposedly.

Can somebody please tell me why Python gives that error message, what it Means by non-package!, why and how do you define a 'package', and **the precise answer put in terms easy enough for a kindergartener to understand.** Thanks in advance!

Edit: The imports were done from the console.

---

### [BrenBarn](http://stackoverflow.com/users/1427416/brenbarn)的精彩答案（这个哥们可以算是import专家了，答了好多这方面的题）

简单地说，直接运行.Py文件和import这个文件有很大区别。Python解释器判断一个py文件属于哪个package时并不完全由该文件所在的文件夹决定。它还取决于这个文件是如何load进来的（直接运行or import）。

有两种方式加载一个py文件：作为top-level脚本或者作为module。前者指的是直接运行脚本，比如`python myfile.py`。如果执行`python -m myfile`，或者在其它py文件中用`import`语句来加载，那么它就会被当作一个module。有且只能有一个top-level脚本，就是最开始执行的那个（比如`python myfile.py`中的`myfile.py`，译者注）。

当一个py文件被加载之后，它会被赋予一个名字，保存在`__name__`属性中。如果是top-level脚本，那么名字就是`__main__`。如果是作为module，名字就是把它所在的packages/subpackages和文件名用`.`连接起来。

例如，`moduleX`被import进来，它的名字就是`package.subpackage1.moduleX`。如果import了`moduleA`，它的名字是`package.moduleA`。如果直接运行`moduleX`或`moduleA`，那么名字就都是`__main__`了。

另一个令人担忧的问题是，一个module的名字取决于它是直接从它所在的文件夹import还是通过某个package import的。不过只有当你在某个路径中运行Python并试图从当前文件夹import一个py文件时，才需要关注它们的不同。例如，在路径`pacakge/subpackage1`中运行python解释器，然后脚本中有`import moduleX`这个语句，此时`moduleX`的名字正是`moduleX`，而不是`package.subpackage1.moduleX`。这是因为Python解释器在启动时把当前路径加入了它的搜索路径(search path)；如果发现要import的module就在当前路径，那么Python解释器就不知道当前路径是属于哪个package的，所以pacakge的信息就不会成为module的名字的一部分。

一个特例是直接运行python REPL，这个REPL的session的名字是`__main__`。

关于你遇到的错误信息，关键点来了：**如果一个module的名字中没有点（即package.subpackage1中的那个点，译者注），那么它就被认为不属于任何一个package**。文件在磁盘上的位置在哪里都不影响，唯一起决定作用的就是*module的名字*，而这又取决于它是如何被加载的。

先在我们看看你在问题中引用的这段话
> Relative imports use a module's name attribute to determine that module's position in the package hierarchy. If the module's name does not contain any package information (e.g. it is set to 'main') then relative imports are resolved as if the module were a top level module, regardless of where the module is actually located on the file system.  

relative import使用module的名字来决定它是否属于一个package，属于哪个package。当你使用这种relative import `from .. import foo`，其中的点的数量代表了package结构中的某个层次。例如，如果当前module的名字是`package.subpackage1.moduleX`，那么`..moduleA`代表`package.moduleA`。为了让形如`from .. import`的这种导入能够正常工作，module的名字里的点数量应当至少和import语句中一样多。

前面说了，如果module的名字是`__main__`，那么Python就不认为它属于某个package。**由于名字里不包含点，所以在这个py文件中`from .. import`语句无法正常工作**。试图执行这条语句就会报"relative-import in non-package"错误。

你犯的错误可能是从命令行运行`moduleX`或类似的操作。当你执行这个操作，moduleX的名字被设置成`__main__`，所以relative imports失败了，因为不包含package信息。正如前面说的，如果在同一个路径里import一个文件，这时module的名字就是文件名，不包含package信息，所以相对导入也会失败。

记住，因为REPL session的名字总是`__main__`，所以试图在REPL里执行relative import是不行的。relative import应当只在module文件中被使用。

（无法相对导入的问题）有两个解决方法。如果你真的想直接运行`moduleX`，同时又希望它被当作所在package的一部分，可以这么做：`python -m package.subpackage.moduleX`。`-m`参数告诉Python解释器，把这个文件当作一个module载入，而不是top-level脚本。

如果你并不想直接运行`moduleX`，而是想在另一个文件比如`myfile.py`中使用`moduleX`中定义的函数，那么解决方法是把`myfile.py`文件挪到另一个地方，只要不在`moduleX`所属的package的文件夹里就行。然后在`myfile.py`中执行`from package.moduleA import spam`，就能正常工作了。

注意，不论哪种解决方案，都需要package的路径（上文中的`package`）在python的搜索路径也就是`sys.path`里。如果不在，那么就无法使用这个package中的任何文件。

（更严谨的说明：从Python2.6开始，在做package-resolution时，module的“名字”并不完全等于`__name__`属性，还和`__package__`属性有关。这也是为什么上文中我一直尽量避免用`__name__`来指代module的名字。从python2.6开始，一个module的“名字”实际上是`__package__ + '.' + __name__`, 或者直接就是`__name__`，如果`__package__`是 `None`的话）