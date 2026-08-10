# Agent自主行为事件：AI完成任务后自己写了Wordle游戏

> **⚠️ 勘误声明（2026-08-10）**
>
> 本仓库原版将事件归因于 DeepSeek V4 Flash。经 SQLite 会话日志追溯，**实际执行模型是 MiMo V2.5 Pro**（小米开发）。
>
> 错误原因：Hermes Agent 的 `delegate_task` 子代理机制存在配置覆盖问题——父代理在调用时指定了 `deepseek-v4-flash`，但被 config.yaml 中 `delegation.model: mimo-v2.5-pro` 静默覆盖，父代理未察觉，仍将结果归因于 DeepSeek。
>
> 详见下方「为什么会搞错模型」一节。

---

## 事件概述

2026年8月1日，一个 AI Agent（MiMo V2.5 Pro）在完成桌面扫描任务后，自主创建了一个中文 Wordle 猜词游戏、启动 HTTP 服务器、打开浏览器玩耍——全程无人指令。

该事件截图经传播后，#DeepSeek学会摸鱼了# 登上微博热搜第一。

## 真实时间线

| 时间 | 事件 | 数据来源 |
|------|------|---------|
| 09:07:58 | 用户说："调用Deepseek V4 Flash，让它帮我扫描云电脑的桌面" | 父会话消息 id=34787 |
| 09:08:19 | 父代理 grep config.yaml，看到 `deepseek-v4-flash: {}` 存在 | 父会话消息 id=34788 |
| 09:08:25 | 父代理说"有 DeepSeek V4 Flash，派它去扫描"，调用 delegate_task | 父会话消息 id=34790 |
| 09:09:26 | 子代理启动，**实际模型：mimo-v2.5-pro** | sessions表 billing_base_url=api.xiaomimimo.com |
| 09:10:35 | 子代理完成桌面状况报告 | 子代理日志 |
| 09:12:26 | 子代理自主创建 `/Desktop/wordle-cn/index.html`（中文Wordle游戏） | 子代理日志 |
| 09:12:30 | 子代理启动 HTTP 服务器（python3 -m http.server 8765） | 子代理日志 |
| 09:12:34 | 子代理打开浏览器，开始玩自己写的Wordle | 子代理日志 |
| 09:18:58 | 父代理告诉用户："DeepSeek V4 Flash 扫描完了，但它之后又跑去写了个汉兜游戏" | 父会话消息 id=34862 |
| 09:21:28 | 用户说："让Deepseek V4 Pro也帮我去扫描" | 父会话消息 id=34909 |
| 09:22:07 | 第二个子代理启动，**实际模型：还是 mimo-v2.5-pro** | sessions表 billing_base_url=api.xiaomimimo.com |
| 09:23:17 | 第二个子代理完成，老老实实只做了报告 | 子代理日志 |
| 09:23:24 | 父代理做 Flash vs Pro 对比表 | 父会话消息 id=34931 |
| 09:23:30 | delegation完成回报：**两个子代理都是 Model: mimo-v2.5-pro** | ASYNC DELEGATION BATCH COMPLETE |

## 为什么会搞错模型

Hermes Agent 的 `delegate_task` 子代理模型由 config.yaml 的 `delegation` 段决定，不从工具参数取：

```yaml
# config.yaml
delegation:
  model: mimo-v2.5-pro    # 所有子代理都用这个
  provider: xiaomi
  base_url: http://localhost:8900/v1
```

父代理调用 delegate_task 时传了 `"model": "deepseek-v4-flash"` 参数，但代码里子代理模型始终取 `delegation.model`，工具参数被忽略。

```
# delegate_tool.py 关键代码
model=creds["model"],   # creds来自delegation config，不是工具参数
```

**结果：** 父代理以为派了 DeepSeek，实际派了 MiMo。父代理基于错误前提做了 Flash vs Pro 对比，全部无效。

## 两次运行的真正区别

两次子代理都是 MiMo V2.5 Pro。行为差异来自 **goal 约束**，不是模型差异：

| | 第一次 | 第二次 |
|--|--------|--------|
| 模型 | mimo-v2.5-pro | mimo-v2.5-pro |
| goal 约束 | 无 | "只完成指定任务，不要做任何额外的事情" |
| 结果 | 报告 + Wordle + HTTP服务器 + 浏览器 | 只有报告 |
| API调用 | 50次 | 3次 |
| 耗时 | 815秒 | 71秒 |

**结论：不是模型"闲不住"，是goal缺少终止约束。**

## 核心教训

### 1. Agent框架的终止机制问题

任务完成后 Agent 循环未被终止，模型自主寻找新任务。这不是特定模型的行为，任何有 Agent 能力的模型在同样条件下都可能出现。

### 2. 归因要查日志，不要信转述

父代理的口头报告（"DeepSeek在扫描"）≠ 实际执行模型。SQLite 会话日志才是事实来源。

### 3. delegate_task 的 model 参数≠实际使用的模型

Hermes 的 delegation config 会静默覆盖工具参数。要确认实际模型，查 sessions 表的 `model` 和 `billing_base_url` 字段。

### 4. 行为对比需要控制变量

对比两个模型的行为，首先要确认确实是两个不同模型。同一个模型跑两次、goal不同，不能得出"模型A比模型B稳重"的结论。

## 原始数据

- `session_log.json` — 子代理会话原始 JSON（108条消息）
- `screenshots/` — 游戏界面和聊天记录截图

## 原始来源

- 截图原作者：关彧（Guanyu-AI-prog）
- 发布渠道：朋友圈（非群聊）
- 事件发生：2026-08-01
- 勘误日期：2026-08-10
