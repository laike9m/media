本文是我在 [PyCascades 2021](https://2021.pycascades.com/) 的[演讲](https://www.youtube.com/watch?v=eXlTVrNZ67Q)的 proposal。虽说是 proposal，却是以接近博文的风格写作的（毕竟我只会这种风格。。），所以就直接放出来水一篇了。对应的 [slide 在这里](https://laike9m.github.io/Slides/Let's%20Rethink%20Debugging%20-%20PyCascades%202021.html)。下一篇文章我会聊聊参加 PyCascades 的经历。

This is my talk proposal for [PyCascades 2021](https://2021.pycascades.com/). Even though it's a proposal, it reads very much like an article, so I just post it here.

## Abstract

As programmers, we do debugging almost every day. What are the major options for debugging, what advantages and disadvantages do they have? We'll start the talk by giving the audience an overview of the history of debugging and existing tools so they know how to pick from them.

Then, we'll help the audience gain a deeper understanding of what debugging is really about, and talk about two pain points with existing solutions. We'll introduce a novel approach to solve these pain points, with basic introduction to bytecode tracing so the audience can learn this useful technique.

Finally, we'll look into the future and talk about why it's important to be more innovative. We hope that by listening to this talk, the audience can be more open-minded thinking about debugging, and programming as a whole.  

No specific knowledge required, but basic experience with debugging would be helpful.

## Description

Here is a detailed description of each part.

### Part 1: What debugging is really about?

Broadly speaking, a Python program can have four types of errors:

- Syntax Error
- Exits abnormally (e.g. unhandled exceptions, killed by OS)
- **The program can run, but gives wrong results**
- Gives correct results, but consumes more resources than expected (e.g. memory leak)

Among which, the third type of error is the most common, and also where programmers spent most of their time debugging. In this talk we focus on this type of error, aka "A Program can run, but gives wrong results".

I'll let the audience recall how they usually do debugging. It's not hard to spot that, no matter what approach we take, we're trying to answer one question:

**What is the root cause of the wrong value?**

This sounds straightforward, but it is vital that we realize it before going into the later sections.

### Part 2: Retrospect the history of debugging

In the early days of programming, debugging meant dumping data of the system or output devices - literally printing, or displaying some flashy lights if there's an error. A very patient programmer then would go step-by-step through the code, reading it to see where the problem may be.

Then, in the 70s and 80s, the idea of "debugging software" came along, and people started to build command-line debuggers like gbx and GDB. Since then, despite new  features like breakpoint, reverse debugging and graphical interface were added, the way people use debuggers stays pretty much the same: step through the program and look around. 

Today, print, logging, and debugger remain to be the major ways for debugging, each with its advantages and drawbacks:

- `print`: 
  - **Advantages**: available out-of-the-box, clean information, does not affect program execution.
  - **Drawbacks**: requires familiarity with code, needs tweaking repeatedly, lack of context, hard to manage output.
- Logging:
  - **Advantages**: configurable, easy to manage output (e.g. Sentry), richer context (lineno, filename, etc).
  - **Drawbacks**: configuration is not easy, requires familiarity with code, hard to search what you need, context still not enough.
- Debugger:
  - **Advantages**: powerful, does not require familiarity with code, richest context to help identify problems.
  - **Drawbacks**: not always available, decent learning curve, can't persist output, needs human interaction.

Yet, with all these options, debugging is still hard sometimes. We'll see why in the next section.

### Part 3: Let's rethink debugging

There are two pain points with existing debugging solutions:

- **There is no tool that is as easy-to-use as a print, yet provides rich information like a debugger**.

  |   Tool   | Effort required | Information provided |
  | :------: | :-------------: | :------------------: |
  |  print   |       low       |        simple        |
  | logging  |     medium      |        medium        |
  | debugger |      high       |         rich         |
  |    ?     |       low       |         rich         |

- **Existing tools only give clues, without telling why.**

  This is a bigger (yet hidden) problem. 

  In the first part we talked about the goal for debugging, which is finding out the root cause of the wrong value. Let's use debugger as an example to recall how we usually debug. Let's say you're debugging a program, where `c` has an unexpected value:

  ```python
  c = a + b  # c should be "foo", but instead is "bar"
  ```

  Here are the normal steps:

  1. Set a break point at this line.
  2. Run the program, inspect the value of `a` and `b`.
  3. Figure out whether the error lies in `a` or `b`.
  4. Set another break point, repeat 🔁

  Or, if you want to do it in one run:

  1. Set a break point at the entry point of the program.
  2. Step through and program and remember everything happened along the way.
  3. Stop at `c = a + b`, use your brain to infer what happened.

  Either way, we still need to spend time reading the code and following the execution. We also need to keep monitoring all relevant variables in every step, compare them with the expected values, and memorize the results, because debuggers don't persist them. This is a huge overhead to our brain, and as a result made debugging less efficient and sometimes frustrating.

  The problem is obvious: debuggers only give clues, without telling why. We've been taking the manual work for granted for so long, that we don't even think it's a problem. In fact it is a problem, and it can be solved.

### Part 4: A novel approach to tackle the pain points

To reiterate, An ideal debugging tool should

- **Easy-to-use and provide rich information.**
- **Tell you why a variable has a wrong value with no or minimal human intervention.**

For a moment, let's forget about the tools we use every day, just think about one question: who has the information we need for debugging?

The answer is: **the Python interpreter**.

So the question becomes, how do we pull relevant information out of the interpreter?

I will briefly introduce the `sys.settrace` API, and the `opcode` event introduced in Python 3.7, with the example of `c = a + b` to demonstrate using bytecode to trace the sources of a variable. In this case, the sources of `c` are `a` and `b`. With this power, reasoning the root cause of a wrong value becomes possible.

I will then introduce [Cyberbrain](https://github.com/laike9m/Cyberbrain), a debugger that takes advantage of the power of bytecode to solve the pain points with **variable backtracing**. What it means is that, besides viewing each variable's value at every step (which Cyberbrain also supports), users can "see" the sources of each variable change in a visualized way. In the previous example, Cyberbrain will tell you that it's `a` and `b` that caused `c` to change. It also traces the sources of `a` and `b` all the way up to the point where tracing begins.

I'll do a quick demo of using Cyberbrain to debug a program to show how it solves the two pain points. By the end of the demo, the audience will realize that traditional debugging tools do require a lot of manual effort which can be automated. 

Bytecode tracing also has its problems, like it can make program slower and generate a huge amount of data for long running programs. But the important thing is that we realize the pain points, and don't stop looking for new possibilities, which brings the next topic.

### Part 5: Where do we go from here?

Now is an interesting time.

On the one hand, existing tools are becoming calcified. [Debug Adapter Protocol](https://microsoft.github.io/debug-adapter-protocol/) is gaining popularity, which  defines the capabilities a debugger should provide. Tools that conform to DAP will never be able to provide capabilities beyond what the protocol specifies.

On the other hand, new tools are coming out in Python's debugging space, just to list a few:

- [PySnooper](https://github.com/cool-RR/PySnooper), [IceCream](https://github.com/gruns/icecream), [Hunter](https://github.com/ionelmc/python-hunter), [pytrace](https://github.com/alonho/pytrace): lets you trace function calls and variables with no effort, automating the process of adding `print()`.
- [birdseye](https://github.com/alexmojaki/birdseye), [Thonny](https://thonny.org/): graphical  debuggers that can visualize the values of expressions.
- [Python Tutor](http://www.pythontutor.com/): web-based interactive program visualization, which also visualizes data structures.
- Cyberbrain.

These new tools share the same goal of reducing programmers' work in debugging, but beyond that, they are both trying to pitch the idea to people that the current "standard" way of debugging is not good enough, that more things can be achieved with less manual effort. The ideas behind are even more important than the tools themselves.

Why is this important? Dijkstra has some famous words:

> The tools we use have a profound (and devious!) influence on our thinking habits, and, therefore, on our thinking abilities. 

Imagine a world where all these efforts don't exist, will the word "debugger" gradually change from "something that can help you debug" to "something that conforms to the Debug Adapter Protocol"? That is not impossible. We need to prevent it from becoming the truth, and preserve a possible future where programmers are debugging in an effortless and more efficient way. So what can we do?

- Think of new ways to make debugging better;
- Create tools, or contribute to them;
- Spread this talk and the ideas;
- Create new programming languages that put debuggability as the core feature. 

And the easiest, yet hardest thing: **keep an open mind**.