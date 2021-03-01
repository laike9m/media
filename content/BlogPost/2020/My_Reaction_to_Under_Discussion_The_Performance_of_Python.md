Today I read an article *[Under Discussion: The Performance of Python](https://www.welcometothejungle.com/en/articles/btc-performance-python)*, and discussed it with some friends. I got some ideas from the discussion, so I decided to write an article.

My overall reaction to this article is: it points straight to some of the key factors that prevent Python to gain major performance **improvements**, thus really worth reading. 

In the article, the author interviewed Python core developer [**Victor Stinner**](https://twitter.com/VictorStinner) (VS) and  experienced Python developer **[Julien Danjou](https://twitter.com/juldanjou)** (JD). Their names should be familiar to those who's been in the Python community for a while. Unlike some other articles that talk about the technical reasons of why Python is slow, this article focused on the efforts made and difficulties met to improve Python's performance.

To start, let me quote the part that I find most interesting:

> **VS**: Now, there are different reasons behind Python’s slowness. The first is the official statement, which is that there is no such performance issue! Saying this is preventing us from working on the issue.

It blew my mind when I saw this. You know it, I know it, but as a core dev, you still need a bit of courage to say that in public. I appreciate what Victor said, and I'm 1000% [agree](https://metarabbit.wordpress.com/2018/02/05/pythons-weak-performance-matters/#comment-1776) with it.

> **JD:** I agree with Victor. I don’t see a lot of people working upstream on Python who are trying to improve its performance, which is a shame considering the number of people who use the language and want better performance from it in general.

(Julien has [clarified](https://twitter.com/juldanjou/status/1260501803516547076) that he was referring to big companies not investing enough in Python.)

I think it's too harsh to say that. I respect the work people (especially core devs) did, and they've made Python better than it ever has been. My friend [@uranusjr](https://twitter.com/uranusjr) reacted more strongly. He said:

> I don’t see why it’s a shame. **If you want it to improve, feel free to contribute.** Core developers don’t owe you anything.

This brings to the main topic I want to talk about today.

# The mindset of "If you want it to improve, feel free to contribute" does not help

Don't get me wrong. I agree with the word. What I'm saying is, it's not helpful. I'll explain it later.

As you might know, I hosted a Python podcast in my free time, and we once [invited](https://www.pythonhunter.org/episodes/9) [Xiang Zhang](https://twitter.com/angwerzx), the only core developer from China. One thing he said in the show still strikes me until today. He said, Python's evolvement pretty much relies on 个人英雄主义 (individualistic heroism), and that's why it's so hard to make big improvements.

The real problem behind the scenes is the **mindset**. We're so used to relying on individualistic heroism, relying on "feel free to contribute", and almost forgot that contributing to a project as complicated as Python, is hard. Python does have a comprehensive [Developer’s Guide](https://devguide.python.org), which is nice, but it's not enough. **We need to change the mindset from "*If you want it to improve, feel free to contribute*" to "*If you want it to improve, we'll make it easy for you to contribute*"**. This is important because, once we build this new mindset, it is obvious that a lot more can (or should) be done other than having a good developer guide. I'll list a few that came to my mind.

## Documentation

Again, it's documentation. So far I've made [4 commits](https://github.com/python/cpython/commits?author=laike9m) to improve, but there are a lot more problems out there. Reading documentation is often the first step to start contributing. If the documentation itself is wrong or outdated, it would confuse people and scare them away. The more problems the documentation has, the less people care about fixing it, since it's "already broken", and that's the so-called "[broken windows theory](https://en.wikipedia.org/wiki/Broken_windows_theory)".

## Specification

Bugs in documentation sometimes reveal a bigger problem: lack of specification. Personally, I think this is THE MOST important issue that Python needs to solve. To give an example, there is no formal definition for "value stack", but the word "stack" is used everywhere in bytecode's documentation, which represents the "value stack". Sometimes I got the feeling that the boundary between (C)Python's behavior and its implementation is vague, which makes it hard for both people to use and improve the language. Sure CPython is the reference implementation, but perhaps it can do better?

This topic is a bit beyond the scope of this article and I'm not an expert either, so I'll link to a recent discussion on Python Language Summit 2020: *[A formal specification for the (C)Python virtual machine](https://pyfound.blogspot.com/2020/04/a-formal-specification-for-cpython.html)*.

## Tools

When I was talking to Xiang Zhang, he mentioned he wish Python has gdb's functionalities built-into the language. I didn't understand why this is important, until one day I tried to use gdb on Mac to trace code execution in Python's VM. It was painful, and I didn't get it to work. Later I found that gdb has quirks on Mac and does not work well in certain cases. If Python provides this functionality, then perhaps Mac and Windows users are more likely to contribute?

Xiang also mentioned that he wish Python has hooks that provide memory usage and thread information. Could this be the consequence of lacking a formal specification, or simply because the community lacks interest in them? I don't know. But I do know these tools (if they exist) will not only benefit application developers but Python contributors as well.

## Roadmap

This is the hardest topic, and I honestly don't feel qualified to talk about it. But it has been on my mind for quite a long time, so I might as well give my two cents.

If you observe the lifetime of a programming language, there are two typical trajectories. If the language failed to gain enough popularity in the first few years after it was created, it gradually fades away. If it did, often with a few killer apps or certain traits that other languages don't have at the time, it becomes "mainstream". Once it gets there, its lifespan suddenly grows from less than a few years to, say, at least 30 years, or even longer (remember the [call for COBOL programmers](https://www.cnn.com/2020/04/08/business/coronavirus-cobol-programmers-new-jersey-trnd/index.html)?).

One problem arises with the long lifespan, that is, the language gradually becomes **outdated**. Why does this happen? Because human is progressing, errors are learned and new ideas are created. The creators of new languages learn from existing ones, avoid their mistakes, and adopt new ideas from academia and industry. Most will fail, but there are always a few who managed to create a better language. You'll probably argue that there's no language that's perfect in every way, and there are always tradeoffs, which is true. But hey, making better tradeoffs is also a technique, and people learn it as well. Meanwhile, we should not forget that old mainstream languages are also being improved constantly, but just as any other legacy system, having to keep backward compatibility slows this progress down, and even made certain improvements impossible.

Python is right at this stage. It has become one of the most popular programming languages, but some new languages are trying to take its place, little by little. It is unavoidable and we have to accept it, then take action. But how? C++ gives a great example. Take a look at this document:

***[Direction for ISO C++](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2018/p0939r0.pdf)***

And its abstract:

> This is intended as a document updated as times go by and new facts and opinions emerge. It tries to articulate our aims for C++, the challenges faced, and some concrete suggestions for priorities.
>
> This document differs from most papers by aiming for a global view and placing individual features in context, trying to avoid delving into technical details of individual proposals except where that “details” could affect the language as a whole or its tool environments.
>
> To prevent this document from becoming toothless “pure philosophy,” we propose giving priority to specific areas of development and to specific proposals in those areas.

I know, people are complaining about C++'s evolvement being slow, but let's not forget C++ is probably the most complicated language with way more historical burden than languages like Python. Despite all that, I feel this kind of "long term road map" is what Python needs. Again, there are tradeoffs here. Python does not have a committee, and everybody pretty much works on its own, which allows it to move fast. However, I've seen many people say that, despite Python gaining more and more features, they don't feel like upgrading the version they've been using, because there isn't much big of a difference. Many say they'll stick with 3.6, cause that's a version with major improvements (f-string, compact dict). While I know this is not true, that recent Python versions do carry many improvements they should care, I still think it is dangerous that people start to think that way. Part of the reason, IMHO, is because Python does not have a real road map, therefore most contributors just tend to focus on low-hanging fruits. There's nothing wrong with making small improvements, but certain aspects, like **performance improvements**, is not small, and it can only happen if there's careful long-term planning and thinking, aka, a roadmap. A roadmap  is not a magic scroll, it does not turn Python into a better language right away. In fact, it's a long process, and will probably take years before we can tell the difference. However, that's also the reason why Python needs it so urgently. Years can pass in the twinkling of an eye, if we don't do something now, in five years, the world can be totally different from what we see today.

**Edit**: Ruby is similar to Python in many ways, except it has a roadmap. See [Matz - Ruby 3 and Beyond](https://youtu.be/Fz9-CQnh2hA).

# Summary

This article is way longer than I expected, so I'll do a quick summary. In our latest [show](https://www.pythonhunter.org/episodes/ep14), my friend who currently works in the Google New York office talked about what he thinks makes a good manager. He said a good manager should create an environment for you to work effectively. And I think that's also true for Python.