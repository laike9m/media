This blogpost describes the Python development setup that makes the most sense to me (and probably you as well) in 2022 and beyond. **By "Development Setup", I mean the local development environment**, and not about deploying apps on a server.

## The Best Setup, in A Nutshell

- For executables/utility tools (e.g. mypy, black), install with **system Python + [pipx](https://pypa.github.io/pipx/)**
- For project development, install different Python versions via **[pyenv](https://github.com/pyenv/pyenv)**
- For dependency management, use **[PDM](https://pdm.fming.dev/)** (PDM is installed via pipx)

I'll explain each choice in more details below.

## Why Use Both System Python and Pyenv Python?

One unusual thing you probably have noticed is that the setup consists of two ways to install Python: the system Python and Python installed by pyenv. Here, by "system Python", I mean the Python carried with your system, or the Python installed with a system package manager (e.g. `apt`, `brew`). Many articles suggested that we should never using system Python, so why still use it? Because we need **two independent systems for two different purposes**:

- For utility tools, **reliability** matters the most. You want them to always work when you need them. The less things they depend on, the better. System Python is the best choice, because they have the least dependencies. Also, for utility tools, It doesn't really matter which Python it is running on. Therefore even if system Python is behind the latest version, it is usually fine.
- For Python interpreters used by projects, **flexibility** is more important. You may need to upgrade them when a patch version fixed a bug, or using a specific Python version because that's what you have on the server. pyenv fits this purpose well.

## pipx

[pipx](https://pypa.github.io/pipx/) needs no introduction, it has become the de-facto way to install Python executables. But imagine if you install pipx with pyenv-installed Python. This creates a dependency like this:

```
mypy --> pipx --> 3.9.7 (pyenv) --> pyenv --> system Python
```

As you can see, the `3.9.7 (pyenv) --> pyenv` part is unnecessary. It also creates trouble when upgrading Python: you either have to reinstall pipx, or keep Python 3.9.7 forever since pipx and all the tools above relies on it.

## PDM

Now comes to the controversial part: **[PDM](https://pdm.fming.dev/)**. Disclaimer: PDM is my personal choice, and it's totally ok if you chose Poetry, Pipenv, or any other tools. Here are the reasons why I use PDM:

⚠️⚠️ *UPDATE 2023.5: I should clarify that, [PEP 582](https://pdm.fming.dev/latest/usage/pep582/) is good for developing libraries; **for applications, PDM's [virtualenv mode](https://pdm.fming.dev/latest/usage/venv/) should still be preferred.*** 

- **PDM helped me get away from virtual environments**  
  I used to have 10+ virtual environments, and I get a headache every time I saw the long list. It essentially makes it impossible to ever upgrade my project Python, because I really don't want to setup virtual environments again for every project.

  With PDM, I have no virtual environments. Each project directly depends on the pyenv-installed Python. I'm also free to upgrade patch versions (e.g. from 3.9.7 to 3.9.9), which allow me to always use the latest Python release.

  See also *[You don't really need a virtualenv](https://frostming.com/2021/01-22/introducing-pdm/)*.

- **PDM is well maintained**  
  PDM is created and maintained by PyPA member [Frost Ming](https://twitter.com/frostming90). If you don't know what PyPA members do, let's just say they know the best about Python packaging. Frost Ming also maintains Pipenv.

  I used to use Poetry, which is also a nice tool. But sometimes Poetry is buggy, especially when upgrading project Python. What's worse is that no one seems to be dealing with those bugs ([example](https://github.com/python-poetry/poetry/issues/3071)). PDM is the opposite. Issues are usually responded within a day or two, including [feature requests](https://github.com/pdm-project/pdm/issues/846).

  Benefit from being maintained by a PyPA member, PDM is usually the fastest in adopting new proposals in the Python packaging world, which lets its users stay on high productivity.

Overall, I find PDM better than other tools I've used, and it's constantly getting better ([5~6 releases](https://github.com/pdm-project/pdm/releases) per month). I [tweeted](https://twitter.com/laike9m/status/1443812946325348356?s=20&t=1AIVAsNRwQ_JTRJEQk9CkQ) about my transition from Poetry to PDM, might be interesting to read.

## Caveats

You may still need to install a new system Python when the old version gets deprecated. You can use `pipx reinstall-all` to reinstall existing tools, which is not so bad.

The setup may or may not work on Windows. If you can confirm which case it is, and/or if there's any issues, let me know, I'm happy to put them in the article.

## Summary

In the past few years, Python gained a lot of great improvements in the development/packaging ecosystem, but this also created a mess that even the most experienced developers had a hard time picking tools. Luckily, I've found the silver bullet that works for me, and I hope it would also work for you.