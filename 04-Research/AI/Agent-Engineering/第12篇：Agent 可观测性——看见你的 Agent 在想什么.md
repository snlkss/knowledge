---
title: "第12篇：Agent 可观测性——看见你的 Agent 在想什么"
source: "https://mp.weixin.qq.com/s/fWbJWGevZfqHRcUk9q2XLw"
author:
  - "[[AI追梦少年]]"
published:
created: 2026-07-07
description: "传统服务挂了，看日志、看指标就能找到原因。Agent 失败了，你看到的是一个错误的最终答案——中间 20 步推理发生了什么，你完全不知道。可观测性三支柱（Traces/Metrics/Logs）在 Agent 时代需要重新定义。"
tags:
  - "AI Agent"
  - "Agent工程"
  - "可观测性"
---
AI追梦少年 FutureCraft AI *2026年6月2日 08:30*

---

## 第12篇：Agent 可观测性——看见你的 Agent 在想什么

上一篇讲了多 Agent 编排——怎么让多个 Agent 协作完成复杂任务。

这篇讲一个随之而来的问题：当你的 Agent（或多个 Agent）在生产中跑起来，出了问题，你怎么知道哪里错了？

---

## 为什么 Agent 可观测性比传统可观测性更难

传统服务的可观测性模型很成熟：函数调用有确定的输入输出，异常有 stack trace，性能有 latency 分布，错误有 HTTP status code。出了问题，日志 + 指标基本能定位。

Agent 的问题在于： **执行路径是非确定性的** 。

同样的输入，Agent 可能走完全不同的推理路径，调用不同的工具，产生不同的中间结果，最终到达（或没到达）正确答案。传统的「输入→处理→输出」三段式，在 Agent 这里变成了「输入→推理循环 N 轮→工具调用 M 次→可能的答案」。

**具体难点：**

**推理不透明** ：模型内部的推理过程不是显式的函数调用，是 token 序列。你看到的是最终文本，不是推理步骤。

**工具调用链** ：Agent 调用工具 A，结果作为输入调用工具 B，B 的结果触发调用工具 C——这个链路可能有 10-20 步，每一步都可能出错，错误还可能是级联的（A 的轻微偏差导致 C 的完全失败）。

**概率性失败** ：同一任务重跑可能成功。这让传统的「复现 bug → 修复 → 验证」工作流完全失效。你不能稳定复现，就很难确认修复是否有效。

**多 Agent 时序** ：上一篇的多 Agent 场景里，Agent A 和 Agent B 并行执行，共享状态，时序依赖——这类问题在传统分布式系统里已经够难了，加上 LLM 的非确定性，难度再上一层。

---

## 重新定义可观测性三支柱

传统可观测性的三支柱（Traces、Metrics、Logs）在 Agent 时代需要重新定义其含义。

### Traces：推理-工具调用链路追踪

传统 Trace 追踪的是函数调用栈——A 调 B 调 C，每层的耗时和参数。

Agent Trace 追踪的是 **推理-行动序列** ：

```
Agent 收到任务 (t=0)
  ├── LLM 推理 (t=0~2.3s)  ← 模型在想什么
  │     思考："需要先了解代码库结构，用 glob 搜索"
  ├── Tool Call: Glob("**/*.py") (t=2.3~2.8s)
  │     输入: pattern="**/*.py"
  │     输出: [128 个文件路径]
  ├── LLM 推理 (t=2.8~5.1s)
  │     思考："文件太多，聚焦 src/ 目录"
  ├── Tool Call: Read("src/auth/jwt.py") (t=5.1~5.4s)
  │     输入: file_path="src/auth/jwt.py"
  │     输出: [文件内容 847 tokens]
  ├── LLM 推理 (t=5.4~9.2s)
  │     思考："发现问题：JWT 密钥硬编码"
  └── 最终答案生成 (t=9.2~11.0s)
```

这个 Trace 让你看到：推理花了多少时间、工具调用了多少次、每次工具的输入输出是什么、在哪个步骤出错了。

### Metrics：超越延迟和错误率

传统 Metrics 关注延迟（p50/p95/p99）、错误率、吞吐量。

Agent Metrics 还需要关注：

- • **Token 使用** ：输入 token、输出 token、缓存命中率——这直接决定成本
- • **工具调用次数** ：平均完成任务需要调用多少次工具，趋势是否在上升
- • **任务完成率** ：Agent 成功完成任务的比例（不是「没报错」，是「真的完成了」）
- • **重试次数** ：Agent 需要重试多少次才能成功
- • **上下文使用率** ：任务结束时上下文窗口用了多少——持续接近上限是问题信号
- • **成本/任务** ：每完成一个任务花了多少钱，趋势如何

### Logs：模型输入输出完整记录

传统日志记录应用层事件。

Agent 日志需要记录：

- • **完整的模型输入** （系统提示 + 对话历史）——方便复现和调试
- • **完整的模型输出** （包括思考过程，如果模型支持）
- • **工具调用参数和返回值** ——每一次，完整记录
- • **Token 计数和成本** ——每次 LLM 调用
- • **错误详情** ——工具失败、模型拒绝、上下文溢出

---

## 主流可观测性平台对比

### LangSmith

LangChain 官方的可观测性平台。如果你用 LangChain/LangGraph 生态，LangSmith 的集成几乎是零成本：

```
import os
os.environ["LANGCHAIN_TRACING_V2"] = "true"
os.environ["LANGCHAIN_API_KEY"] = "..."

# 之后所有 LangChain/LangGraph 调用自动追踪
# 不需要修改任何业务代码
```

核心优势：

- • **近零接入成本** ：环境变量打开，自动追踪
- • **原生 LangChain 集成** ：追踪粒度精确到每个 Chain、每个 Tool、每个 LLM 调用
- • **数据集和评估** ：可以把实际运行的 trace 导出为评估数据集，直接用来跑评估

适合场景：LangChain/LangGraph 技术栈，或者需要快速上手可观测性的团队。

### Arize AI

企业级 AI 可观测性平台，覆盖从 LLM 到完整 Agent 工作流。

核心优势：

- • **Span 级追踪** ：精确追踪到每个操作的 Span，支持分布式 Agent 追踪
- • **Phoenix（开源部分）** ：Arize 的开源追踪工具，基于 OpenTelemetry，可以自托管
- • **漂移检测** ：监控模型行为随时间的漂移——相同输入，输出质量是否在下降
- • **企业特性** ：SSO、RBAC、数据留存、合规报告

适合场景：有合规要求的企业，或者需要长期监控模型行为漂移的团队。

### Langfuse

开源的 LLM 可观测性平台，可以完全自托管：

```
from langfuse import Langfuse
from langfuse.decorators import observe, langfuse_context

langfuse = Langfuse()

@observe()  # 装饰器自动追踪这个函数
def run_agent(task: str) -> str:
    # 你的 Agent 逻辑
    result = agent.run(task)
    
    # 手动记录评估结果
    langfuse_context.score_current_observation(
        name="task_completion",
        value=1 if task_completed else 0
    )
    return result
```

核心优势：

- • **自托管** ：数据留在自己的基础设施，适合数据敏感场景
- • **成本透明** ：开源，自托管几乎零平台费用
- • **灵活评估** ：内置 LLM-as-Judge 评估，可自定义评估维度

适合场景：数据隐私要求高（不能把数据发给第三方 SaaS）、或者需要控制可观测性平台成本的团队。

### Galileo

定位是「评估 + 护栏 + 可观测性」一体化，特别强调质量评估：

- • **内置评估指标** ：幻觉率、忠实度、相关性——不需要自己写评估逻辑
- • **护栏集成** ：发现问题可以直接触发护栏（拦截、告警、降级）
- • **实时监控** ：生产环境实时监控质量指标，不只是 trace 存档

适合场景：需要对 Agent 输出质量做持续监控，或者要把可观测性和护栏系统统一的团队。

### 平台选择速查

| 场景 | 推荐 |
| --- | --- |
| 用 LangChain/LangGraph，快速上手 | LangSmith |
| 数据不出境，自托管 | Langfuse |
| 企业合规、漂移检测 | Arize AI |
| 质量监控 + 护栏一体 | Galileo |

![图片](https://mmbiz.qpic.cn/sz_mmbiz_jpg/h3GXXT2J8FCl8hG2hqPzzdGjaHVSC0x1SX7gXkSIialwriaR0uLXGaRt3LOb0BfIy5ndGD8O5FNvxclUJPdPKZ1PhPTFibKJibB0jApDJlhMKOk/640?wx_fmt=jpeg&watermark=1&tp=webp&wxfrom=5&wx_lazy=1#imgIndex=0)

---

## OpenTelemetry for LLMs：新兴标准

传统可观测性领域，OpenTelemetry（OTel）已经成为事实标准——统一的 Trace/Metric/Log 数据格式，各平台都兼容。

LLM 可观测性正在走同样的路。 **OpenLLMetry** （Traceloop 主导）是目前最成熟的 OTel for LLMs 实现，定义了 LLM span 的标准属性：

```
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider

# 标准化的 LLM Span 属性
span.set_attribute("llm.vendor", "anthropic")
span.set_attribute("llm.request.model", "claude-opus-4-7")
span.set_attribute("llm.request.max_tokens", 4096)
span.set_attribute("llm.usage.prompt_tokens", 1247)
span.set_attribute("llm.usage.completion_tokens", 382)
span.set_attribute("llm.usage.total_tokens", 1629)
span.set_attribute("llm.response.model", "claude-opus-4-7")
```

标准化的意义：写一次 instrumentation，数据可以同时发给 LangSmith、Langfuse、Arize——不被任何一家平台绑定。

Anthropic 的官方 SDK 正在逐步支持 OTel 标准。如果你现在开始搭可观测性基础设施，建议用 OTel 作为底层，上层再选择展示和分析平台。

---

## 生产典型组合

单一平台往往不能满足所有需求。生产中常见的双层组合：

**网关层（成本追踪）+ 分析层（质量监控）**

**Helicone / Portkey** 作为 LLM 网关：所有 LLM 调用经过网关，自动记录 Token 用量和成本，支持按 Key/用户/任务类型做成本归因，还能做请求缓存（相同请求直接返回缓存结果，省钱）：

```
# Helicone 接入只需要替换 base_url
import anthropic

client = anthropic.Anthropic(
    base_url="https://anthropic.helicone.ai",
    default_headers={
        "Helicone-Auth": f"Bearer {HELICONE_API_KEY}",
        "Helicone-Property-Task": "code-review",  # 自定义属性，用于成本归因
    }
)
# 之后所有调用自动被 Helicone 记录
```

**Langfuse / LangSmith** 作为追踪和评估层：完整的 Trace 记录、质量评估、失败分析。

两层分工：网关层专注成本和流量，追踪层专注质量和调试，不互相耦合。

---

## 调试 Agent 失败的实战方法

有了可观测性基础设施，调试 Agent 失败就有了具体的工作流：

### 定位上下文腐烂

「上下文腐烂」是 Agent 长时间运行后常见的失败模式——随着对话历史积累，早期的关键信息被淹没，Agent 开始给出矛盾或错误的结果。

用 Trace 定位：找到任务开始走偏的那个 LLM 调用，查看那一刻的完整上下文。通常能看到上下文里充斥着大量已完成步骤的结果，原始任务描述占比很小，信噪比极低。

修复方向：增加 Compaction 频率，或者在任务超过一定步数时主动重注入原始任务描述（第 4 篇讲过这个模式）。

### 追踪工具调用链中的错误传播

Agent 的最终答案是错的，但不知道从哪一步开始偏的——用 Trace 逐步展开工具调用链：

```
Step 1: Glob("**/*.ts") → 返回了 247 个文件  ✓
Step 2: Read("src/config/database.ts") → 返回了文件内容  ✓
Step 3: LLM 推理 → "数据库连接字符串在 config.host"  ← 这里出错了
         实际上 config 对象结构不是这样的
Step 4: 基于错误假设继续推理 → 级联错误
```

找到 Step 3 的 LLM 推理，查看当时的输入——问题通常是上下文提供的信息不足以让模型做出正确推断，或者模型对不熟悉的代码库结构做了错误假设。

### 复现概率性失败

这是最难的部分。同一任务，跑 5 次有 2 次失败，你无法稳定复现。

实用方法：

1. 1\. 从失败的 Trace 里导出完整的模型输入（系统提示 + 对话历史 + 当前 prompt）
2. 2\. 用这个完整输入直接调用模型，看是否能复现失败
3. 3\. 如果能复现，在固定输入上做调试（调整系统提示、增加上下文、换模型）
4. 4\. 如果不能复现（真正的概率性），在相同输入上多次调用，统计失败率，评估是否需要修复

Langfuse 和 LangSmith 都支持从 Trace 直接导出「Playground 可用」的格式，简化这个过程。

---

## 实战：为 Agent 搭建可观测性基础设施

一个适合大多数项目的最小可观测性配置：

**Step 1：接入 Langfuse（5 分钟）**

```
pip install langfuse
```
```
import os
os.environ["LANGFUSE_PUBLIC_KEY"] = "pk-..."
os.environ["LANGFUSE_SECRET_KEY"] = "sk-..."
os.environ["LANGFUSE_HOST"] = "https://cloud.langfuse.com"  # 或自托管地址

from langfuse.decorators import observe

@observe()
def your_agent_function(task: str) -> str:
    # 你的 Agent 逻辑，不需要任何改动
    return result
```

**Step 2：添加关键 Metrics（10 分钟）**

```
from langfuse.decorators import observe, langfuse_context

@observe()
def run_agent_task(task: str) -> dict:
    start_time = time.time()
    
    try:
        result = agent.run(task)
        success = validate_result(result)
        
        # 记录任务级别的评估结果
        langfuse_context.score_current_trace(
            name="task_success",
            value=1 if success else 0,
            comment=f"耗时: {time.time() - start_time:.1f}s"
        )
        return {"success": success, "result": result}
    except Exception as e:
        langfuse_context.score_current_trace(
            name="task_success",
            value=0,
            comment=f"错误: {str(e)}"
        )
        raise
```

**Step 3：设置成本告警**

用 Helicone 或 Langfuse 的内置预算功能，设置每日/每周 Token 用量上限，超限发告警。防止 Agent 出 Bug 时默默烧掉大量 API 费用。

**Step 4：建立失败分析工作流**

每周 review 失败的 Trace：

- • 任务完成率低于 90%？找最常见的失败步骤
- • 某类任务的 Token 用量异常高？找上下文膨胀的位置
- • 错误集中在特定工具？检查工具的描述或实现

可观测性的价值不在于数据收集本身，在于建立「发现问题 → 定位 → 修复 → 验证」的闭环。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/h3GXXT2J8FAB0oibw2TJm1MiasHsbLkx4AFQ7nKS77fx7syAntcG9AlnG5C6Nibm8TxVrX3W2ez9ezXeNicsFj9v1g7TFfVQxs13WRaZIba1fR8/640?wx_fmt=jpeg&watermark=1&tp=webp&wxfrom=5&wx_lazy=1#imgIndex=1)

---

## 可观测性是 Harness 的眼睛

这个系列讲了很多 Harness 的主动控制手段——上下文管理、安全护栏、多 Agent 编排。可观测性是 Harness 的另一面：被动但同样关键。

主动控制决定 Agent 怎么运行，可观测性告诉你 Agent 实际在怎么运行——两者的差距，就是你需要持续改进的地方。

没有可观测性，你是在黑盒里调 Agent。有了可观测性，你才真正拥有这个系统。

---

下一篇： **成本优化实战** ——从 Peter Steinberger 的月花 $20,000 教训开始，讲清楚模型路由、Prompt 缓存、子 Agent 隔离，60-80% 成本节省的具体方案。

---

*本文是「Agent Harness Engineering 技术连载」第 12 篇。* *第 1 篇：什么是 Harness Engineering | 第 2 篇：六大核心组件 | 第 3 篇：Martin Fowler 分类法 | 第 4 篇：上下文工程 | 第 5 篇：工具设计 | 第 6 篇：长程自主执行 | 第 7 篇：安全护栏 | 第 8 篇：评估与基准测试 | 第 9 篇：解剖 Claude Code | 第 10 篇：解剖 OpenAI Codex | 第 11 篇：多 Agent 编排实战*

**微信扫一扫赞赏作者**

Agent Harness Engineering 技术详解 · 目录
