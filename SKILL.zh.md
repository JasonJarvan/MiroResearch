---
name: MiroResearch
description: 当问题需要深入、多步骤的网页调研并附引用时使用——对比分析、文献/市场/技术综述、「X 的最新进展是什么」、需跨多个来源查证的事实调查——且单次快速回答不够用时。封装 MiroMind Deep Research API（mirothinker），可自主搜索网页、抓取页面并在服务端运行代码，最后返回综合报告。跨 Agent 通用（Claude Code、Codex 等）；需要 MIROMIND_API_KEY。
---

# MiroResearch

通过 MiroMind Responses API 执行深度调研。`mirothinker` 模型是自主调研 Agent：给定一条提示后，它会规划、搜索网页、抓取页面，并可选择运行代码——全部在**服务端**完成——最后返回带引用的综合报告。你无需提供工具，也无需在本地执行任何操作。

**核心原则：** 一次调用委托整个调研任务。封装脚本提交后台任务并在*内部*轮询，因此你只需一次工具调用即可拿到一份报告——无需由模型驱动的轮询循环，token 消耗最小。

## 何时使用

- 多来源问题：「对比 X 与 Y」「综述 Z 的现状」「……的最新进展是什么」
- 需要最新网页数据与引用的查证
- 单次 LLM 快速回答不够用、否则需要手动搜索→阅读→综合的场景

**何时不要使用：**
- 可直接回答或一次网页搜索即可解决的简单查询 → 直接回答即可
- 需要本地仓库/文件上下文、而 API 无法访问的任务
- 未配置 `MIROMIND_API_KEY` 时

## 快速参考

在本技能目录下运行（脚本从环境变量读取 `MIROMIND_API_KEY`）：

| 目标 | 命令 |
|------|------|
| 调研、等待、获取报告（默认） | `python3 scripts/miro_research.py run "QUERY"` |
| 使用旗舰模型（更深、更慢） | `python3 scripts/miro_research.py run "QUERY" --full` |
| 提交后不管，获取任务 id | `python3 scripts/miro_research.py submit "QUERY"` |
| 查看状态 | `python3 scripts/miro_research.py status RESP_ID` |
| 获取已完成报告 | `python3 scripts/miro_research.py result RESP_ID` |
| 取消 | `python3 scripts/miro_research.py cancel RESP_ID` |
| 列出近期任务 | `python3 scripts/miro_research.py list` |

模型：`mirothinker-1-7-deepresearch-mini`（默认，快/省）、
`mirothinker-1-7-deepresearch`（`--full`，更深）。两者均为 256k 上下文。

## 使用方法（Agent 工作流）

1. **默认路径 — `run`：** 绝大多数情况调用 `run "QUERY"`。它会阻塞直到报告就绪（深度调研通常需数分钟），并将报告打印到 stdout。进度/状态输出到 stderr；用量统计在结束时打印。
2. **长任务 / 并行 — `submit` + `status` + `result`：** 若不想长时间占用一次调用，或希望多个调研并行进行，`submit` 会立即返回 `resp_…` id；轮询 `status`；在 `completed` 后用 `result` 获取。若 `run` 可能超过其超时时间，请用此方式。
3. **任意读取命令加 `--json`**，当你需要原始结构化输出（output items、tool calls、usage）而非仅报告正文时。

用 `--timeout SECONDS`（默认 900）和 `--interval SECONDS`（默认 10）调整等待。超时后任务仍在服务端继续运行——稍后可用 `result RESP_ID` 重新获取。

## 配置

- 必须设置 `MIROMIND_API_KEY`（`sk_live_…`，在 MiroMind 控制台创建）。
- `MIROMIND_BASE_URL` 可选，默认 `https://api.miromind.ai/v1`。
- 唯一依赖为 `requests`（多数环境已自带）。

## 常见错误

- **由模型驱动轮询循环。** 普通任务不要自己 `submit` 后反复调 `status`——那会浪费 token。用 `run`，它在进程内部轮询。
- **把 tool call 当作需要本地执行的东西。** 输出中的 `tool_call` 项已在 MiroMind 服务端执行完毕；读 `item.result`，切勿重跑。
- **期望 OpenAI 函数调用 / `response_format`。** 不支持。这是自包含的调研 Agent；你只发送提示并阅读报告。
- **超时过短。** 深度调研是分钟级；提高 `--timeout`，或改用 `submit`/`result` 模式，而不是快速失败。

## 文件

- `scripts/miro_research.py` — CLI 封装（run/submit/status/result/cancel/list）。
