用途
-----
**1. 多个文件作为输入, 遍历文件每一行**  
**2. 在原处修改文件**  

#### 用途1

如果我们有如下的文件夹结构：  

```bash
test/  
|_____ 1.txt   content: 
|                 1_line1  
|                 1_line2    
|_____ 2.txt   content: 
|                 2_line1
|                 2_line2  
|_____ test_fileinput.py  
```

```python
import fileinput  
import sys  

for line in fileinput.input(sys.argv[1:]):
     print(fileinput.filename(), fileinput.filelineno(), line)
```

在 Linux 下执行

```bash
$ python test_fileinput.py *.txt
```
输出

```bash
1_line1

1_line2
2_line1

2_line2
```

在Windows下直接用*.txt, 系统不认  
![Windows Fail](/media/content/BlogPost/images/windows_fileinput_fail.jpg)

所以修改成

```python
import glob
all_files = [f for files in sys.argv[1:] for f in glob(files)]
for line in fileinput.input(all_files):
     print(fileinput.filename(), fileinput.filelineno(), line)
```

这样兼容Linux/Windows, 两种形式 `*.txt` 和 `1.txt 2.txt` 都可以作为输入。  
总之最后提供一个 filelist 给 input 就行。也可以这样:  

```bash
$ ls | ./filein.py
```
甚至并不一定是多个文件, 也可以是多个文件的内容, 例如  

```bash
$ cat *.py | python fileinput_grep.py fileinput
```
也能正常工作, 会遍历"所有.py文件的每一行"。以上的用法对单个文件/文件内容也同样适用。
<br>
#### 用途2: 在原处(inplace)修改文件

```bash
for line in fileinput.input('file.txt', inplace=True):
     line = ...  # edit line
     print line,  # stdout is redirected to the file
```
<br>
#### Note:

1. fileinput 能够提供很多元数据信息, 例如上面看到的`fileinput.filename()`, `fileinput.filelineno()`,还有`fileinput.isfirstline()`, `fileinput.isstdin()` 等, 更多内容参考[官方文档][1]

2. fileinput 支持 context manager
例如在原处修改文件最好写成

    ```python
    with fileinput.input('file.txt', inplace=True) as f:
        for line in f:
        ...
    ```
否则最后还得调用 fileinput.close()

3. 一个常见的任务是，如果某一行满足某个条件，那么修改/删除这一行。这时候**不要忘记原样输出别的行！**

4. 如果单纯地`print`会多出一些空白行，这是因为`print`的时候末尾自带一个换行符。解决方法有两种  
(1) Python2.X  
去掉末尾的换行，`strip`默认删除空白符（包括'\n', '\r',  '\t',  ' ')

    ```python
    line.rstrip()
    print line
    ```
(2) Python3.X  
`print()`函数有个参数是`end`，代表以什么结尾，默认是`\n`，我们用空字符做结尾

    ```python
    print(line, end='')
    ```

[1]: http://docs.python.org/3/library/fileinput.html#fileinput.filename