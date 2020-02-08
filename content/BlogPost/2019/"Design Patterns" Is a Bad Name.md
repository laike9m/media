There has been many [criticisms](https://en.wikipedia.org/wiki/Software_design_pattern#Criticism) on design patterns over the years. Generally speaking, people think those patterns are not that useful(or even, harmful), and can be simplified or eliminated if some features are directly supported by the programming language itself. However, I have yet to see any criticism on "design patterns", I mean, the name. So here's what I think:

**"Design patterns" is a really bad name.**

## "Design patterns" are not pattern

![](https://upload.wikimedia.org/wikipedia/commons/thumb/f/f0/Settee_MET_SF2007_368_img4.jpg/640px-Settee_MET_SF2007_368_img4.jpg)

This is a pattern —— the leaves on the carved woodwork repeat in a predictable manner. Normally, you would observe things and find a pattern.

"Design patterns" are not. It may originally come from observing the commonalities in **existing** code, but it has now become (at least that's what believed by most people) the guidelines people should follow when writing **new** code. See the difference here? A pattern is the outcome of an observation, they are not, and never meant to be the guidelines for creating new things. Why would you write code like others, just because they wrote it in a certain way? Are you solving the same problem? Do you know the requirements of your program? Do you know the requirements of their programs?

**The root cause here, is the word "pattern"**. We often say "follow a pattern", but that does not mean that "we should follow a pattern when writing new code", in contrast, it is describing a feature of old code, and the program you are about to write is NOT part of it. The word and the phrase are so misleading, it has caused so many programmers (especially Java programers) to believe from the start, that these "patterns" are the law of god, and they will be punished if they ever disobey them.

##  "Design patterns" are not about "design"

In Google, we write design docs for projects & features that we believe will take longer than a few weeks to accomplish. I've written and read many design docs, but I have not seen any design doc saying "we should use Singleton/Adapter/Proxy... in this project", not even once. Why? **Because it is not part of the design**. You should define how components communicate with each other, that's design, but nobody cares about whether it is using a singleton. I'm not saying singleton is a bad idea, it may or may not be, but that is what code review is there for, not design.

**Again, the word "design" misled so many people, just like "pattern".** The authors of GoF are truly geniuses, as I write this, I realized that there couldn't be a better name than "design patterns" that would made the concepts more popular and well accepted than it has ever become today. Let's imagine your friends, who knows nothing about programming, looking at this magical word. What would they think? I actually asked two friends, one thought it's the principles to design a product, the other said it's probably a math model, or a mental model to help you think about a problem. This is quite like what I expected. Like everyone, we understand words by examining their literal meaning. No one is a born programmer, therefore when seeing the word for the first time, we got the impression that it's "about design", it's "how we should model a problem". But actually, it's not, it's implementation details.

With that false impression in mind, some programmers (especially Java programers) tend to decide which design patterns to use before writing the first line of code. What's even worse is that, not many people have the habit of writing design docs, but they still believe they are doing the "designing" work, because they try very hard to pick what design patterns to use! Great job, GoF 👏👏

## What "design patterns" really are

"Design patterns" are just coding techniques. That's all. They are by no means more "advanced" or "higher-level" than other techniques, like short-circuiting.

> There are only two hard things in Computer Science: cache invalidation and naming things.
>
> -- Phil Karlton

So, my final conclusion is, "design patterns" could either be a terrible name, or a brilliant name, depending on how you look at it.