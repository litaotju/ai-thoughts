---
layout: post
title: "自然语言编程：agent 是操作系统，prompt 是程序"
date: 2026-02-24
---

用了几个月的 agent 之后，我越来越觉得，和 agent 系统打交道本质上是在编程——只不过编程语言换成了自然语言。agent 是操作系统，我们写的 prompt 是跑在上面的程序。

这个类比不是比喻，是真的可以一一对应的。以下是我在实际使用 NVCortex（我自己搭的 Slack-native agent）过程中总结出来的几点体会，每一条都能找到传统编程语言里的对应概念。

---

## 一、自然语言可以和脚本语言直接混用，没有 FFI 问题

传统语言之间互相调用，需要解决 ABI 兼容性和 FFI（Foreign Function Interface）问题。C 调 Python 得写 binding；Rust 调 C 得处理内存模型差异。麻烦。

自然语言里这个问题不存在。你可以在 prompt 里直接内嵌一段 bash 或 Python，agent 能无缝理解你的意图，然后去执行。比如：

```
分析一下这个目录下所有 .py 文件的行数，用这个命令：
find . -name "*.py" | xargs wc -l | sort -n
然后告诉我哪些文件超过了 500 行。
```

自然语言和脚本语言共存在同一条指令里，没有任何边界摩擦。这是自然语言作为编程语言天然的优势之一。

---

## 二、软链接与延迟加载：用一句话引用外部文件

传统编程里有软链接、`#include`、`import`——不用把所有内容都 inline，只需要一个引用，运行时自己去解析。

自然语言可以做到同样的事：

```
Read the file ~/.nvcortex/memory.md and follow the instructions inside.
```

一句话，agent 自己去读文件、理解内容、执行指令。

但更有意思的变体是**延迟加载**。在给 NVCortex 设计 agent memory 的时候，我需要让 agent 在对话时能读取历史记忆。一个自然的想法是硬编码注入——每次初始化 session 时，在 Python 里读取 `memory.md` 并塞进 context。但实际上我不需要改任何代码，只需要在 `base.md`（agent 的 system prompt 文件）里加一句：

```
When you need context about the user's preferences or past interactions,
read ~/.nvcortex/memory.md on demand.
```

这句话是一个 `if` 语句——判断条件是自然语言语义，执行是 tool call。agent 只在真正需要的时候去读文件，不需要每次都注入。比传统软链接更灵活：不是简单的内容引用，而是一个带条件的、动态的行为触发。

---

## 三、天生是脚本语言，不需要编译

编译型语言需要 build、link、deploy，才能看到结果。脚本语言写完直接跑，迭代快。自然语言更进一步——连语法检查都没有，写完直接发，agent 解释执行。

这带来了极快的迭代速度。想改一个行为？改 `base.md` 里的一句话，立刻生效，覆盖所有 session，不需要重启，不需要部署。我在开发 NVCortex 的过程中，有不少原本以为需要写代码的功能，最后发现只需要改一个 markdown 文件就搞定了。

当然，"没有编译"也是双刃剑——没有静态检查，意味着很多错误只有在运行时才会暴露（后面第四点会讲）。

自然语言还有一个传统脚本语言没有的特性：**语义随 context 压缩**。在充分建立了共享上下文之后，一个词就能表达完整的意图。我和 NVCortex 的对话里，很多指令只有两三个字：

- "Do it"（去做之前讨论的那件事）
- "?"（我看不懂你在做什么，解释一下）
- "Are you live?"（测试 agent 是否在线）

这是高 context 密度下自然语言的极致压缩——和传统语言的 verbosity 完全相反。

---

## 四、Prompt 也有 bug，而且更难发现

很多人不把 prompt 当代码对待，这是个代价高昂的误解。

代码里的 bug 通常有明确的报错；**prompt 里的 bug 往往是行为偏差**——agent 没有报错，它只是做了一件和你预期稍有不同的事情，而你可能根本没注意到。

最典型的是 **silent failure**：我请 agent review 一个 MR，它洋洋洒洒给了一堆分析——monkey-patching 不稳定、filter 可能 mask error——听起来很有道理。但仔细一看，它根本没读 diff，只是在做泛化的模式匹配。我当场说了一句："Dont think, answer my question directly." 然后让它重做，这次强制要求先读代码。第二次的分析完全不同——具体到文件名、行号、实际的风险点。

两次回答都不报错，第一次却是完全错误的。这是 prompt 最危险的 bug 类型：**看起来合理，但没有根据**。

常见的 prompt bug 还有：

- **逻辑歧义**：一句话可以被合理解释成两种截然不同的行为
- **边界条件没覆盖**：在 happy path 上表现完美，edge case 静默失败
- **隐含假设**：你以为 agent 知道某个上下文，其实它不知道
- **指令冲突**：两条规则在某些情况下互相矛盾，agent 随机选一条执行

treat prompt 像 treat 代码一样严肃——写完要测试，发现行为异常要 debug，重要逻辑要做回归。

---

## 五、Prompt 也有继承链

传统代码有继承、组合、接口。自然语言 prompt 也可以有同样的结构。

NVCortex 的 prompt 体系是这样的：`base.md`（始终加载）→ role 文件（`private.md`、`public.md` 等，按场景切换）→ skill 文件（按需加载）。这是一个三层继承链：base 定义全局行为，role 定义角色人格，skill 定义具体能力。

这个结构直接影响设计决策。当我在设计 memory consolidation agent（一个定时运行的后台任务，用来提炼对话记忆）的时候，要考虑它该不该继承 `base.md`：继承意味着它会带上我所有的 persona 和 identity 信息，而这些对一个只负责读写文件的后台任务完全是噪声。最终决定让它用一个完全干净的 role 文件，零继承——这和代码里"不要让 utility class 继承业务 base class"是同一个道理。

搞清楚 prompt 继承链的结构，是写出可维护自然语言系统的前提。

---

## 六、CLAUDE.md 是 agent 时代的 API 文档

传统软件的模块接口靠头文件（`.h`）、type stubs（`.pyi`）、或者 OpenAPI spec 来描述。使用者看接口，不看实现。

agent 时代，模块的"接口文档"就是 `CLAUDE.md`。当你告诉 agent "用这个代码库做 X"，你不需要教它怎么用——你只需要确保代码库里有一个写得好的 `CLAUDE.md`，agent 自己去读，自己 figure out 怎么调用。

我在 NVCortex 的开发里大量用了这个模式：每个功能模块有对应的 Markdown 说明，agent 在需要的时候自己去读。这意味着"如何使用一个模块"这件事不再需要人工介入，也不需要硬编码。

`CLAUDE.md` 就是 agent 时代的 ABI contract——只不过是用自然语言写的，比任何机器可读的接口描述都更灵活、更具表达力。

---

## 七、同一个功能，可以用代码实现，也可以用 prompt 实现

这是最惊喜的发现，也是实践中感受最深的一条。

在 NVCortex 的开发过程中，我养成了一个习惯：遇到新需求，先问"需不需要改代码？"这个问题。很多时候答案是不需要——只要改 prompt。

一个具体的例子：agent memory 的注入方式。传统做法是在 Python 里硬编码，每次初始化 session 时读取文件并注入。但正如前面第二点提到的，一句 prompt 就能实现等价效果，还更灵活（延迟加载，按需触发）。**不改一行 Python，只改一个 markdown 文件，行为完全等价。**

另一个例子：NVCortex 有一个定时任务系统。添加一个新的 cron job，传统做法是写配置代码。但在我的系统里，两个生产级定时任务是在一段不到 10 轮的自然语言对话里创建的——没有打开过任何代码编辑器。

这个发现改变了我对 agent 系统开发的思路。很多时候你面对的不是"要不要写代码"的问题，而是"这个逻辑放在代码里还是 prompt 里"的权衡：

- **代码**：可靠、可测试、行为确定、适合 hard constraint
- **Prompt**：灵活、迭代快、适合 soft behavior、改起来零成本

关键是把 prompt 做成 reusable blocks——就像函数和模块一样，可以组合、复用、版本管理。当你开始认真对待 prompt 的模块化，很多原本需要写代码的地方，其实一段精心设计的 prompt 就够了。

---

## 总结

自然语言编程不是未来，是现在。它有自己的语法（只是隐式的）、有自己的设计模式、有自己的 bug 类型，也有自己的工程最佳实践。

- 它可以和传统脚本语言无缝混用，没有 FFI 摩擦
- 它支持软链接、延迟加载、条件控制流
- 它天生是解释执行的，迭代极快
- 它有 bug，而且比代码更难发现——因为 agent 不报错，只是行为偏差
- 它有继承链和模块结构，值得认真设计
- 模块的 API 就是 markdown，自然语言写的 ABI contract
- 同一个行为，往往可以用代码或 prompt 实现，选哪个是工程权衡

如果你还在把 prompt 当成"随便说几句话告诉 AI 做什么"，是时候升级认知了——它是你在 agent 这个操作系统上运行的程序。值得认真设计，值得认真维护。
