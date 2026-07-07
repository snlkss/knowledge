---
title: "Agent 工程终于有脚手架了， Google开源一个开发agent的工具"
source: "https://mp.weixin.qq.com/s/rZzul1J3pvKbzxDnfGs_1A"
author:
  - "[[硅基铁匠]]"
published:
created: 2026-07-07
description: "Google 的 agents-cli 把脚手架、评估、部署、企业注册串起来，给 Agent 开发补上了工程化那一段。"
tags:
  - "clippings"
---
硅基铁匠 硅基铁匠 *2026年7月1日 11:32*

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/p7phJnGKXGiaScnYQdyLytdjpquPShT8q62qH337KBMlKZv5GpBh2uCvG4oYAw9ibmBA4bBzVQmURzg3uEibeN1BON71iaMnZuxsfJT6ZEC28a8/640?from=appmsg&watermark=1&tp=webp&wxfrom=5&wx_lazy=1#imgIndex=0) Karpathy 前段时间把一个词讲热了：Agentic Engineering 。它听起来很抽象，落到项目里，其实就是三件事：先把需求写清楚，再用评估反复压问题，最后把安全、权限、部署、观测这些工程活补齐。

过去做 Agent，最容易卡在中间。写代码在编辑器里，起项目在终端里，测试要开浏览器，部署要进云控制台，评估还要再接一套框架。每一步都能做，但每换一个工具，脑子里的上下文就丢一截。

Google 这个 `agents-cli` 解决的正是这段断裂。它并不提供一个新的聊天机器人，也不替代 Claude Code 、 Codex 、 Cursor 。它更像一套给 coding agent 装上的工程技能包，让这些 coding agent 知道怎么用 Google 的 ADK 、 Agent Runtime 、 Cloud Run 、 Gemini Enterprise 去搭、测、发一个企业级 Agent 。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/p7phJnGKXGiap60NxvvzGtHbrOCfiabdUpmZfoIdX2UOgMicKGX0WX9ibDLAqGAmHdSVjpc7Z5xQbrVYt9oVw9kgRlrr2GhbMI9mWX24xedib1IE/640?from=appmsg&watermark=1&tp=webp&wxfrom=5&wx_lazy=1#imgIndex=1)

GitHub 上的定位很直白： `agents-cli` 是一套 CLI 和 skills，用来把常见 coding assistant 变成更懂 Google Cloud Agent Platform 的开发助手。

它支持 Antigravity CLI 、 Claude Code 、 Codex，也可以配合其他 coding agent 。安装后，它会给 coding agent 注入 7 类技能：

- • `google-agents-cli-workflow` ：Agent 开发生命周期和代码保留规则。
- • `google-agents-cli-adk-code` ：ADK Python API 、 tools 、 callbacks 、 state 等写法。
- • `google-agents-cli-scaffold` ：创建、增强、升级 Agent 项目。
- • `google-agents-cli-eval` ：评估集、指标、 LLM-as-judge 、 rubric 。
- • `google-agents-cli-deploy` ：Agent Runtime 、 Cloud Run 、 GKE 、 CI/CD 、 secrets 。
- • `google-agents-cli-publish` ：注册到 Gemini Enterprise 。
- • `google-agents-cli-observability` ：Cloud Trace 、日志和观测接入。

这个设计的重点不止是“又多一个命令行工具”。它想把 Agent 项目从 demo 拉到可交付状态。能创建项目只是第一步，能测试、能部署、能被组织里的人找到，才算走完。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/p7phJnGKXGhHz0vibzd3frEkWOyDnEvLFOiaW6DZO7nkiaGpmRI2JlRXvICT10DXAyIJ6hS9GveWicW5e8UsiabNaze6vUiacQ3TZUjrMN55LVGTM/640?from=appmsg&watermark=1#imgIndex=2)

## 第一步：安装

准备好 Python 3.11+、 `uv` 和 Node.js 后，直接跑：

```
uvx google-agents-cli setup
```

如果只想装 skills，让自己的 coding agent 接管后面的工作，也可以用：

```
npx skills add google/agents-cli
```

装完后打开你常用的 coding agent，比如 Claude Code 、 Codex 、 Cursor 或 Antigravity CLI，让它按自然语言指令去创建项目。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/p7phJnGKXGjSvk8z21hdyOQr5hFJsPNLXaWbhGVgKnibFsK6ORC8FSDRhQZo5g0pWI6jbFZpxgAqvvEP5SAnrmkEruOku8wiakmMLt6cBL4ow/640?from=appmsg&watermark=1#imgIndex=3)

## 第二步：让 coding agent 搭一个 RAG Agent

一个可复现的起手式是：

```
Build a RAG agent that ingests documents, retrieves relevant context,
and answers questions with source citations. Use the ADK agentic_rag
template with Gemini 3.5 Flash.
```

在 Akshay 的测试里，Claude Code 调用了 `agents-cli` 的 ADK skills，从 `agentic_rag` 模板搭出项目，用 Vector Search 做 datastore，还补了 citation 相关逻辑：回答必须有引用，retriever 返回文档时带 source ID 。

这一步很关键。很多 RAG demo 只演示“能答”，企业里更关心回答有没有资料依据。引用链如果一开始没设计，后面再补会很麻烦。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/p7phJnGKXGgHIIVYKV7HsGTm8Y65bwICLDgNAgdrLic5PzYuXuyy60K8xMfJ5c5Bdgf8VpKw1cj3oYIWXQgTZwDIcI8TjXFXImrYcV4mAbT0/640?from=appmsg&watermark=1#imgIndex=4)

## 第三步：本地先测一轮

项目起来后，让 coding agent 启动 ADK Web UI：

```
Spin up a local dev server so I can test this.
```

本地测试至少看两类问题。

第一类是资料里能回答的问题，比如 “how to merge two dictionaries?”，Agent 应该能检索到对应内容，解释 `|` 合并和 `update()` 方法，并附上类似 `[source: 1003]` 的引用。

第二类是资料里没有的问题，比如 “who won the FIFA World Cup in 2022?”，Agent 应该承认资料不足，不能凭常识硬答。 RAG 项目上线前，这类拒答测试比“答得很顺”更有价值。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/p7phJnGKXGiaticJ5qhA6iaHOOl4WY7nCPG9ibChaucsYOLm6fZUzPF6ksTcPB3tRxx9cGiardZh85J9VibIU6ZNCvb6ibayYT0QJR3KVUU3UfNrOc/640?from=appmsg&watermark=1#imgIndex=5)

## 第四步：上线前做评估

很多 Agent 项目死在这里：demo 能跑，评估没有。 Karpathy 提过一个数据，运行 Agent 的团队里，做 observability 的比例高于做 evals 的比例。可没有 evals，日志再多也很难判断改动有没有把系统弄坏。

可以直接让 coding agent 生成评估集：

```
Generate 20 test scenarios for this RAG agent covering correct retrieval,
insufficient context where the agent should say it doesn't know,
multi-hop questions, and citation accuracy. Run the full eval suite and
show me the results.
```

这 20 个 case 可以分成四组：

- • 6 个正确检索问题；
- • 5 个资料不足时的拒答问题；
- • 5 个需要多文档推理的问题；
- • 4 个 citation accuracy 问题。

Akshay 的测试结果里，引用准确率 20/20，通过。但 eval 也抓到一个洞：当问题不在语料里时，Agent 有时会补一句通用知识。问题来自 instruction 里的一行宽松规则，大意是“简单问题可以不用工具直接回答”。删掉这行，拒答行为才会稳定。

这就是 eval 的价值。分数表只是表面结果，最有用的是提前暴露那些容易被忽略的指令漏洞。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_jpg/p7phJnGKXGgBYFErUjtPd0RlSVxB1ZPry1Y265S1eedKcU31dHKfcibqZHG2ibhPp0MPaU0Gxic2MnCBrCP3nszWQqwrOlkXxaKlHM7xh0AM0A/640?from=appmsg&watermark=1#imgIndex=6)

## 第五步：部署到 Agent Runtime

评估过后，就可以让 coding agent 处理部署：

```
Deploy this to Agent Runtime in us-central1.
```

`agents-cli` 会把项目补齐为 Agent Runtime 可部署的形态，加入入口文件和基础设施配置。根据这次测试，部署到 Google Cloud 大概花了 2 到 3 分钟。

Cloud Trace 默认接入，这一点对团队协作很实用。 Agent 出问题时，不能只看聊天记录，还要能回到 trace 、日志、调用链里定位是哪一步坏了。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/p7phJnGKXGgJeY8oJNz8TuhhSb2zib8YvwtQxUhYjqLcgXIjvEyngNXJhbdc8qMFbz0FoqSNyCKFEKXMWWBC5MjIx66G9ul73QPmzy2FWHnA/640?from=appmsg&watermark=1#imgIndex=8)

## 第六步：注册到 Gemini Enterprise

很多内部 Agent 做完后，只停留在“某个同事机器上能跑”。别人不知道它存在，也拿不到 endpoint 、权限和使用方式。这样的 Agent 很快就会被遗忘。

继续让 coding agent 执行：

```
Register this agent to Gemini Enterprise.
```

注册后，它会出现在 Gemini Enterprise app 里，组织内有权限的人可以发现和使用。 IAM 控制访问，企业面板负责观测。到这一步，一个 RAG Agent 才从个人 demo 变成团队可用的内部知识助手。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_jpg/p7phJnGKXGia2zfQI1qdmDNE6KU2cXOiaoiaErEcw06BKdtVSIl1IMP4wg55VpaqdKTzZibviaHdZtzD9UHPVgariaFesPLDG5dibkG6LOiaD7LaRkY/640?from=appmsg&watermark=1#imgIndex=9)

## 可以怎么用在自己的项目里

如果只是想试水，不用一上来就做复杂 Agent 。更稳的路径是：

1. 1\. 先选一个低风险知识库，比如团队 FAQ 、产品术语表、内部 onboarding 文档。
2. 2\. 用 `agents-cli scaffold` 或 setup 后的 coding agent 建一个 RAG 项目。
3. 3\. 写 15 到 30 个真实问题，里面故意混入资料不足、歧义、多跳问题。
4. 4\. 先跑 eval，再改 instruction，不要只靠手感调 prompt 。
5. 5\. 本地测过后再部署，部署后补 trace 、权限、成本监控。
6. 6\. 最后再考虑注册到企业入口，让团队成员能找到它。

GitHub README 里列出的常用命令也值得保存：

```
agents-cli scaffold <name>
agents-cli eval generate
agents-cli eval grade
agents-cli deploy
agents-cli publish gemini-enterprise
```

如果你已经有一个 ADK 项目，也可以用：

```
agents-cli scaffold enhance
```

它会给旧项目补部署、 CI/CD 或 RAG 相关能力。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_jpg/p7phJnGKXGjm1abqxlAPQlk9aVtEUcVV6oerbOCNoOtvNsjdnu5f9VP76x6vR2G6APEX1uvReyFfC6SP4iaSYUHibbDdedhT2aOQVKBrGPssg/640?from=appmsg&watermark=1#imgIndex=11)

## 使用前要先想清楚的地方

`agents-cli` 很适合 Google Cloud 和 ADK 体系内的 Agent 工程。如果你的团队已经在用 Vertex AI 、 Cloud Run 、 Gemini Enterprise，它能省掉不少胶水工作。

但它也带来一个前提：部署、观测、企业注册这些能力都和 Google Cloud 绑定得比较深。个人开发者可以本地玩起来，真要走云端和企业入口，还是要处理账号、计费、权限、服务条款和区域合规。

另一个提醒是，不要把“coding agent 能自动跑完整流程”理解成可以少做验收。脚手架能加速，eval 和权限检查不能省。 Agent 最危险的地方往往不在答不上来，而在资料不足时答得太顺。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_jpg/p7phJnGKXGj4MAkWvUcHZBHxzaV2G8BwLCDoymFNzU6Wa23BhCGC7yqz4obcvibZr2w5tG7Ln3z1bzgscCUx2Et6fqYMIqEqIRPvI6JlAmZY/640?from=appmsg&watermark=1#imgIndex=12)

## 我会怎么判断它值不值得用

如果你的 Agent 还停留在玩具 demo 阶段， `agents-cli` 可能显得有点重。直接用 ADK 或 LangGraph 写一个本地原型，反而更快。

如果你已经遇到这些问题，它就值得试：

- • 每个 Agent 都要重新搭项目结构；
- • 评估集总是上线前才想起来；
- • 部署脚本、权限、 Cloud Run 配置反复复制；
- • 内部 Agent 做完后没人知道入口在哪里；
- • 团队希望 coding agent 不只写代码，还能按工程规范把项目推到可上线状态。

Agent 开发接下来拼的不会只是模型调用能力。更麻烦的部分在评估、权限、部署、观测、组织分发。 Google 这次把这些环节塞进一个 CLI 和一组 skills 里，方向是对的：让 coding agent 少当“会写代码的助手”，多承担一点工程交付的脏活。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/p7phJnGKXGiaMYMeCnNSMNrgNbNM27ApcibDTq59QZbVDDBx86zkowFJ9NYKCscjFOsdmHhYBre77byWGZP9k58tejlvlCYqSyY6hsXUIx3WM/640?from=appmsg&watermark=1#imgIndex=13)

> 来源链接: https://github.com/google/agents-cli

**微信扫一扫赞赏作者**

教程 · 目录