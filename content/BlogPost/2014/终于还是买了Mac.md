其实我一直很喜欢Windows。作为一个OS，Windows可以说是成功得不能再成功了。普遍的观点是，Windows虽然适合使用，但是不适合做开发，所以程序员需要一个 *nix 操作系统的电脑。这么说自然是有道理的，在我看来，Windows不适合做开发，主要因为以下几点：

1. 编码  
搞Python开发的同学绝对被 Windows 蛋疼的 GBK 编码折磨过，更不要说 Python2 本身就存在诸多编码的坑，需要大量的经验或者指导才能够顺利跳过去。所以我后来有个强迫症，就是所有代码文件，不管里面有没有中文字符，统统弄成 utf-8，并且还写了个[工具](https://github.com/laike9m/all_in_utf8)来把某个文件夹中所有内容转成utf-8。

2. CRLF  
这一点其实还好，大多时候不会造成什么影响。就算是跨平台工作，问题也不大。顺便说一下我 msysgit 的设置一直是 line ending 强制转成 LF，checkout 的时候不修改。因为 Windows 下 \n 就可以正常完成换行工作。

3. shell 不够强大  
这是致命的弱点。对于正常的程序员来讲，很多工作用命令行完成效率的确更高。好在有个工具能够弥补：[GetGnuWin32](http://getgnuwin32.sourceforge.net/)。说实话，我认为有了 GetGnuWin32 之后，在 Windows 下做开发的障碍就扫清了不少（cmd.exe 太丑陋这点估计是无法改变）。但是还存在一个最大的障碍，就是第4点：

4. 工具链缺乏  
之前我没有意识到工具链对平台的影响，认为 Windows 之所以不适合做开发要归结于OS自身的问题。某天我偶然在 Youtube 上看到了 Jessica McKellar 的一个演讲 [The Future of Python](https://www.youtube.com/watch?v=d1a4Jbjc-vU)。她主要是在讲 Python 的话题，但是其中提到了重要的一点，就是 Python 用户在 Windows 上使用 Python 的时间不够多（毕竟大家主要还是在 *nix 上工作）导致 Python 代码的 Windows 版本（不论是 stdlib 还是 third party）都更新缓慢，并且存在大量未发现的 bug。实际上，导致我最终决定要买 Mac 的原因之一就是，之前某项任务需要的几个第三方库无论如何在 Windows 上都装不好。  
工具链缺乏的问题，和平台本身的质量其实不应该混为一谈，在我看来，两者对于 Windows 作为开发平台的负面影响各占一半，而如果我们把 GetGnuWin32 以及 msysgit/clink/chocolatey/Vim... 这些东西都装到 Windows 上之后，其实前者的影响就已经很小了。如果要探究 Windows 工具链缺乏的原因，无非也就是两个，首先是历史原因，毕竟 Unix hacker 之所以叫 Unix hacker 是因为他们都在 Unix 下工作，再有就是恶性循环，工具链越缺乏，就越没有人在 Windows 下做开发，在 Windows 下做开发的人越少，工具链就越缺乏。为什么大家觉得在 Windows 上开发 C++ 还是不错的，甚至强于 *nix 平台，就是因为微软花大力气把 Windows C++ 开发的工具链给完成了，铺平了这条道路。

好吧，说了这么多，最后还是要抱怨一句：**为什么Evernote不打算出Linux版？？**。我曾经花了很多时间研究怎么在 Linux 上顺畅使用 Evernote 或者类似产品，虽然取得了一些成效，但是结果仍然不满意（比如Wine下Evernote字体显示极为丑陋还有各种乱码，Wizbook 剪藏功能一坨屎，NixNote 客户端丑得没法看），最后只能勉强靠网页版度日。网页版用来浏览笔记是不错，编辑功能却烂的要命。可以这么说，如果印象笔记有官方Linux版，我绝对不会花钱去买 Mac。

PS: 有见过中学就开始编程且现在仍从事CS相关工作且编程水平一般的人吗？我感觉，从中学开始编程的人如果仍然在编程，不是大牛的可能性几乎是零。这让我感觉到，编程的学习其实是一个逐渐加速的过程（似乎很多人都这么觉得）。有时间看看能不能就这个问题写篇文章。 