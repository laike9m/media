当初为了无痛开发换到 Mac，之后大大小小也遇到过不少奇怪的问题，不过都没有这次的奇怪。考虑到之后也可能遇到，姑且记录一下。

问题的表现是无法安装任何带 C 扩展的 Python 库。在编译的过程中会提示 `<stdio.h> not found`。系统是 10.11 El Capitan。

于是就去网上搜，得知 Mac 下 `stdio.h` 应该位于 `/usr/include`。搞笑的是，`include` 文件夹消失了！去网上搜，发现很多人都遇到这个问题，估计是 Xcode 升级带来的 bug，然后基本所有人都说要通过 `xcode-select --install` 命令装 commandline tools。问题是我早就装了。

终于找到一个地方提示说可以手动把 Xcode 里面的文件夹链接过去。尝试执行

```
sudo ln -s /Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX10.11.sdk/usr/include /usr/include
```
提示
```
ln: /usr/include: Operation not permitted
```
搜索发现原因是 Mac 默认开启了 [Configuring System Integrity Protection][1]，简单来说就是一些系统相关的文件夹被设为只读。解决方法参考[这里][2]：

1. 重启电脑
2. 在启动过程中按住 `cmd + r`，进入 Recover Mode
3. 选择 utilities > terminal，输入命令
    ```
    csrutil disable   
    reboot
    ```
4. 再次执行 `ln`，成功。

再次尝试安装，这里以 `greenlet` 为例
```
$ pip install greenlet
...
Building wheels for collected packages: greenlet
  Running setup.py bdist_wheel for greenlet
  Complete output from command /usr/local/opt/python3/bin/python3.5 -c "import setuptools;__file__='/private/var/folders/sy/msgzx60s2_332s1wdb92fqw80000gn/T/pip-build-amdm1ven/greenlet/setup.py';exec(compile(open(__file__).read().replace('\r\n', '\n'), __file__, 'exec'))" bdist_wheel -d /var/folders/sy/msgzx60s2_332s1wdb92fqw80000gn/T/tmp4a1nacfbpip-wheel-:
  running bdist_wheel
  running build
  running build_ext
  building 'greenlet' extension
  creating build
  creating build/temp.macosx-10.10-x86_64-3.5
  clang -Wno-unused-result -Wsign-compare -Wunreachable-code -fno-common -dynamic -DNDEBUG -g -fwrapv -O3 -Wall -Wstrict-prototypes -I/usr/local/include -I/usr/local/opt/openssl/include -I/usr/local/opt/sqlite/include -I/usr/local/Cellar/python3/3.5.1/Frameworks/Python.framework/Versions/3.5/include/python3.5m -c greenlet.c -o build/temp.macosx-10.10-x86_64-3.5/greenlet.o
  Assertion failed: (!contains(New, this) && "this->replaceAllUsesWith(expr(this)) is NOT valid!"), function replaceAllUsesWith, file /Users/laike9m/Dev/C_CPP/Lib/cling/src/lib/IR/Value.cpp, line 343.
 ...
 
    ********************
    error: command 'clang' failed with exit status 254

    ----------------------------------------
Command "/usr/local/opt/python3/bin/python3.5 -c "import setuptools, tokenize;__file__='/private/var/folders/sy/msgzx60s2_332s1wdb92fqw80000gn/T/pip-build-amdm1ven/greenlet/setup.py';exec(compile(getattr(tokenize, 'open', open)(__file__).read().replace('\r\n', '\n'), __file__, 'exec'))" install --record /var/folders/sy/msgzx60s2_332s1wdb92fqw80000gn/T/pip-l6qwwxng-record/install-record.txt --single-version-externally-managed --compile" failed with error code 1 in /private/var/folders/sy/msgzx60s2_332s1wdb92fqw80000gn/T/pip-build-amdm1ven/greenlet 
```
我不相信 greenlet 的代码有问题，遂怀疑是 clang 的问题。于是想用 gcc 来编译。尝试 `export CC=gcc` 未果，发现仍然在使用 clang，报同样的错。还是在 SO 上找到了[解释][3]
> It has been this way for a long time already. The "GCC" that came with 10.8 was really GCC front-end with LLVM back-end.
  
解决方法是用 homebrew 安装 gcc，这样不会和系统的 gcc 冲突。
```
$ brew install gcc49
```
然后
```
export CC=/usr/local/Cellar/gcc49/4.9.3/bin/gcc-4.9
```
这下终于安装成功。

[1]: https://developer.apple.com/library/mac/documentation/Security/Conceptual/System_Integrity_Protection_Guide/ConfiguringSystemIntegrityProtection/ConfiguringSystemIntegrityProtection.html
[2]: http://stackoverflow.com/questions/32659348/operation-not-permitted-when-on-root-el-capitan-rootless-disabled#
[3]: http://stackoverflow.com/questions/19535422/os-x-10-9-gcc-links-to-clang
