---
title: "拆解 Agent Harness 可观测：解决智能体黑盒、排障、成本管控全方案"
source: "https://mp.weixin.qq.com/s/igw3l9op3rNj0NwttnL6ow?open_in_browser=true"
author:
  - "[[人人都是程序猿]]"
published:
created: 2026-07-07
description:
tags:
  - "AI Agent"
  - "Agent工程"
  - "可观测性"
  - "Harness"
---
人人都是程序猿 *2026年6月29日 12:00*

自今年伊始，Agent 的基础执行环境在性能方面已臻完善，故而诸多产品逐渐迈向实际生产阶段。此时，确保 Agent 能够长期稳定运行，以及正确执行长链路复杂任务，便显得尤为关键。

现阶段，所有围绕 Agent 工程架构的技术被统称作 Harness。关于 Harness 的定义，我们此前已做过概括性介绍。而今日，我们将话题聚焦，着重探讨“Agent 的执行可观测性”这一课题。

![图片](https://mmecoa.qpic.cn/sz_mmecoa_jpg/oTVZYCicttut7dhfJA04OSPjWMeLOfxrD8ibIUrDft8qc80affEbMY5sOlFIzB7B9sPWuW34QKjv7ZLFcjuic7sv7fCEgic5P3WwbT5uicOKW2y0/640?wx_fmt=jpeg&from=appmsg&tp=webp&wxfrom=10005&wx_lazy=1#imgIndex=0)

该课题的灵感源自某产品负责人学员的一番感慨：“我终于明白，为何我搞不懂公司里那批程序员的工作内容了。他们在进行技术架构设计时，秉持的是 AI Max 思路：若一项开源技术行不通，便更换另一项；若单智能体无法满足需求，就尝试多智能体。在尝试过所有方案后，便宣称 AI 已达上限，毫无优化空间。待有新的开源技术出现，便又重复这一过程。”

我有时着实好奇，忍不住询问他们如何量化上限，是否有过程方法论。而这些程序员则表示无法量化，也难以沉淀，只需运行他人的成果即可。我总感觉其中存在不妥之处，却因自身知识匮乏，难以道出所以然，只能听之任之。如今，事实证明此路不通，那就由我来为他们规划技术路径！

近期，我一直投身于 Mini - Openclaw 这一 Agent 项目的开发工作。目前，该项目的基本架构已初步搭建完成，涵盖模型、工具、技能、记忆管理、会话压缩以及多 Agent 协作等方面。

接下来，我打算开发一些工具和技能，并通过实际案例进行测试，以评估 Agent 的性能，检验其能否顺利完成任务。我首先选取了一个简单的任务，即让 Agent 撰写一篇关于 AI 手机的文章。此任务极为简单，仅需调用 write\_file 工具，将内容写入指定文件即可。然而，Agent 却不断提示“Error: 缺少 ‘path’ 参数”，文件始终未能成功写入。

看到这一提示，我推测可能是路径出现问题，或许是目录不存在，亦或是工具参数描述定义不够清晰。我仔细检查了相关代码，并未发现明显的错误。依据我以往的软件开发经验，我需要打印日志输出，以查看模型的返回结果以及工具出错的原因。

![图片](https://mmecoa.qpic.cn/mmecoa_jpg/oTVZYCicttuuiaPia4NmocbzzJEZTlcD3Y6EiakFJ0ua7S5IUddZW6NmkM20U2cgr0wK2ZoL72C2LqYibgU9HQQpCDDzRndHBvW9nVxx9UyVmL4k/640?wx_fmt=jpeg&from=appmsg#imgIndex=1)

于是，我着手开发了 Agent 执行日志和模型日志这两个模块，用于记录和显示相关信息，以便查看模型的输入输出以及工具的执行结果。功能开发完成后，我再次执行撰写文章的任务，很快便找到了问题的根源：模型连续多次将工具参数封装成了错误的 \_raw 结构。工具期望的是结构化参数，如 path 和 content，而模型却将它们塞进了 \_raw 字符串中。由于工具无法获取所需字段，任务自然以失败告终。

这个问题其实并不复杂，只是模型返回的参数格式有误，程序只需进行一定的兼容处理即可。至此，或许有人会认为，今后遇到问题查看刚刚开发的两个日志面板就足够了。但实际情况并非如此，我们还需关注 Agent 是否按照预期完成任务，以及其耗时和成本等情况。因此，为 Agent 构建一个可观测的面板显得尤为必要。

那么，什么是可观测性呢？可观测性其实不难理解，即系统运行时，我们能否从外部洞悉其内部发生的状况。在开发其他软件时，常见的三件套为指标、日志和链路追踪。指标能够告知我们服务的运行速度、花费成本以及错误率等信息；日志则记录具体发生的事件；链路追踪可记录调用关系，明确谁调用了谁，从而找出系统的瓶颈所在。将这三者结合起来，基本能够满足大部分后端系统的故障排查需求。

![图片](https://mmecoa.qpic.cn/sz_mmecoa_jpg/oTVZYCicttutp8fBgISo9apia0RO9JPLjqzdGGMWnuwiae7m2icUZ1kJOP5IWaG8u4EiazfvvGWZt87EMknaMLRLepicGKj3u2qE096mNPLibmaLwU/640?wx_fmt=jpeg&from=appmsg#imgIndex=2)

非 Agent 程序具备一个显著特性，即它们的执行路径是固定的。一旦出现错误，我们能够重新运行程序以复现问题，进而借助前文提及的“三件套”来排查问题。然而，Agent 则有所不同。面对同样的问题，每次的执行路径都可能大相径庭，每次输出的内容也不尽相同，甚至工具的选择也会有所差异。那么，究竟该如何设计 Agent 的可观测性呢？

在开启这个话题之前，我们不妨先探讨一下模型的能力边界问题。

### 模型的能力边界

首先，需要明确的是，可观测性并非 Agent 所独有的特性。只要是 AI 项目，都会面临可观测性的难题，这其中就包括最为传统的知识库 RAG 项目。不过，无论哪种项目，其底层逻辑均为前文所述的“三件套”。

在给学员授课时，我最常说的一句话便是：做 AI 应用一定要了解模型边界！这里所说的模型边界，涉及到 AI 应用的两个流派：AI Max（能用 AI 就用 AI）和 AI Min（能不用 AI 就不用 AI）。所谓的可观测性，仅在“能不用 AI 就不用 AI”的模式下才可行，这背后体现的是对模型边界的认知：追求完美的准确率是不现实的，关键在于要清楚错在哪里、为何出错以及如何改正，并且能够证明技术框架是闭环且可重复的。

在此，为大家举个例子。之前在开展 AI 课程时，由于学员数量过多，需要一个排班系统。大致需求如下：学员在微信群中告知自己每天的空余时间，AI 会主动统计大家都有空的时间，若满足条件则预约会议。学员在群里的聊天信息如下：

- A：20.00 - 22.00 有空
- B：18 - 20 点没空，其他都可以
- C：二十点后可以
- D：下午 4 点前没空
- E：我随便了，都行

这是一个极为简单的需求，但就是这样一个简单的系统，足以阐释什么是模型边界。

如果采用“能用 AI 就用 AI”的 AI Max 模式，操作会非常简单，直接将所有信息一股脑地丢给模型，并加上一句“请问今天我该安排什么时间上课”即可。例如，在简单场景下，AI Max 是最优解，像许多智能体（如 Manus）在简单任务中的表现就十分出色。

接下来是“能不用 AI 就不用 AI”的模式，也就是最小化 AI 应用。所谓最小化 AI 应用，就是仅在不得不使用 AI 的地方使用它。比如在这个例子中，不得不使用 AI 的地方就是提取关键词，即通过语义识别每个学员的空闲时间。

A：其空闲时间段为 20:00 - 22:00，也就是晚上 8 点至 10 点。  
B：18:00 - 20:00 无空闲时间，其余时间皆有空，即 00:00 - 18:00 以及 20:00 - 24:00 处于空闲状态。  
C：二十点之后时间充裕，意味着 20:00 - 24:00 为空闲时段。  
D：下午 4 点（即 16:00）之前没有空闲，所以 16:00 - 24:00 是其空闲时间。  
E：全天 00:00 - 24:00 均处于空闲状态。

获取到这些空闲时间后，我们可以自行运用算法来实现相关功能。此时，另一个问题随之而来：在最小化 AI 应用的场景中，何时需要借助 AI 呢？答案其实很简单，当面临充满泛化性的场景时就需要用到 AI。就像上述 ABCDE 学员的回答，很难运用正则方法进行匹配。类似这种关键词（关键知识）的提取，只能依赖 AI 来完成。

与之类似的场景还有，我要求学员的昵称必须遵循学号 - 昵称 - 城市的格式，但学员给出的格式却五花八门，诸如学号\_昵称\_城市、城市\_学号\_昵称、学号昵称@城市等稀奇古怪的排布方式。在学员自行设置之后，也唯有 AI 能够迅速帮助他们进行更正。所有这类对泛化要求较高的情况，往往都需要 AI 发挥作用，并且 AI 在这一领域的表现颇为出色！

基于此，我们便可以探讨可观测性的相关问题了：

那么，在这个基础之下，就可以讨论可观测性了：

#### 可观测性

现阶段的 AI 项目其实有个巨大不确定性因素， **因为大模型本身就是个巨大黑盒（还是概率输出）** 。

还是以上面排班系统为例，可观测性的价值在于： **如果出现了AI识别不了的情况，能很快识别并解决！**

比如现在出现一个F，他给的答案比较另类：

```
戌亥之时，余有暇。
```

类似于这种回答，模型很可能识别不了，那么排班系统就会出问题，这个在能不用AI就不用AI的模式下就可以被识别并优化。

这里的可以被识别且优化就是我们所谓的模型能力可观测。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_jpg/6Uzn2S5AAySMGZHmS220aKg6t1PayYqFPY0Z61I5OUQhYvUMsrnF6VFI6DKlIC5rGaBV9K7KhSlhkBsSZGKwuFzohNPX8t4MW0Akpia8vfZY/640?wx_fmt=jpeg&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1#imgIndex=5)

上述是比较泛的模型可观测性介绍，理解他们后，我们再回归主题，进入 Agent 可观测性的讨论：

#### Agent 可观察性

我们参考了其他系统的可观察性加上 Agent 的特性，Agent 的可观察性大概包含以下几个部分：

*原始日志数据* ，记录会话历史，模型的输入和输出，记录思考过程，工具的输入和输出

*指标聚合* ，设计对应的指标，回答 服务有多快，花费了多少钱，错误率多少等等

*Trace 调用树* ， 回答谁调谁，调用链是什么样的

*决策归因* ，这一步 **为什么** 要这么做？考虑过别的吗

*任务状态流转* ， 目标拆成了什么？哪一步在哪个状态？计划改了几次？

*异常检测* ， Agent 走了哪条路径？有没有打转 / 反复重试 / 死循环？

*评估* ， 这次到底成功了吗？输出对不对？

*回放与对比* ， 改一行 prompt，重跑这个 case，看会不会更好？

接下来我们逐步展开讨论：

![图片](https://mmecoa.qpic.cn/sz_mmecoa_jpg/oTVZYCicttusOwMBicbnlXJhbKf6VYVbO1DmG8ZrDzxWBZR9pGibpjFvrWDKO1qvNvWMb9pg9yXBz4icWJV4BrjR0kSsBiaN70h4CxKUJhBVAXXQ/640?wx_fmt=jpeg&from=appmsg#imgIndex=4)

## 原始数据的记录

我们精心开发的 Mini - OpenClaw 是一个具备单机运行、自托管特性且采用文件存储方式的项目。在每一个会话中，都会保存一份 jsonl 文件，其中囊括了用户请求消息、模型的请求与响应、工具的输入与输出、异常状况、会话压缩情况以及评估结果等原始内容。这些数据皆为原汁原味的原始数据，未经过任何形式的加工处理，仅仅如实地记录了 Agent 在执行过程中所发生的所有动作。

![图片](https://mmecoa.qpic.cn/mmecoa_jpg/oTVZYCicttuu4svhPGmzk5XfJ9ibuel0YMVyRgtNSBKNV6etr9IGh7AZiae26lhwXnn7c46Mhlib48xkOpc1zf3skFPmTDCxvxzh7icicdngP3nz8/640?wx_fmt=jpeg&from=appmsg#imgIndex=5)

以下几类事件会被优先留存：

- model\_call 与 model\_result：用于记录模型的输入输出、token 数量、运行耗时以及模型名称。
- tool\_call 与 tool\_result：负责记录工具的名称、参数、执行结果以及出现的错误。
- 上下文和状态的变化：例如会话压缩操作、任务状态的迁移等情况。
- 观测系统自行生成的事件：诸如 anomaly（异常）、evaluation（评估）等。

我们将这类基础数据划分为两个部分。一部分是关于模型的输入和输出，借助这部分数据，能够便捷地查看提示词是否按照我们预先设计的格式提供给了模型；另一部分则用于记录 Agent 的执行日志，通过它可以按照时间线来查看 Agent 的执行流程，我们能够依照时间的先后顺序，完整地了解 Agent 是如何一步一步进行思考，并调用工具以完成任务的。

![图片](https://mmecoa.qpic.cn/mmecoa_jpg/oTVZYCicttuv78cDkV0r0uwoTGfXGUicJAUDAib99fCV8YnY5cc4qRmnH2za5Uo9p8r7290RVFcakgUJTWEicxHhvz7Ks3amS1WaRN3cXXFhibMQ/640?wx_fmt=jpeg&from=appmsg#imgIndex=6)

## 指标设计

有了原始数据，各项指标便能够轻松计算得出。那么，我们究竟需要哪些指标呢？我们简单设计了以下几个指标：

- 工具错误率
- 模型调用耗时
- token 消耗
- 上下文压缩是否过于频繁
- 成本评估

这些指标具有重要的作用，它们能够提示我们系统中可能存在异常的地方。例如，若工具错误率偏高，我们就需要深入排查工具出错的原因；要是上下文压缩过于频繁，我们则要考虑上下文窗口的设置是否合理，或者压缩算法是否存在问题。然而，指标虽能让我们知晓哪里出现了问题，但却无法明确具体的错误根源。比如，工具错误率达到 30%，仅能表明工具调用频繁失败，却无法解释失败是由于工具实现存在缺陷，还是模型传递的参数有误，亦或是工具描述让模型对 schema 产生了误解。同样，平均迭代次数升高，也不能直接说明 Agent 陷入循环的原因，有可能是任务本身的难度增加了，也有可能是它一直在重复同一个错误。

## Trace 调用树结构

至此，我们已不再满足于简单的事件列表日志，而是期望能够直观呈现一次 Agent 执行过程的调用树。

Agent 的执行并非线性的，它更类似于树形结构。一次模型调用往往会触发一个或多个工具调用，工具调用的结果又会作为输入进入下一轮的模型调用。若某个工具触发了委派操作，还会展开一个子 Agent 的完整执行流程。

在设计原始日志保存机制时，我们记录了几个关键字段：model\_call\_id、tool\_call\_id 和 delegation\_id。其中，model\_call\_id 用于将一次模型请求、响应、决策记录与后续动作关联起来；tool\_call\_id 用于将工具调用与其结果进行配对；delegation\_id 则用于将子 Agent 的事件挂载回父 Agent 的委派节点。有了这些字段，Trace 调用树的构建就无需依赖时间顺序进行推测。

例如，我们之前遇到的写文件失败问题，若从 Trace 调用树的角度审视，会清晰许多。你可以展开某一次的 model call，查看其选择了 write\_file 工具；进一步展开 tool call，能够看到参数中出现了 \_raw；再查看 tool result，会发现工具返回字段缺失或参数解析失败。接着查看下一轮的 model call，检查它是否将上一轮的失败情况纳入了上下文。随后你会发现，它又生成了一次几乎相同的 \_raw。

这个时候，我们几可以很清楚的看到 模型在重复错误参数结构。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/6Uzn2S5AAySmhxOCP2QVYSicdN2NqUbV5IiahR3oetWxa15Vr5ibgwc5cIdySrdgCv7eLvejhFNVFvqSpv8nP5w6AmUuJBopC7dzSaicrvrapBE/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1#imgIndex=9)

如果没有 Trace，我们还需要在众多的日志记录里面去翻看，查找对应的数据。有了Trace 我们可以很清楚看到 调用链路是怎么样的，排查问题的效率会有很大的提升。

![图片](https://mmecoa.qpic.cn/sz_mmecoa_jpg/oTVZYCicttuupdZGRvoG4ZEaEmzo9Z54QbHaQWRLohO8WHmfAFLqmUCnialic2ukMVy5QqjiaI9UKNf0ib8SCCibbWyIOQnVe4gHuVic2E9s1ABmLQ/640?wx_fmt=jpeg&from=appmsg#imgIndex=8)

子 Agent 也是同理。没有 `delegation_id` ，子 Agent 的事件会像一段插进来的噪声，而且这个是并行执行，只看时间线就会很乱了。你知道它做了什么事情，但很难知道它属于父任务的哪一次委派。挂成树以后，父子关系才稳定。

## 决策归因，为什么要这么做

通过Trace我们可以看到Agent的执行树，可以看到它不停的选择工具执行，这个时候我们自然就想知道模型选择这个工具的原因，它下次执行还会不会选择这个工具，还有没有其他的选择呢，特别是Agent如果把任务跑偏了要如何排查它在哪一步跑偏，它为什么会这样选择。

我们知道 Agent 做了什么，但不知道它为什么这么做。

对传统程序来说，行为来自代码。对 Agent 来说，很多行为来自模型当时的判断。

为了把 *为什么做* 记录下来，我们的做法是在 system prompt 里加入一个决策记录规范，让模型在需要选择动作时，在 reasoning 中输出一个固定格式的决策块。里面包括当前目标、候选动作、最终选择、选择原因和预期结果。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/6Uzn2S5AAySmmj9ud1MFicl1wJkCiaOgLOkTIPwYpdzm7v5Kax5GRSNctfdw09n93cI0ZoNm5tkep5ibibutOmX1XI3c6oncejI9TT9yPiama62c/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1#imgIndex=10)

后端拿到模型响应后，会解析这个决策块，生成 `decision` 事件，再挂到对应的模型调用节点上。

这样看 Trace 时，就不只是看到它调用了 `write_file` ，还可以看到它当时认为自己的目标是什么，为什么选择写文件，而不是继续搜索或者询问用户。

我觉得这个设计是有价值的，它可以显示的告诉我们模型做出选择的原因。

我们把这个决策放到了reasoning 里面。reasoning 模型通常更容易输出这类结构，普通 chat 模型，系统只能退回启发式：从实际 tool\_call 里拿到选择动作，再从 reasoning 或文本里截取一段理由。

当然我们不能完全相信这个决策，但对调试 Agent 来说，这已经比纯日志强很多。事实上在输出这个决策依据的时候，等同于大模型在自己反思，决策的正确率也会变高。

![图片](https://mmbiz.qpic.cn/mmbiz_png/6Uzn2S5AAyQsdn1iaLVAicNSe8oib7Kjn9iazuR5gBoZb9a97PcocfzhVMHQMic3aicUUkciavRKG6sogJ0QjFPmW2qLqNV25l7HyZOKicgnI0o5YrY/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1#imgIndex=11)

## 把任务状态显式化

对于简单一点的任务通过Trace面板就能看出问题所在，但是复杂任务光有 Trace 还不够。

比如我让Agent去搜索热点新闻，主Agent理解需求后，可能委派几个子Agent来完成任务，这时候用户真正关心的是任务状态：任务有没有完成？哪个卡住了？当前进行到了哪一步？

我们在系统中单独设计了任务状态。

每个会话开始时会创建一个 root task。调用 `delegate_task` 时，会创建子 task。任务有自己的状态机，从 pending、planning、running，到 waiting\_child、succeeded、failed、cancelled。

```
PS  delegate_task 这个是我们Agent自带的一个工具，用来创建子Agent
```

这让 Agent 的过程从一串对话事件，变成了一个可以查看状态的任务系统。

同时项目里多了2个基础日志记录 `tasks.jsonl` 和 `tasks.json` 。前者记录任务变化历史信息，后者保存当前任务状态。

在主流程里，会话开始时创建 root task，并在会话结束时把它转为 succeeded。创建子Agent的时候同时创建子 task，并根据子 Agent 的执行结果转为 succeeded 或 failed。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/6Uzn2S5AAyS2H7srSWibU7ooWwe4Pdjgm3NskWdI5OYHk1ZMk25w51pZBhoQlkg2TOG41WZs5PX74Me1IFyViafWmSDAOniaM5peMGakFY3DVE/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1#imgIndex=12)

## 异常检测

Agent 一次工具调用失败很正常。工具 schema 没写清楚，模型第一次试错，或者环境里缺文件，都可能发生。真正危险的是它连续失败，不能收敛，还在不停的执行。

它可能同一个工具连续失败。也可能连续两轮模型都没有响应。也可能 token 突然暴涨。还有一种很常见的情况：它一直说自己在调整，但实际上只是换一种说法重复失败。

为了判断Agent有没有在正常执行任务，我们需要一个执行轨迹异常检测。

我们在项目里先实现了一组规则，比如重复失败、接近迭代上限、空响应循环、压缩频繁、未知工具。

这些规则会在执行过程中或者会话结束后触发。一旦命中，就写入 `anomaly` 事件。前端可以在可观察性页面展示，也可以在日志和 Trace 里定位。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/6Uzn2S5AAyQ9lszjRX1BkAkNHoI2V5gaVbntEtSGCtNYGJSZtouEwZ7k0oLexQcwicLTo7Im8VDsWfWsDaaeVqPjia4taLNuAuZibCyiank0UyM/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1#imgIndex=13)

## 评估

前文所述，更多聚焦于审视任务执行的过程。当我们对过程有了清晰的认知后，还需回答一个更为基础的问题：此次任务究竟是否成功完成？以撰写文章这一任务为例，文件是否成功写入，内容是否符合要求，若过程中出现诸多异常情况，最终结果是否仍可接受？

为此，我构建了一套评估体系，并设置了三种评估方式：  
第一类为用户反馈。用户能够对模型的回复进行点赞或点踩操作。  
第二类是启发式评估。会话结束后，系统会检查是否存在明显的失败信号，例如未给出最终回复、出现高严重度异常、模型调用失败、迭代次数接近上限以及工具错误率过高等情况。  
第三类是“LLM - as - judge”，即借助另一个模型对 Agent 的输出进行评估。

有了评估体系，优化工作便不再仅凭直觉。否则，当我们修改了提示词（prompt）后，只能感觉流程似乎更顺畅了，但却无法知晓错误率是否降低、循环次数是否减少以及最终成功率是否提高。

![图片](https://mmbiz.qpic.cn/mmbiz_png/6Uzn2S5AAySCUqTfECD5icp64nYSEXKEwNZKIDHMQvXDkJjFry4NC0HiatriavO3A2YicVWZe8aQHOgzmJAfIndTtJ6INNbVX6UZWwbloV4dNZw/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1#imgIndex=14)

## 回放和对比

在实际调试 Agent 时，最为常见的操作流程是：捕捉到一个失败案例后，对提示词、工具描述或 Agent 配置进行修改，然后再次运行。然而，问题在于，如何判断这些修改是否有效呢？严格意义上的复现几乎是不现实的，因为模型具有随机性，工具的运行结果也可能存在差异，外部环境同样处于不断变化之中。我们只能使用同一个案例，在相似的条件下再次运行。

具体操作方法十分简单：利用原有的用户消息，结合新的提示词、工具描述或 Agent 配置，创建一个新的会话并重新运行。运行结束后，对比两个 Trace 调用树。例如，原会话中写文件失败了 4 次，而新会话仅失败 1 次甚至没有失败；原会话触发了高风险异常告警，新会话却未触发；原会话的评估结果为失败，新会话则变为成功。通过这样的对比，我们能够更清晰地判断修改提示词或工具描述是否改善了执行轨迹。

两个会话的迭代过程、模型调用、工具调用以及委派节点会进行结构化对齐。哪些节点保持一致，哪些是新增的，哪些已经消失，哪些节点的状态或耗时发生了变化，都能够清晰地展示出来。这使得优化工作形成了一个闭环：从发现问题、定位执行轨迹，到修改配置，最后进行回放对比。这里的回放类似于使用同一个案例在相似条件下进行再次验证，虽然存在一定的可控偏差，但足以帮助我们确定优化的方向。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/6Uzn2S5AAySPibu5S5Fq53hYTeVL5pNxRibswTXoVu5AP9MNwFCzkb4ib3wroEJaOVwRh4v5pR0bK6GUOCGBhIIiakrG5LJDgv2CpNAxtdBrJYo/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1#imgIndex=15)

## 总结

这就是我在开发 Mini-Openclaw 这个Agent时候，对Agent可观察性的一些思考，目前开发的功能看起来都还比较粗糙，随着后续开发还会逐步的完善，目前实现的思路就是这些

- 日志记录
- 指标设计
- 链路追踪
- 异常告警
- 决策归因
- 任务状态
- 结果评估
- 回放对比

**\- END -**

![图片](https://mmbiz.qpic.cn/sz_mmbiz_jpg/jB3fIA4CibvjXmVDlmFnWz8qlTxavcia2nyCEdbqrInicTsxx4kj2bicmFQyO01s7MYscU1AibIu1MtOr1RMpbhknMw/640?wx_fmt=other&from=appmsg&wxfrom=10005&wx_lazy=1&wx_co=1&randomid=u3j7j50l&tp=webp#imgIndex=9) ![](https://wst.wxapp.tc.qq.com/161/20304/snscosdownload/SH/reserved/686e29d4000b04cc120c3bc9dcc11715000000a100004f50?imageView2/1/w/800/h/800/q/50)

【新】Photoshop入门教程 案例视频版 自学PS视频教程零基础入门书籍 抠图调色海报修图电商美工主图详情页制作

运费险

先用后付

7天无理由

立减5元

回购9.5折

回购满50减5

已售301

¥31.8

![](https://res.wx.qq.com/shop/public/2024-10-17/fae7be51-beb6-4e61-aad8-4c1f7ccfab83.png)

智博尚书图书

![](https://res.wx.qq.com/shop/public/2025-06-25/61fc5241-1492-4f11-a5dc-421449d72741.png)

![](https://res.wx.qq.com/t/components/icons/base/arrow_down_regular.svg) 收起

使用手机微信

扫码了解商品信息

![](https://wst.wxapp.tc.qq.com/161/20304/snscosdownload/SH/reserved/6972d9330005227229c04cb303ab1715000000a100004f50?imageView2/1/w/800/h/800/q/50)

【excel新书】高效数据分析：Excel函数和动态图表实战（案例视频精华版）

运费险

先用后付

7天无理由

立减5元

回购9.5折

回购满50减5

已售1

¥26.5

![](https://res.wx.qq.com/shop/public/2024-10-17/fae7be51-beb6-4e61-aad8-4c1f7ccfab83.png)

智博尚书图书

![](https://res.wx.qq.com/shop/public/2025-06-25/61fc5241-1492-4f11-a5dc-421449d72741.png)

![](https://res.wx.qq.com/t/components/icons/base/arrow_down_regular.svg) 收起

使用手机微信

扫码了解商品信息

![](https://wst.wxapp.tc.qq.com/161/20304/snscosdownload/SH/reserved/68390d500001d7e7105e4c4223468e0b000000a100004f50?imageView2/1/w/800/h/800/q/50)

Excel VBA快速入门数据处理实战技巧精粹

运费险

先用后付

7天无理由

立减5元

回购9.5折

回购满50减5

已售512

¥31.8

![](https://res.wx.qq.com/shop/public/2024-10-17/fae7be51-beb6-4e61-aad8-4c1f7ccfab83.png)

智博尚书图书

![](https://res.wx.qq.com/shop/public/2025-06-25/61fc5241-1492-4f11-a5dc-421449d72741.png)

![](https://res.wx.qq.com/t/components/icons/base/arrow_down_regular.svg) 收起

使用手机微信

扫码了解商品信息

![](https://wst.wxapp.tc.qq.com/161/20304/snscosdownload/SH/reserved/67c80e92000f28732c398a3687ef1715000000a100004f50?imageView2/1/w/800/h/800/q/50)

【配套教学视频】剪映+AI短视频剪辑从入门到精通 零基础学视频剪辑（手机版+电脑版+网页版）

运费险

先用后付

7天无理由

立减5元

回购9.5折

回购满50减5

已售64

¥30.8

![](https://res.wx.qq.com/shop/public/2024-10-17/fae7be51-beb6-4e61-aad8-4c1f7ccfab83.png)

智博尚书图书

![](https://res.wx.qq.com/shop/public/2025-06-25/61fc5241-1492-4f11-a5dc-421449d72741.png)

![](https://res.wx.qq.com/t/components/icons/base/arrow_down_regular.svg) 收起

使用手机微信

扫码了解商品信息

![](https://wst.wxapp.tc.qq.com/161/20304/snscosdownload/SZ/reserved/681b1269000162dd0b61b392befe0a15000000a000004f50?imageView2/1/w/800/h/800/q/50)

【excel新书】效率是这样练成的：Excel高效数据处理与分析（案例视频精华版）

运费险

先用后付

7天无理由

立减5元

回购9.5折

回购满50减5

已售1

¥30.9

![](https://res.wx.qq.com/shop/public/2024-10-17/fae7be51-beb6-4e61-aad8-4c1f7ccfab83.png)

智博尚书图书

![](https://res.wx.qq.com/shop/public/2025-06-25/61fc5241-1492-4f11-a5dc-421449d72741.png)

![](https://res.wx.qq.com/t/components/icons/base/arrow_down_regular.svg) 收起

使用手机微信

扫码了解商品信息

![](https://wst.wxapp.tc.qq.com/161/20304/snscosdownload/SH/reserved/676388f8000b14bb10e73e140661b01e000000a000004f50?imageView2/1/w/800/h/800/q/50)

【电工必备系列】PLC梯形图语句表编程控制应用速查手册（图解·视频·案例）电工电路+电子元器件+plc+万用表

运费险

先用后付

7天无理由

收藏店铺立减5元

回购9.5折

回购满50减5

已售3621

¥47.31

起

回购专享价

![](https://res.wx.qq.com/shop/public/2024-10-17/fae7be51-beb6-4e61-aad8-4c1f7ccfab83.png)

智博尚书图书

![](https://res.wx.qq.com/shop/public/2025-06-25/61fc5241-1492-4f11-a5dc-421449d72741.png)

![](https://res.wx.qq.com/t/components/icons/base/arrow_down_regular.svg) 收起

使用手机微信

扫码了解商品信息

![](https://wst.wxapp.tc.qq.com/161/20304/snscosdownload/SH/reserved/6810704200072f592971e7d49a458e0b000000a100004f50?imageView2/1/w/800/h/800/q/50)

【excel新书】从超繁到极简：Excel高效办公实用技能技巧（734分钟案例视频精华版）

运费险

先用后付

7天无理由

收藏店铺立减5元

回购9.5折

回购满50减5

已售3643

¥43.51

起

回购专享价

![](https://res.wx.qq.com/shop/public/2024-10-17/fae7be51-beb6-4e61-aad8-4c1f7ccfab83.png)

智博尚书图书

![](https://res.wx.qq.com/shop/public/2025-06-25/61fc5241-1492-4f11-a5dc-421449d72741.png)

![](https://res.wx.qq.com/t/components/icons/base/arrow_down_regular.svg) 收起

使用手机微信

扫码了解商品信息

![](https://wst.wxapp.tc.qq.com/161/20304/snscosdownload/SZ/reserved/6a0fb2c60001d9872b2dfcf65c087615000000a000004f50?imageView2/1/w/800/h/800/q/50)

【2026全新正版】常用电子元器件+电工电路实物+plc接线全集（全彩图解+视频教程）从零基础到实战入门书籍电工必备书籍

运费险

先用后付

7天无理由

回购满50减5

回购9.5折

已售2299

¥51.8

起

回购专享价

![](https://res.wx.qq.com/shop/public/2024-10-17/fae7be51-beb6-4e61-aad8-4c1f7ccfab83.png)

智博尚书图书

![](https://res.wx.qq.com/shop/public/2025-06-25/61fc5241-1492-4f11-a5dc-421449d72741.png)

![](https://res.wx.qq.com/t/components/icons/base/arrow_down_regular.svg) 收起

使用手机微信

扫码了解商品信息

![](https://wst.wxapp.tc.qq.com/161/20304/snscosdownload/SZ/reserved/69c39859000657cf2e73ae9769861c15000000a000004f50?imageView2/1/w/800/h/800/q/50)

【AI人工智能系列】DeepSeek使用方法与应用实践100例 AI人工智能办公提效市场营销跨境电商自媒体运营副业变现

运费险

先用后付

7天无理由

立减5元

回购9.5折

回购满50减5

已售9

¥31.8

![](https://res.wx.qq.com/shop/public/2024-10-17/fae7be51-beb6-4e61-aad8-4c1f7ccfab83.png)

智博尚书图书

![](https://res.wx.qq.com/shop/public/2025-06-25/61fc5241-1492-4f11-a5dc-421449d72741.png)

![](https://res.wx.qq.com/t/components/icons/base/arrow_down_regular.svg) 收起

使用手机微信

扫码了解商品信息

![](https://wst.wxapp.tc.qq.com/161/20304/snscosdownload/SH/reserved/681034040009f21618bb079af734b00b000000a100004f50?imageView2/1/w/800/h/800/q/50)

秒懂DeepSeek：让AI成为你的贴心助手 AI时代专业级的DeepSeek通识书

运费险

先用后付

7天无理由

立减5元

回购9.5折

回购满50减5

已售8

¥30.8

![](https://res.wx.qq.com/shop/public/2024-10-17/fae7be51-beb6-4e61-aad8-4c1f7ccfab83.png)

智博尚书图书

![](https://res.wx.qq.com/shop/public/2025-06-25/61fc5241-1492-4f11-a5dc-421449d72741.png)

![](https://res.wx.qq.com/t/components/icons/base/arrow_down_regular.svg) 收起

使用手机微信

扫码了解商品信息
