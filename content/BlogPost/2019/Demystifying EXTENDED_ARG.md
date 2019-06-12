Recently I was studying Python [bytecode](https://docs.python.org/3.8/glossary.html#term-bytecode) for my personal project. One particular thing that bothered me was [EXTENDED_ARG](https://docs.python.org/3/library/dis.html#opcode-EXTENDED_ARG). Guess what, the documentation is outdated(or, you could say, wrong), thus causing my confusion. But even without the error, it is not easy to understand at first glance. In this article, I'll explain it in detail.   

**Early warning**: If you've never heard of Python's bytecode, you probably want to learn about it before continue reading.

## Before Python 3.6

Let's first take a look at the \*original\* documentation.

> **`EXTENDED_ARG`**(*ext*)
>
> Prefixes any opcode which has an argument too big to fit into the default two bytes. *ext* holds two additional bytes which, taken together with the subsequent opcode’s argument, comprise a four-byte argument, *ext* being the two most-significant bytes.

Before Python 3.6, each instruction takes 1 or 3 bytes, depending on whether it takes an argument. Example:

```bash
RETURN_VALUE             # opcode takes 1 byte, total is 1
LOAD_CONST     0x0000    # argument takes 2 bytes, total is 3
```

But what if you want to have a larger argument which cannot fit into 2 bytes? That's when `EXTENDED_ARG` comes into play. Let's say, the argument is `0x123456`

```bash
LOAD_CONST     0x123456  # INVALID!! argument exceeds size limit
##################################################
EXTENDED_ARG   0x0012
LOAD_CONST     0x3456    # valid
```

Here's what Python does: it splits the argument into two parts, with 2 bytes each. The most significant 2 bytes `0x0012` becomes argument of `EXTENDED_ARG`, the remaining 2 bytes becomes argument of `LOAD_CONST`. When Python virtual machine sees `EXTENDED_ARG`, it knows to read the next instruction, and adds their arguments together. So the actual operation Python does is `LOAD_CONST` with argument `0x00123456`.

## After Python 3.6

Python 3.6 changed the size of instructions. So starting from Python 3.6, **every** instruction takes **2 bytes**, where opcode still takes 1 byte, argument also takes 1 byte now. If there's no argument, it's just zero.

So what's the equivalent bytecode representation of `LOAD_CONST 0x123456` in the latest version of Python? It's like this:

```bash
EXTENDED_ARG   0x12
EXTENDED_ARG   0x34
LOAD_CONST     0x56
```

The change is pretty straight forward, the only difference is the use of multiple `EXTENDED_ARG`. The size limit for argument is 4 bytes, so there will be 3 `EXTENDED_ARG` instructions at most, for each subsequent instruction(in this example, `LOAD_CONST`).

## Wait, the documentation is wrong?

Funny enough, the documentation was not updated to reflect this change(which is understandable given the amount of work needed to be done). So I made a [PR](https://github.com/python/cpython/pull/13985) to fix it.

## Play with it yourself

Finally, some code to prove the things we talked about.

I'll be using a brilliant library made by Victor Stinner, called **[bytecode](https://github.com/vstinner/bytecode)**. Thanks [@thautwarm](https://twitter.com/thautwarm) who introduced it to me.

Complete program can be found [here](https://gist.github.com/laike9m/6527fa614c208a3686a46f3dff87c61e).

In this program, I constructs a series of instructions by hand, which does two things: `LOAD_CONST 0x1234567` , and then `RETURN_VALUE` to pop up the loaded value from VM stack. With the help of bytecode lib, things become really simple.

```python
from bytecode import ConcreteInstr, ConcreteBytecode

CONST_ARG = 0x1234567  # The real argument we want to set.
cbc = bytecode.ConcreteBytecode()
cbc.consts = [None] * (CONST_ARG + 1)  # Make sure co_consts is big enough.
cbc.consts[CONST_ARG] = "foo"  # Sets the value we want to load.

if sys.version_info >= (3, 6):
    cbc.extend([
        ConcreteInstr("EXTENDED_ARG", 0x1),
        ConcreteInstr("EXTENDED_ARG", 0x23),
        ConcreteInstr("EXTENDED_ARG", 0x45),
        ConcreteInstr("LOAD_CONST", 0x67),
        ConcreteInstr("RETURN_VALUE"),
    ])
```

`cbc.extend` manually constructs the instructions, and the instructions should look familiar now. The tricky thing is about preparing value. Since we want to call `LOAD_CONST 0x1234567`, there needs to be a value located at `co_consts[0x1234567]`(If you don't know what `co_consts` is, check out [the doc](https://docs.python.org/3/library/inspect.html#module-inspect)). So what we do is set the value manually: `cbc.consts[0x1234567] = "foo"`.

Now comes the interesting part, let's `dis` the code object we just created:

```python
  1           0 EXTENDED_ARG             1
              2 EXTENDED_ARG           291
              4 EXTENDED_ARG         74565
              6 LOAD_CONST           19088743 ('foo')
              8 RETURN_VALUE
```

Is something wrong? Why is the argument different from what we set? Here's how it works

```python
1 = 0x01
291 = 0x0123
74565 = 0x012345
19088743 = 0x01234567
```

Now it's clear, Python accumulates the arguments when seeing `EXTENDED_ARG`, and it becomes [Instruction.arg](https://docs.python.org/3/library/dis.html#dis.Instruction). But under the hood, the byte value is still exactly what we set.

```python
for raw_byte in code.co_code:
    print("raw code is: ", raw_byte)
    
"""
raw code is:  144   # EXTENDED_ARG
raw code is:  1     # 0x01
raw code is:  144   # EXTENDED_ARG
raw code is:  35    # 0x23
raw code is:  144   # EXTENDED_ARG
raw code is:  69    # 0x45
raw code is:  100   # LOAD_CONST
raw code is:  103   # 0x67
raw code is:  83    # RETURN_VALUE
raw code is:  0     # no argument, so zero
"""
```

The program also supports running with versions before 3.6, the only difference is the two bytes argument:

```python
cbc.extend([
    ConcreteInstr("EXTENDED_ARG", 0x123),
    ConcreteInstr("LOAD_CONST", 0x4567),
    ConcreteInstr("RETURN_VALUE"),
])
```



