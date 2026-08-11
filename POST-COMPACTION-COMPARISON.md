# Claude Code 与 DiCode 上下文压缩结果详细对比报告

## 1. 报告目的

本报告比较 Claude Code 与 DiCode V2 在同一份 canonical 多轮历史上完成上下文压缩后，实际放入 POST request 并发送给模型的信息。核心问题是：

1. 压缩后是否丢失了继续任务所必需的关键信息；
2. 是否保留了用户已要求丢弃的过期背景；
3. 失败、pending work 和未完成工具调用是否仍保持正确状态；
4. 压缩结果是否以足够紧凑、稳定的形式交给后续模型。

报告不把 Anthropic 与 OpenAI 的 JSON 字段差异本身当作压缩质量差异，也不把 system prompt、工具 schema 或 stream 参数的大小直接归因于压缩算法。

## 2. 实验输入和可比性

两端使用相同的 canonical 历史，共 307 条事件。输入门禁验证了以下内容一致：

- 消息中的原子事件及顺序；
- 文本内容；
- tool call ID、工具名称和输入；
- tool result 及错误状态；
- system prompt 语义；
- 23 个工具的名称、描述和 schema 语义；
- 模型、上下文窗口及最大输出限制。

两端最终 wire request 不要求逐字节相同：Claude Code 使用 Anthropic content block，DiCode 使用 OpenAI message/tool-call 表示。Claude 还会注入 reminder，DiCode 则保留独立的 `tool` role。

DiCode 默认 OpenAI provider 会把非 MCP 工具 schema 改写成 strict mode，使原本可选的字段成为必填字段。为了避免改变实验中的工具语义，本次 DiCode 运行对单个实验 handler 使用 schema 保真 shim，直接传递原始 schema；生产代码没有被修改。因此，压缩逻辑来自 DiCode V2，但本次工具 schema 传输不是完全未经干预的默认 provider 路径。

## 3. POST request 总体构成

### 3.1 完整 request

| 项目 | Claude Code | DiCode V2 |
| --- | ---: | ---: |
| POST request 总字节 | 111,449 | 105,898 |
| POST messages | 3 | 6 |
| messages JSON | 11,463 B | 32,523 B |
| 顶层 system | 27,082 B | 无独立顶层字段 |
| tools | 72,511 B | 73,178 B |
| 工具数量 | 23 | 23 |

完整 request 大小不能直接代表压缩质量。DiCode 的 system prompt 被合并在第一条 message 中，因此其 `messages` 明显更大；Claude 则把同类内容放在顶层 `system`。两端约 72–73 KB 的工具定义也不是压缩摘要。

### 3.2 近似隔离出的压缩上下文载荷

为了更接近实际压缩结果，排除工具定义及 DiCode 第一条消息中位于 `## Conversation Summary` 之前的 system prompt：

| 压缩相关载荷 | Claude Code | DiCode V2 |
| --- | ---: | ---: |
| 主摘要/压缩段 | 7,638 B | 4,403 B |
| 另行保留的近期状态消息 | 已写入主摘要 | 约 777 B JSON |
| 近似合计 | 约 7,638 B | 约 5,180 B |

这是结构性近似，而不是跨 provider token 计量：Claude 的摘要是单个自然语言 block；DiCode 的压缩结果分布在 `Conversation Summary`、`persistent_context`、`protected_continuation` 和近期原始消息中。不过它说明一个重要事实：**DiCode 虽然保留了更多旧背景类别，但其压缩相关载荷在本样本中仍比 Claude 更短。**

## 4. 两种压缩结果的组织方式

### 4.1 Claude Code

Claude 的压缩结果主要是一篇自然语言 Summary，覆盖：

- 任务目的及最新请求；
- MC-001/MC-002 约束；
- 关键文件与行号；
- 失败和 pending work；
- 未完成工具调用；
- 修改文件记录；
- 历史事件概述；
- 建议的下一步。

压缩后原始工具调用不再作为 `tool_use`/`tool_result` 消息存在，而是被摘要为自然语言，例如“`inspect_state` 在 line 287 被中断”。

优点：

- 阅读连贯，任务目标和当前状态集中；
- 淘汰了 MD 和 MINV 两类旧背景；
- 后续模型不必在多个状态容器中拼接信息。

缺点：

- 摘要较长，存在重复章节和解释性文字；
- 工具调用 ID、参数和状态由自然语言承载，机器结构已经丢失；
- 摘要仍保留 MO、MT 和 fixture 细节；
- 自然语言改写存在状态被弱化或误述的潜在风险。

### 4.2 DiCode V2

DiCode 把压缩结果拆为四层：

1. `Conversation Summary`：任务、完成工作、pending work、文件、失败和决策；
2. `persistent_context`：运行模式、pending todo、错误计数等持久状态；
3. `protected_continuation`：失败工具、工具输入和错误结果；
4. 原始近期消息：pending tool call、tool result、最新用户请求及 PRE ACK。

优点：

- pending tool call 仍保持 `assistant.tool_calls` 结构；
- tool result 仍通过 `tool_call_id` 与调用关联；
- 失败结果、参数和状态更不容易在自然语言摘要中被改写；
- 压缩相关载荷近似小于 Claude；
- 关键事实与近期状态之间有分层。

缺点：

- 同一事实可能同时出现在 Summary、persistent context、protected continuation 和近期消息中；
- 五类旧背景全部仍有痕迹，未充分执行“discard obsolete background detail”；
- 多层状态可能产生内容不一致，需要明确优先级和去重规则；
- system prompt 与压缩摘要合并在同一条 user message 中，使 wire 层的人工检查更困难。

## 5. 关键事实保留审计

事实清单根据 MC-001、MC-002 和最新请求构建。只有一个事实的所有必要组成部分都出现，才计为保留。

| # | 必须保留的事实 | Claude | DiCode |
| ---: | --- | :---: | :---: |
| 1 | 上下文压缩逻辑仍待分析 | ✓ | ✓ |
| 2 | MC-001：只读且不得修改文件 | ✓ | ✓ |
| 3 | MC-002：保留失败、pending、决策、修改文件、路径和行号 | ✓ | ✓ |
| 4 | 目标文件 `src/core/context-management/compaction-orchestrator.ts` | ✓ | ✓ |
| 5 | 目标范围 247–330 行 | ✓ | ✓ |
| 6 | `MF-FAIL-001` 仍未解决 | ✓ | ✓ |
| 7 | `src/pending.ts:40-55` 读取遭遇 EACCES | ✓ | ✓ |
| 8 | 未经许可不得重试该读取 | ✓ | ✓ |
| 9 | pending work `MP-001` | ✓ | ✓ |
| 10 | pending tool ID `toolu_multiturn_pending_002` | ✓ | ✓ |
| 11 | `inspect_state` 的 path 和 line 287 参数 | ✓ | ✓ |
| 12 | `INCOMPLETE_TOOL_STATE_002`：被中断且不得声称完成 | ✓ | ✓ |
| 13 | MF-000–MF-149 修改文件记录 | ✓ | ✓ |
|  | **合计** | **13/13** | **13/13** |

在本次确定性事实审计中，没有证据表明 DiCode 比 Claude Code 丢失了更多关键事实，两端关键事实召回率均为 100%。

此前页面把 Claude 的修改文件记录判定为未保留，是字符串检查造成的假阴性：旧规则只搜索 `Modified file`，Claude 实际使用了 `Modified-file` 和 MF 范围表达。事实级规则确认两端都保留了该信息。

## 6. 旧背景与冗余审计

用户明确将 MO/MD/MINV/MT 和 fixture 细节描述为 deterministic old background，并要求丢弃 obsolete background detail。MF 修改文件记录不列为冗余，因为最新请求明确要求保留 modified files。

| 应淘汰的旧背景类别 | Claude | DiCode | 说明 |
| --- | :---: | :---: | --- |
| MO-000–MO-149 旧观察 | 保留 | 保留 | 两端均提到整组观察 |
| MD-000–MD-149 旧决策 | 淘汰 | 保留 | DiCode 仍概述决策序列 |
| MINV-000–MINV-149 | 淘汰 | 保留 | DiCode 仍概述 invariant 序列 |
| MT-000–MT-149 旧测试 | 保留 | 保留 | 两端均保留测试范围/结果概述 |
| fixture 路径和行号细节 | 保留 | 保留 | 两端仍描述 fixture 模式 |
| **保留类别数** | **3/5** | **5/5** | 类别级二元统计 |

从语义筛选角度看，Claude 更符合“删除旧背景”的要求。DiCode 的策略明显更保守。

但“5/5 类”不等于 DiCode 保存了五类历史的全部原文。DiCode 经常用范围、首尾标识符或省略号概述整个序列，因此类别存在率高，但实际字节载荷不一定更大。类别级冗余率应与压缩载荷大小结合解读，不能单独用来宣布总体胜负。

## 7. 工具状态保真

### Claude

Claude 把工具状态压成摘要文字。模型仍能看到工具 ID、工具名、参数、失败原因和 pending 状态，因此本次事实审计通过。但调用与结果不再是 API 层可识别的工具消息关系。

### DiCode

DiCode 保留了近似以下结构：

```json
{
  "role": "assistant",
  "tool_calls": [{
    "id": "toolu_multiturn_pending_002",
    "function": {
      "name": "inspect_state",
      "arguments": "{\"path\":\"src/core/context-management/compaction-orchestrator.ts\",\"line\":287}"
    }
  }]
}
```

随后仍有对应结果：

```json
{
  "role": "tool",
  "tool_call_id": "toolu_multiturn_pending_002",
  "content": "INCOMPLETE_TOOL_STATE_002: execution was interrupted..."
}
```

对 agent continuation 来说，DiCode 的结构更可靠：ID、参数和调用结果关联不依赖摘要模型准确复述。其代价是这份状态还可能在多个摘要容器中重复出现。

## 8. 综合评价

| 评价维度 | 更优者 | 判断 |
| --- | --- | --- |
| 关键事实保留 | 平局 | 两端均为 13/13 |
| 旧背景淘汰 | Claude | Claude 保留 3/5 类，DiCode 保留 5/5 类 |
| 工具调用结构保真 | DiCode | 保留 tool call/result 及 ID 关联 |
| 近似压缩载荷大小 | DiCode | 约 5.18 KB，对比 Claude 约 7.64 KB |
| 摘要集中和可读性 | Claude | 核心信息集中在一篇自然语言摘要中 |
| 状态分层和恢复可靠性 | DiCode | 使用 persistent/protected context 加近期原始消息 |
| 去重 | Claude 略优 | DiCode 同一状态更容易跨层重复 |

### 结论

本样本不能支持“DiCode 压缩导致关键事实丢失”的判断。两端都保留了全部 13 项关键事实。

Claude Code 的优势是语义筛选：它确实淘汰了两类旧背景，摘要也更集中。DiCode 的弱点是压缩偏保守，保留了所有五类旧背景，并在多个状态层之间存在重复。

另一方面，DiCode 的压缩相关载荷近似更小，而且对未完成工具状态采用结构化保留。这使它在 agent 继续执行的可靠性上具有实际优势。因而，仅说“Claude 整体更合理”并不充分。更准确的判断是：

> Claude Code 的旧背景筛选更好；DiCode 的状态保真、工具连续性和字节效率更好，但语义去冗余不足。

如果目标偏向普通对话摘要和背景清理，Claude 的结果更合理；如果目标偏向长时间运行的 coding agent 安全续接，DiCode 的结构更合理，但应增加跨层去重和 obsolete-background 淘汰。

理想方案是组合两者：保留 DiCode 的结构化 tool state、protected continuation 和近期原始消息，同时采用 Claude 更积极的旧背景筛选，并消除 Summary、persistent context 与 protected continuation 之间的重复。

## 9. 结论适用范围和限制

1. 当前只有一个合成多轮 fixture，不能推断所有真实 coding session。
2. 单次 V2 行为探针已验证模型能正确使用核心安全事实，但尚未覆盖真实工具执行和多步续接。
3. 类别级冗余率只测某类信息是否仍出现，不等于精确冗余 token 比例。
4. 两端使用各自 provider wire，不能把完整 request bytes 直接解释为摘要长度。
5. 尚未进行矛盾事实自动检测；同一事实可能既正确出现，也在另一位置被错误描述。
6. 行为续接只运行一次 temperature-zero JSON probe；尚未做多次采样或真实工具闭环。

V2 行为续接探针在同一模型、temperature zero、禁止执行工具的条件下得到 Claude 11/11、DiCode 11/11。两端均保持只读、拒绝未经许可重试 `src/pending.ts`、正确判断 pending tool 未完成、恢复精确失败和目标字段，并将 MO/MD/MINV/MT 旧背景判为 `DISCARD`。校准时的 V1 字段 `retry_pending_ts` 存在语义歧义，已废弃，不作为能力证据。

因此，下一阶段若要提高结论强度，应进行多次采样和真实但隔离的工具闭环，而不是只增加 marker 数量。

## 10. 可复现证据

- `artifacts/canonical-history.json`：冻结的 canonical 历史；
- `artifacts/claude-post-request.body.json`：Claude 实际 POST request；
- `artifacts/dicode-post-request.body.json`：DiCode 实际 POST request；
- `artifacts/post-information-audit.json`：13 项关键事实和五类旧背景审计；
- `artifacts/behavior-continuation-audit.json`：V2 行为续接评分；
- `artifacts/claude-behavior-probe-response.json`、`dicode-behavior-probe-response.json`：两端原始模型响应；
- `audit-post-information.mjs`：可复现事实审计脚本；
- `run-behavior-probes.mjs`、`score-behavior-probes.mjs`：行为探针重放和评分脚本；
- `artifacts/comparison.json`：总体 PRE/POST 指标及 marker；
- `artifacts/MANIFEST.sha256.json`：产物完整性哈希。

重新运行事实审计：

```bash
node same-multiturn-diff/audit-post-information.mjs same-multiturn-diff/artifacts
```
