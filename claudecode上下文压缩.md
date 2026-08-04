# Claude Code Best 上下文压缩技术报告

## 1. 范围与结论

本文研究对象是 [claude-code-best/claude-code](https://github.com/claude-code-best/claude-code) 仓库在提交 `2ccc216289833994ba3121afdd95b694126d495c` 的实现。

> 注意：该仓库的 `AGENTS.md/CLAUDE.md` 明确说明它是对 Claude Code 的逆向、反编译和工程化复原，并非 Anthropic 官方开源仓库。因此，本文描述的是该仓库可验证的源码行为，不能自动外推为官方闭源 Claude Code 的精确实现。

该项目的上下文管理不是单一“摘要”动作，而是多层流水线：

```text
单次/并行工具结果限额
          ↓
Snip（可选的选择性删除）
          ↓
Micro-compact（旧工具结果清理或 cache edit）
          ↓
Context Collapse（实验性、可选）
          ↓
主动 Auto Compact 判断
          ↓
Session Memory 轻量压缩优先
          ↓ 失败/不可用
完整对话摘要压缩
          ↓
API prompt-too-long 时 Reactive Compact 兜底
```

核心思想是：尽可能先用确定性、低成本方式减小工具输出；只有仍接近上下文上限时，才调用模型生成会话摘要。

---

## 2. 请求前主流水线

`src/query.ts` 在每次模型请求前按以下顺序处理模型可见消息：

```text
完整会话消息
    |
    v
getMessagesAfterCompactBoundary()
    |  只取最近压缩边界后的有效历史
    v
释放仅供 UI 使用的 toolUseResult 原始对象
    |
    v
applyToolResultBudget()
    |  限制同一 user message 内并行工具结果总量
    v
snipCompactIfNeeded()             [HISTORY_SNIP 开启时]
    |  移除被 snip boundary 点名的消息
    v
microcompactMessages()
    |  清理旧工具结果或向 API 注入 cache edits
    v
Context Collapse 投影             [实验功能开启时]
    |
    v
autoCompactIfNeeded()
    |  达到阈值则做 Session Memory 或完整摘要
    v
模型 API 请求
    |
    +---- prompt-too-long / media-too-large
                |
                v
          Reactive Compact 后重试
```

这条顺序很重要：工具输出压缩发生在全量摘要之前，因此可能直接把请求降到安全区，避免摘要模型调用。

---

## 3. 上下文窗口与自动触发

### 3.1 窗口解析

默认上下文窗口为 `200,000 tokens`。`getContextWindowForModel()` 会结合环境变量、模型能力、`[1m]` 后缀和 1M beta 等信息解析实际窗口。

压缩判断使用“有效窗口”：

```text
summaryOutputReserve
    = min(模型最大输出额度, 20,000)

effectiveContextWindow
    = contextWindow - summaryOutputReserve
```

预留最多 20K tokens，是为了确保压缩摘要本身仍有输出空间。

### 3.2 Auto Compact 阈值

```text
autoCompactThreshold
    = effectiveContextWindow - autoCompactBuffer
```

Buffer 随窗口变化：

```text
effective window < 400K    -> 13K
effective window 400K~799K -> 30K
effective window >= 800K   -> 50K
```

以默认 200K 窗口、20K 摘要输出预留为例：

```text
context window             200K
- summary output reserve    20K
= effective window         180K
- auto compact buffer       13K
= auto compact threshold   167K
```

源码还提供 `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` 供测试或覆盖，但结果不会超过正常阈值。

### 3.3 预测式触发

项目不仅检查当前 token，还估计下一轮增长：

```text
estimatedMaxTurnGrowth
    = min(maxModelOutput, 20K)
    + 15K 工具结果增长估计
```

如果当前请求加预计增长可能超过有效窗口，会在真正调用模型前再尝试一次自动压缩，降低“检查时没超限、工具执行后突然爆窗”的风险。

### 3.4 逃逸条件与熔断

以下情况不会主动 Auto Compact：

- 查询来源本身是 `compact` 或 `session_memory`，避免递归死锁；
- 设置了 `DISABLE_COMPACT` 或 `DISABLE_AUTO_COMPACT`；
- 用户关闭 `autoCompactEnabled`；
- Reactive-only 或 Context Collapse 正在接管上下文；
- 连续自动压缩失败达到 3 次。

三次失败熔断用于防止不可恢复的超限会话每轮重复发起失败摘要请求。

---

## 4. 第一层：工具结果写入前和请求前限额

### 4.1 单个工具结果落盘

工具声明 `maxResultSizeChars`。系统默认全局阈值为 `50,000 characters`，工具可声明更低阈值；超过阈值的纯文本结果会写入：

```text
<project session directory>/tool-results/<toolUseId>.txt|json
```

模型只收到文件路径和约 2,000 bytes 的头部预览：

```text
<persisted-output>
Output too large (...). Full output saved to: ...

Preview (first ...):
[头部预览]
...
</persisted-output>
```

这不是语义摘要，而是“完整结果外置 + 小预览”。非文本内容不能走这条持久化路径。

### 4.2 并行工具结果总预算

同一轮 user message 可能同时携带多个并行 `tool_result`。项目设置默认聚合上限：

```text
MAX_TOOL_RESULTS_PER_MESSAGE_CHARS = 200,000
```

超过时，优先把较大的结果外置到磁盘，直到该消息回到预算内。替换决定可记录进 transcript，恢复会话时重建同样的投影，以保持 prompt cache 稳定。

---

## 5. 第二层：Micro-Compact

Micro-compact 专门处理旧工具结果，不生成整段会话摘要。

可处理工具集合包括：

```text
FileRead
Shell（Bash/PowerShell 等）
Grep
Glob
WebSearch
WebFetch
FileEdit
FileWrite
```

### 5.1 时间型 Micro-Compact

当主线程空闲时间超过配置阈值时，服务器 prompt cache 很可能已经失效。此时系统保留最近若干工具结果，旧结果替换为：

```text
[Old tool result content cleared]
```

默认配置是关闭；默认时间阈值为 60 分钟、保留最近 5 个可压缩工具结果，实际可由远程配置控制。

```text
旧工具结果 1 ─┐
旧工具结果 2 ─┼─> [Old tool result content cleared]
旧工具结果 3 ─┘
最近结果 4   ───> 保留
最近结果 5   ───> 保留
```

### 5.2 Cached Micro-Compact

支持 cache editing 的 Claude 4.x 模型可使用 API 级 `cache_edits`：

```text
本地消息内容：保持不变

API 附加：
cache_edits:
  - delete_tool_result(tool_use_id=A)
  - delete_tool_result(tool_use_id=B)
```

默认计数策略：

```text
活动工具结果 > 10 个时触发
始终保留最近 5 个
删除更旧的 tool_result cache references
```

它的优势是不用改写本地消息前缀即可减少服务器缓存中的旧工具结果。该能力受 feature/environment 和模型支持条件限制。

---

## 6. 第三层：Session Memory 轻量压缩

Auto Compact 触发后，系统优先尝试 Session Memory，而不是立即再调用一次摘要模型。

前提：

```text
Session Memory 功能开启
Session Memory 文件存在且不是空模板
能够定位 lastSummarizedMessageId
生成后的上下文低于 Auto Compact 阈值
```

处理逻辑：

```text
已有 Session Memory
        |
        v
定位最后一次被 Memory 覆盖的消息
        |
        v
选择需要原样保留的近期尾部
        |
        +-- 至少约 10K tokens
        +-- 至少 5 条带文本消息
        +-- 最多约 40K tokens
        +-- 不拆 tool_use / tool_result
        +-- 不拆同一 assistant message.id 的 thinking/tool blocks
        |
        v
Compact Boundary
+ Session Memory 摘要消息
+ 近期原始消息
+ Plan / SessionStart hook 状态
```

如果 Session Memory 不可用、边界无法确定或结果仍超阈值，则返回 `null`，继续走完整摘要压缩。

---

## 7. 第四层：完整摘要压缩

`compactConversation()` 是完整压缩核心。

```text
1. 计算压缩前 token
2. 执行 PreCompact hooks
3. 合并用户自定义指令和 hook 指令
4. 创建详细摘要 prompt
5. 通过 forked agent 请求摘要
6. 摘要成功后清理旧 file-state cache
7. 重建压缩后上下文
8. 执行 SessionStart / PostCompact hooks
9. 写入 compact boundary 和统计信息
```

### 7.1 摘要内容合同

摘要 prompt 要求模型保留：

- 用户主要请求和意图；
- 技术概念和架构决策；
- 文件、代码段和修改原因；
- 错误、修复和用户反馈；
- 所有非工具结果用户消息；
- 未完成任务和当前工作；
- 安全相关约束；
- 与最新工作直接相关的下一步。

摘要以隐藏的 user message 重新注入，并标记 `isCompactSummary`。

### 7.2 Prompt Cache 复用

摘要通过 forked agent 运行，并尽量复用主会话的 system prompt、tools 和上下文缓存前缀，降低压缩请求的 cache creation 成本。

### 7.3 摘要请求本身超限

如果压缩请求也返回 prompt-too-long：

```text
按 API round 对历史分组
        |
从最老分组开始删除
        |
至少保留一组可摘要内容
        |
添加“早期历史已截断”标记
        |
最多重试 3 次
```

这是有损的最后兜底，但比让会话永久卡死更可用。

---

## 8. 压缩后上下文重建

完整压缩后的模型上下文不是只有一段摘要：

```text
Compact Boundary Marker
        |
        +-- trigger: auto/manual
        +-- preCompactTokenCount
        +-- 旧历史尾 UUID
        +-- 已发现的延迟工具
        |
Compact Summary
        |
可选保留消息（partial/session-memory/reactive 路径）
        |
最近读取文件附件
        +-- 最多 5 个文件
        +-- 单文件最多 5K tokens
        +-- 总预算 50K tokens
        |
Plan 文件与 Plan Mode 指令
        |
已调用 Skills
        +-- 单 Skill 最多 5K tokens
        +-- 总预算 25K tokens
        |
Deferred tools / Agent listing / MCP instructions 增量
        |
SessionStart Hook 与 PostCompact Hook 结果
```

这种设计避免“摘要记住了目标，却忘了当前文件、计划模式、Skill 和动态工具定义”。

压缩边界还保存 `preservedSegment` 元数据，用于恢复时重新连接被保留消息的链，防止磁盘去重或 parent UUID 关系让近期尾部丢失。

---

## 9. Reactive Compact 与 Partial Compact

### 9.1 Reactive Compact

如果 Provider 实际返回 prompt-too-long 或媒体大小错误，query loop 会暂时扣住错误消息，并最多执行一次紧急 `compactConversation()`：

```text
API 413 / prompt-too-long
        |
        v
Reactive Compact
        |
   成功？----否----> 返回原始错误
     |
     是
     |
     v
用压缩后消息重试模型请求
```

它是实际 Provider 超限后的反应式兜底，与请求前的主动 Auto Compact 不同。

### 9.2 Partial Compact

项目支持围绕选中消息做局部压缩：

```text
up_to：摘要选中点以前，保留较新的尾部
from： 摘要选中点以后，保留较早的前缀
```

`from` 可以保留前缀 prompt cache；`up_to` 把摘要放在保留消息之前，通常会破坏原前缀缓存。两种方向都维护工具调用配对和压缩边界。

---

## 10. 持久化与模型投影

项目不会把“界面仍可见”简单等同于“模型仍看见”。关键机制包括：

```text
Transcript / 完整消息链
        |
        +-- compact boundary 决定当前有效区间
        +-- content replacement 记录恢复大结果替换
        +-- preservedSegment 修复保留消息链
        +-- snip boundary 记录选择性移除 UUID
        |
        v
模型请求投影
```

普通压缩后，query loop 使用 `getMessagesAfterCompactBoundary()` 只发送最新有效压缩区间。UI 全屏回滚可以继续保留旧消息用于滚动查看，但模型不再携带这些原始内容。

---

## 11. 总结

该项目的上下文压缩逻辑可以浓缩为：

```text
             便宜、确定性的处理优先
                       |
        +--------------+--------------+
        |                             |
工具结果外置/预算              Micro-compact/cache edit
        |                             |
        +--------------+--------------+
                       |
                 仍接近上限？
                  /          \
                否            是
                |             |
             正常请求    Session Memory
                              |
                         不可用/不够小
                              |
                         完整摘要压缩
                              |
                         重建工作状态
                              |
                         正常模型请求
                              |
                         API 仍然超限？
                              |
                       Reactive Compact
```

工程上的主要优点是：

1. 工具结果有单项、单轮聚合和历史清理三层控制；
2. 主动阈值、预测式阈值和 API 实际超限三道防线互补；
3. Session Memory 可以避免部分重复摘要调用；
4. 完整摘要后会显式恢复文件、计划、Skill、工具和 Hook 状态；
5. 工具调用配对、thinking 分片和恢复链有专门的不变量保护；
6. 连续失败有三次熔断，压缩请求自身超限有最多三次裁头重试。

需要保留的判断边界是：该仓库包含大量 feature gate、远程配置和逆向复原代码，不同构建或用户类型启用的路径可能不同。阅读单个函数不足以断言生产运行行为，必须结合 `feature()`、环境变量、用户配置和模型能力一起判断。

---

## 12. 主要源码索引

- [请求循环 `src/query.ts`](https://github.com/claude-code-best/claude-code/blob/2ccc216289833994ba3121afdd95b694126d495c/src/query.ts)
- [自动压缩 `src/services/compact/autoCompact.ts`](https://github.com/claude-code-best/claude-code/blob/2ccc216289833994ba3121afdd95b694126d495c/src/services/compact/autoCompact.ts)
- [完整与局部压缩 `src/services/compact/compact.ts`](https://github.com/claude-code-best/claude-code/blob/2ccc216289833994ba3121afdd95b694126d495c/src/services/compact/compact.ts)
- [Micro-compact `src/services/compact/microCompact.ts`](https://github.com/claude-code-best/claude-code/blob/2ccc216289833994ba3121afdd95b694126d495c/src/services/compact/microCompact.ts)
- [Cached micro-compact](https://github.com/claude-code-best/claude-code/blob/2ccc216289833994ba3121afdd95b694126d495c/src/services/compact/cachedMicrocompact.ts)
- [Session Memory 压缩](https://github.com/claude-code-best/claude-code/blob/2ccc216289833994ba3121afdd95b694126d495c/src/services/compact/sessionMemoryCompact.ts)
- [Reactive compact](https://github.com/claude-code-best/claude-code/blob/2ccc216289833994ba3121afdd95b694126d495c/src/services/compact/reactiveCompact.ts)
- [上下文窗口解析](https://github.com/claude-code-best/claude-code/blob/2ccc216289833994ba3121afdd95b694126d495c/src/utils/context.ts)
- [工具结果外置和聚合预算](https://github.com/claude-code-best/claude-code/blob/2ccc216289833994ba3121afdd95b694126d495c/src/utils/toolResultStorage.ts)
- [仓库自带 Token 预算文档](https://github.com/claude-code-best/claude-code/blob/2ccc216289833994ba3121afdd95b694126d495c/docs/context/token-budget.mdx)
