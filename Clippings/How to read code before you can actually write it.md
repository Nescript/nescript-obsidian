---
title: How to read code before you can actually write it
author:
  - "[[Fahim ul Haq]]"
published: 2026-01-28
created: 2026-03-24
categories:
  - "[[Clippings]]"
url: https://medium.com/@fahimulhaq/how-to-read-code-before-you-can-actually-write-it-a7d9ca88fe12
topics:
  - Code Reading
---

I’ve built products for software engineers for most of my career. Some of that work took place inside big tech companies, such as [Facebook](https://www.facebook.com/) and [Microsoft](https://www.microsoft.com/). Most of it has happened more recently at Educative, where my role is to help teams and individual developers [learn more efficiently](https://www.educative.io/learn-to-code?utm_campaign=persona_learn_to_code_q1_26&utm_source=medium&utm_medium=text&utm_content=fahim_persona_january&eid=5082902844932096), with fewer dead ends.  
我的职业生涯大部分时间都在为软件工程师打造产品。其中一些工作是在大型科技公司完成的，例如 [Facebook](https://www.facebook.com/) 和 [微软](https://www.microsoft.com/) 。最近，我的大部分工作是在 Educative 完成的，我的职责是帮助团队和个人开发者 [更高效地学习](https://www.educative.io/learn-to-code?utm_campaign=persona_learn_to_code_q1_26&utm_source=medium&utm_medium=text&utm_content=fahim_persona_january&eid=5082902844932096) ，减少学习过程中的死胡同。

Here’s a truth: a common issue for beginners is not writing code, but reading and understanding existing code.  
事实是：初学者常见的问题不是编写代码，而是阅读和理解现有代码。

Opening a large codebase for the first time can feel overwhelming: files scattered across directories, functions calling other functions, and classes that seem to sprawl indefinitely. I see this every week in product feedback, learner conversations, and team enablement calls.  
第一次打开大型代码库可能会让人感到不知所措：文件散落在各个目录，函数之间相互调用，类似乎无穷无尽。我每周都会在产品反馈、学员交流和团队赋能电话会议中看到这种情况。

People assume writing code is the main skill. But in the real world, especially as projects scale, you often spend more time reading code than writing it. That’s true whether you’re a junior dev, a staff engineer, or a founder reviewing a pull request (even during the holiday season).  
人们通常认为编写代码是主要技能。但在现实世界中，尤其是在项目规模扩大后，你花在阅读代码上的时间往往比编写代码的时间更多。无论你是初级开发人员、资深工程师，还是审查代码拉取请求的创始人（即使在假期期间也是如此），情况都是如此。

The following mental model has proven effective across many teams and learning contexts. It’s practical rather than academic. It’s built from real product work, real engineering stories, and the patterns that show up again and again.  
以下思维模型已在众多团队和学习环境中证明行之有效。它注重实践而非理论，源于真实的产品工作、真实的工程案例以及反复出现的模式。

## Why learning to code feels slower than expected<br>为什么学习编程感觉比预期要慢

When people say, “I’m stuck,” they usually mean one of these things:  
当人们说“我被困住了”时，他们通常指的是以下几种情况之一：

- “I don’t know where to start.”  
	“我不知道从哪里开始。”
- “I don’t know what this code is doing.”  
	“我不知道这段代码在做什么。”
- “I don’t know what matters and what doesn’t.”  
	“我不知道什么重要，什么不重要。”

That’s not a writing problem. That’s a reading problem.  
这不是写作问题，这是阅读问题。

Reading code is different from reading natural language. In natural language, it’s often possible to skim and still grasp the intent, reading linearly from start to finish. In code, skimming is unreliable. A single detail, argument, condition, or default value can change behavior entirely.  
阅读代码与阅读自然语言截然不同。在自然语言中，通常可以快速浏览并理解其意图，从头到尾线性阅读即可。但在代码中，快速浏览并不可靠。一个细节、一个参数、一个条件或一个默认值都可能彻底改变代码的行为。

When I was at Microsoft, I watched skilled engineers spend hours debugging a problem that turned out to be one overlooked edge case in a dependency. They weren’t bad at coding. They were human. They missed one line while reading.  
我在微软的时候，亲眼见过一些技术娴熟的工程师花了几个小时调试一个问题，结果发现问题出在某个依赖项中一个被忽略的特殊情况上。他们的编程水平并不差，他们也是人，阅读代码时也会漏掉一行。

I saw the same thing at Meta, except the scale made it even harder. When you’re working inside a massive codebase, the challenge isn’t writing a clever function. It’s figuring out *which* function matters, and what the system is quietly assuming around it.  
我在 Meta 也遇到过类似的情况，只是规模更大，难度也更大。在庞大的代码库中工作时，挑战不在于编写巧妙的函数，而在于弄清楚 *哪个* 函数才是关键，以及系统在它背后默默地做了哪些假设。

The hard part is that beginners don’t know what to look for yet. They think they need to understand everything. They don’t. They need to learn how to scan for meaning.  
难点在于初学者不知道该关注什么。他们以为自己需要理解一切，其实不然。他们需要学习如何快速浏览文本，找到其含义。

That’s what real readers do.  
这才是真正读者会做的事。

## The code reading ladder <br>代码阅读阶梯

If you try to read code like a book, you’ll burn out. A better approach is to climb a ladder, one rung at a time.  
如果你像读书一样读代码，你会精疲力竭。更好的方法是像爬梯子一样，一步一步地往上爬。

Here’s the ladder I teach internally when we’re building courses, reviewing onboarding content, or designing developer learning flows:  
以下是我在内部构建课程、审核新用户培训内容或设计开发者学习流程时所使用的步骤：

1. **What does this code touch?** (inputs and outputs)  
	**这段代码涉及哪些方面？** （输入和输出）
2. **What does it change?** (state and side effects)  
	**它会改变什么？** （状态和副作用）
3. **What does it assume?** (defaults and invariants)  
	**它做了哪些假设？** （默认值和不变式）
4. **Where does it fail?** (exceptions, errors, edge cases)  
	**它在哪里失效？** （异常情况、错误、极端情况）
5. **What does it hide?** (abstractions and libraries)  
	**它隐藏了什么？** （抽象和库）

If you can answer those five questions for a piece of code, you don’t need to memorize it. You *understand it*.  
如果你能回答关于一段代码的这五个问题，你就不需要记住它，你 *已经理解它了* 。

Let’s walk through how to build that skill practically.  
让我们一起来看看如何通过实践来培养这项技能。

## Trace one path <br>追踪一条路径

Once you have the map, [pick one path and follow it](https://www.educative.io/learn-to-code?utm_campaign=persona_learn_to_code_q1_26&utm_source=medium&utm_medium=text&utm_content=fahim_persona_january&eid=5082902844932096). Not five paths. Not the whole system. One path.  
拿到地图后， [选择一条路走下去](https://www.educative.io/learn-to-code?utm_campaign=persona_learn_to_code_q1_26&utm_source=medium&utm_medium=text&utm_content=fahim_persona_january&eid=5082902844932096) 。不要走五条路，也不要走遍整个系统。只走一条路。

Here’s a simple example that shows up everywhere:  
以下是一个随处可见的简单例子：

- User clicks a button.用户点击按钮。
- Button triggers a function.  
	按钮触发某个功能。
- Function calls an API.函数调用 API。
- API returns data.API 返回数据。
- UI renders the result.UI 渲染结果。
That flow exists in almost every product.  
这种流程几乎存在于所有产品中。

If you can trace one path end to end, your confidence changes. You start seeing code as behavior, not syntax.  
如果你能追踪代码从头到尾的路径，你的信心就会改变。你会开始把代码看作行为，而不是语法。

This is where code diverges from natural language. In natural language, reading typically starts at the beginning and proceeds linearly. In code, execution rarely begins at the first line of the file being read. It starts wherever the program is entered, such as a button click, a route handler, a scheduled job, or a main function, and then it jumps.  
这就是代码与自然语言的不同之处。在自然语言中，阅读通常从头开始，并按顺序进行。而在代码中，执行很少从被读取文件的第一行开始。它从程序进入的任何位置开始，例如按钮点击、路由处理程序、计划任务或主函数，然后跳转到该位置。

That’s why real code reading is often about finding the entry point and following the sequence of events that follow.  
这就是为什么真正的代码阅读通常是找到入口点并跟踪后续事件序列的原因。

This is also how professional debugging works. You set a point of entry, pause execution, and follow the chain. In Visual Studio’s [debugging docs](https://learn.microsoft.com/en-us/visualstudio/debugger/get-started-with-breakpoints?view=visualstudio), breakpoints are described as a way to pause execution to inspect variables and see the call stack. That call stack is the story of how you got here. Beginners often skip this and jump straight to guessing. Tracing beats guessing.  
这也是专业调试的工作原理。你设置一个切入点，暂停程序执行，然后沿着调用链追踪。在 Visual Studio 的 [调试文档](https://learn.microsoft.com/en-us/visualstudio/debugger/get-started-with-breakpoints?view=visualstudio) 中，断点被描述为一种暂停程序执行以检查变量和查看调用堆栈的方法。调用堆栈记录了程序是如何运行到此处的。初学者常常忽略这一步，直接进行猜测。但追踪调用堆栈比猜测要好得多。

## Learn to read inputs, outputs, and side effects<br>学会解读输入、输出和副作用

This is where most learners level up fast. If you’re reading a [function](https://www.educative.io/answers/what-are-functions-in-python?utm_campaign=persona_learn_to_code_q1_26&utm_source=medium&utm_medium=text&utm_content=fahim_persona_january&eid=5082902844932096), don’t ask, “What does every line do?”  
这是大多数学习者快速提升的地方。如果你在阅读一个 [函数](https://www.educative.io/answers/what-are-functions-in-python?utm_campaign=persona_learn_to_code_q1_26&utm_source=medium&utm_medium=text&utm_content=fahim_persona_january&eid=5082902844932096) ，不要问：“每一行代码的作用是什么？”

Ask:问：

- What goes in?里面放什么？
- What comes out?出来的是什么？
- What does it change?它改变了什么？

That last part matters. Side effects are the silent killers of understanding.  
最后一点至关重要。副作用是理解过程中悄无声息的杀手。

Side effects include:副作用包括：

- Modifying a global variable  
	修改全局变量
- Mutating a list or object passed in  
	修改传入的列表或对象
- Writing to a file or database  
	写入文件或数据库
- Making a network call 拨打网络电话
- Logging or emitting metrics  
	记录或发出指标

If you train your eye to spot those quickly, code becomes less mysterious.  
如果你训练自己快速发现这些错误，代码就不会那么神秘了。

A lot of “hard” code isn’t hard because it’s complex. It’s hard because it changes state in ways you don’t expect.  
很多“难”的代码之所以难，并不是因为它们复杂，而是因为它们会以你意想不到的方式改变状态。

## Spotting assumptions <br>发现假设

This is where things get real, because code is never just a set of instructions. It’s full of assumptions.  
这才是真正考验人的地方，因为代码绝不仅仅是一组指令，它充满了各种假设。

Sometimes those assumptions are obvious and easy to spot, such as when the code treats an array as if it will always contain something inside it, or when it expects every user record to include an email address. Other times, the assumptions are hidden in plain sight, in default values, in time-outs you didn’t know existed, in type conversions that happen automatically, or in shared objects that quietly change underneath you.  
有时，这些假设显而易见，很容易发现，例如代码将数组视为始终包含数据，或者期望每个用户记录都包含电子邮件地址。而有时，这些假设却隐藏在显而易见之处，例如默认值、你不知道存在的超时设置、自动发生的类型转换，或者在你不知不觉中发生变化的共享对象。

And the reason assumptions matter is simple: when they hold, the code appears to be fine. When they break, bugs appear that seem random.  
假设之所以重要，原因很简单：当假设成立时，代码看起来一切正常；当假设不成立时，就会出现看似随机的 bug。

When assumptions hold, everything appears to be fine. When they break, you get bugs that feel random. One of the most notable examples is the infamous Python pitfall: mutable default arguments.  
当假设成立时，一切似乎都很正常。但当假设失效时，就会出现一些看似随机的 bug。其中一个最著名的例子就是臭名昭著的 Python 陷阱：可变默认参数。

I’ve seen this exact bug confuse beginners and experienced devs alike. It looks harmless. It isn’t.  
我见过这个漏洞让新手和经验丰富的开发者都感到困惑。它看起来无害，但实际上并非如此。

```c
def add_item(item, items=[]):
    items.append(item)
    return items
```

Because the default list is created once and reused, items keeps growing across calls. That means the function doesn’t behave like a fresh, independent helper anymore; it behaves like it’s remembering past executions. Practically, that’s troublesome because you’ll get output that looks wrong even when the input looks correct. A beginner might call this function twice and think the second call is broken, when the real issue is hidden state leaking between runs.  
由于默认列表只创建一次并重复使用，因此每次调用后列表项都会不断增长。这意味着该函数不再像一个全新的、独立的辅助函数那样运行；它的行为就像是在记住之前的执行结果。实际上，这很麻烦，因为即使输入看起来正确，输出结果也可能看起来错误。初学者可能会调用此函数两次，并误以为第二次调用出错，而真正的问题是两次调用之间隐藏的状态泄漏。

The fix is to use None:  
解决方法是使用 None：

```c
def add_item(item, items=None):
    if items is None:
        items = []
    items.append(item)
    return items
```

The fix works because None is just a placeholder; it doesn’t create a list up front. Instead, the function checks: did you pass a list, or not? If you didn’t, it creates a brand new empty list inside the function. That guarantees every call starts fresh, instead of accidentally sharing the same list across runs.  
这个修复之所以有效，是因为 \`None\` 只是一个占位符，它不会预先创建列表。相反，函数会检查：你是否传递了一个列表？如果没有，它会在函数内部创建一个全新的空列表。这样就保证了每次调用都从头开始，而不是意外地在不同运行之间共享同一个列表。

And that’s exactly the kind of detail good code readers learn to catch: when does something get created, and does it get reused? When you read code, always ask: What is this function assuming will be true?”  
而这正是优秀代码阅读者需要关注的细节：何时创建了某个对象，以及它是否会被重用？阅读代码时，务必问自己：“这个函数假设哪些条件为真？”

## Use tools that make reading easier<br>使用让阅读更轻松的工具

This is the part where I get practical. Many developers treat tools as if they’re only for productivity. In reality, the best tools improve understanding.  
接下来我要讲讲实际问题。很多开发者把工具仅仅当作提高效率的工具。但实际上，最好的工具能够提升理解力。

Here are three I recommend to learners and teams because they’re “reading accelerators.”  
我向学习者和团队推荐以下三个工具，因为它们是“阅读加速器”。

**1) Breakpoints:** Breakpoints force code to slow down and reveal itself. They turn execution into a step-by-step process.  
**1）断点：** 断点会强制代码减慢运行速度并逐步显示其内容。它们将执行过程变成一个循序渐进的过程。

**2) The debugger statement in JavaScript:** In JavaScript, you can drop a debugger into your code and let the browser pause execution right there. It’s a simple move that makes code readable in motion.  
**2) JavaScript 中的调试器语句：** 在 JavaScript 中，你可以将调试器插入到代码中，让浏览器立即暂停执行。这是一个简单的操作，可以让代码在运行过程中保持可读性。

**3) Python introspection:** [Python](https://www.educative.io/courses/learn-python?utm_campaign=persona_learn_to_code_q1_26&utm_source=medium&utm_medium=text&utm_content=fahim_persona_january&eid=5082902844932096) gives you a huge advantage when reading code: you can inspect objects while the program runs. The inspect module can show you live object details and even retrieve the source. You can also use help() and pydoc to explore documentation directly from the interpreter when you keep hitting unfamiliar functions.  
**3) Python 自省：** [Python](https://www.educative.io/courses/learn-python?utm_campaign=persona_learn_to_code_q1_26&utm_source=medium&utm_medium=text&utm_content=fahim_persona_january&eid=5082902844932096) 在代码阅读方面提供了巨大的优势：你可以在程序运行时检查对象。\`inspect\` 模块可以显示对象的实时详细信息，甚至可以获取源代码。当你遇到不熟悉的函数时，还可以使用 \`help()\` 和 \`pydoc\` 直接从解释器中浏览文档。

## Where AI tools help, and where they don’t <br>人工智能工具在哪些方面能发挥作用，在哪些方面又不能。

AI has changed the way people interact with code. But it has also created a new failure mode: skipping reading entirely.  
人工智能改变了人们与代码交互的方式。但它也带来了一种新的失败模式：完全跳过阅读。

AI can generate code quickly. That’s real. It can also explain code, summarize files, and suggest improvements.  
人工智能可以快速生成代码。这是真的。它还可以解释代码、总结文件并提出改进建议。

But AI-generated code or explanations can still be wrong in ways that are hard to notice; they may miss edge cases, misunderstand business rules, or gloss over assumptions that matter in your specific system. That’s why it can’t replace your understanding of the context around the code.  
但是，人工智能生成的代码或解释仍然可能出现难以察觉的错误；它们可能会忽略一些极端情况，误解业务规则，或者忽略在你的特定系统中至关重要的假设。这就是为什么它无法取代你对代码上下文的理解。

When using AI to learn how to read code, a single rule applies:  
使用人工智能学习如何阅读代码时，只需遵循一条规则：

*Use it as a guide, not as a key to answers.  
请将其作为指导，而不是答案的关键。*

For example:例如：

- Ask AI to explain what a function does.  
	让 AI 解释某个函数的功能。
- Then verify it using docs or a debugger.  
	然后使用文档或调试器进行验证。
- Then rewrite the explanation in your own words.  
	然后请用你自己的话重写这段解释。

That last step is the difference between “I got help” and “I learned.”  
最后一步决定了“我得到了帮助”和“我学会了”。

And when you need a source of truth, lean on official documentation. For JS language details and debugging fundamentals, MDN’s documentation is the standard reference. For Python, the [official standard library docs](https://docs.python.org/3/library/index.html) are your anchor.  
当你需要权威的信息来源时，请参考官方文档。对于 JavaScript 语言细节和调试基础知识，MDN 文档是标准参考。对于 Python， [官方标准库文档](https://docs.python.org/3/library/index.html) 则是你的指路明灯。

AI can accelerate tasks. Documentation still defines boundaries.  
人工智能可以加速任务完成。文档仍然定义了界限。

## Reading code is how you stop feeling behind<br>阅读代码能让你不再感到落后。

For early learners, one point is worth stating explicitly: there is no prerequisite level required before reading production code. Reading production code is what builds that readiness.  
对于初学者来说，有一点值得明确指出：阅读生产代码不需要任何先决条件。阅读生产代码本身就能培养阅读能力。

Tutorials can feel productive, but they often remove the messy parts of the process. Real repos don’t. They include history, trade-offs, and decisions that made sense under pressure. And once you learn to read that kind of code, you’re not just learning syntax, you’re learning how software is actually built.  
教程或许能让人感觉很有帮助，但它们往往会忽略开发过程中那些繁琐的部分。而真正的代码仓库则不然。它们包含了开发历史、权衡取舍以及在压力下做出的合理决策。一旦你学会阅读这类代码，你学习的就不仅仅是语法，而是软件的实际构建方式。

If you want a simple plan, start small: read one file a day for ten minutes. And if ten minutes feels unrealistic, because you’re busy, you’re tired, or the codebase is using a stack you’ve never seen before, that’s fine. Start with five. Pick one function. Trace one path. Write down what you think it does. Those tiny reps compound faster than you expect.  
如果你想要一个简单的计划，那就从小处着手：每天花十分钟阅读一个文件。如果你觉得十分钟不太现实，比如你很忙、很累，或者代码库使用了你从未见过的技术栈，那也没关系。那就从五个文件开始。选择一个函数。追踪一条路径。写下你认为它的作用。这些小小的重复练习累积起来的速度比你想象的要快得多。