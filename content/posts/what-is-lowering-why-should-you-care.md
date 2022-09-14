+++
date = 2022-09-14T06:57:07+02:00
title = "What is lowering and why should you care"
description = "A brief dive into what lowering your code means for you and for the compiler, and why should you care about it"
authors = ["Stefano Previato"]
tags = ["lowering", "compilers", "SharpLab", "C#", "performance", "high level language"]
categories = ["C#", "Compilers"]
+++

Nowadays we're all writing highly understable code (_or at least that's what I'm telling my coworkers!_) in high level languages, almost completely forgetting what those pieces of code are translated to once they are compiled or interpreted.

Back in the day, people used to punch holes into cards to represent bits as instructions for the CPU, then they invented low level languages, then high level languages, and now we even have high level languages that transpile to other high level languages instead of compiling them (think of TypeScript to JavaScript for example).

All this preface to say that we're used to write concise pieces of code, usually with the aid of complex constructs hiding underneath a couple of handy keywords (`async` and `await` come to mind for example, hiding an entire state machine when you use them) and ignoring how the compiler actually interprets them.

Speaking of C# for example (since that's the main language I use for work everyday), we are used to constructs such as `do ... while` or `async ... await` or even keywords such as `record` with C# 9.0. This means that we can write something silly like this without much effort:

```csharp
int count = 0;

do {
    Console.WriteLine("Hello!");
    count++;
} while (count < 3);
```

Ignoring the extreme dumbness of the code (that's not the point of this post), you may think that this is what the C# compiler reads and translates to the Intermediate Language (IL) ready to be run by the JIT compiler, but that's not actually true!

Before reaching the IL stage of compilation output, C# performs what is called **lowering**.

> Lowering consists of, internally, rewriting more complex semantic constructs in terms of simpler ones. For example, while loops and foreach loops can be rewritten in terms of for loops. Then, the rest of the code only has to deal with for loops.
> -- <cite>Walter Bright, author of the D programming language</cite>

As you can see from the definition, C# lowering means taking your code as an input and outputting C# but written in other, simpler terms.

Recalling the code example above, this is what the C# lowering outputs with that code input:

```csharp
int num = 0;
while (true)
{
    Console.WriteLine("Hello!");
    num++;
    if (num >= 3)
    {
        break;
    }
}
```

It may not seem much, but there are a couple of things to note there:

- the variable `count` has been rewritten into `num`
- the `do ... while` construct has been replaced with a simpler `while (true)` construct
- the loop exit condition has been moved inside the loop with an `if` statement followed by a `break` keyword

As you can see, the C# compiler rewrote our code into simpler to understand code, specifically to avoid dealing with more constructs.

You can see how your code gets lowered with the help of this awesome free tool called [SharpLab](https://sharplab.io/). This tool lets you select the C# version you want to compile, the configuration (_Debug_ and _Release_) to see how the code gets lowered even more, and also lets you see how your code is translated into IL or JIT Asm, for those of you that want to get deeper into how things work at a lower level.

In the next days I'll post a couple of other nice findings about C# lowering, since I think it's a nice topic to dive into and really brings more clarity in how to write more performant code in general.

Stay tuned!
